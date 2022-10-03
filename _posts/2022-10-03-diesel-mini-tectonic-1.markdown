---
layout: post
title:  "diesel <1> - mini tectonic in rust"
date:   2022-10-03 23:00:00 +0930
categories: rust diesel facebook tectonic
---

Facebook에서 2021년도에 FAST'21에 발표한
[Tectonic](https://www.usenix.org/system/files/fast21-pan.pdf) 이라는
흥미로운 논문이 있다. 쉽게 말하면, 스토리지 서버들로 Exabyte 스케일의 분산 파일
시스템을 구축했다는 이야기 이다. 여기서 생길 수 있는 의문은, *"File A를 읽고 싶을 때,
어떤 서버의 어떤 디스크의 어떤 블럭 주소를 읽어야 할까?"* 일 것이다. 단일 
노드에서라면, 파일시스템의 inode에 인코딩된 블럭 주소들을 통해 (윈도우라면 MFT), 
그리고 index traversal을 통해, 원하는 데이터를 읽을 수 있을 것이다. 
분산 파일 시스템에서도 마찬가지의 책임을 기대할 수 있을 것이고, 위 **Tectonic** 논문에서는
고맙게도 Facebook이 사용한 메타데이터 디자인을 공개해 두었다. RUST로 tectonic이
사용하는 **Metadata Store** 비슷한 것을 구현하고 싶다면 어떻게 해야 할지? 
이것이 이번 포스팅의 주제이다. 이번에는 특별히, 시리즈 연재를 해보려 한다.

일단은, metadata의 on-disk layout에 대해서는 먼 미래 언젠가, 
본인이 만든 서비스가 너무 커져서 수 % 수준의 space optimization으로도
매우 큰 경비 절감을 이룰 수 있을 때 다루는 것이 바람직해 보인다. 이 대신, 
DBMS를 사용하여 자료 구조를 디스크에 효율적으로 저장 및 검색하는 문제를 모두
위임하도록 하고, 서비스가 (즉 tectonic) 원하는 쿼리에 응답을 잘 줄 수 있도록
스키마를 디자인하고 인덱스 설계 및 샤딩에 집중하는 편이 실용적인 선택일 것이다.
물론 이번에는 직접 디자인을 하는 대신, 
tectonic이 사용한 메타데이터 디자인을 그대로 사용해볼 것이긴 하지만.

RUST 진영의 [diesel](https://crates.io/crates/diesel) 은 
ORM 및 schema migration 을 지원하는 강력한 DB crates 로 알려져 있으므로, 
이걸 사용해보기로 하자. 일단, [Getting Started](https://diesel.rs/guides/getting-started) 를
참조하여 `diesel_cli`를 설치하도록 한다. 몇 가지 DB backend 중에 한개 이상을
선택하여 설치를 진행할 수 있는데, 예를 들면 아래와 같다:
```bash
$ cargo install diesel_cli --no-default-features --features sqlite
```

최신 언어/라이브러리 답게, cloud-native가 되기 위한 [12-factor](https://12factor.net/backing-services) 원칙을
이해하고 있는 것으로 보인다. `DATABASE_URL`에 DB endpoint 정보를 담아 두면,
`diesel_cli` 실행시, 어떤 DB에 변경을 가하거나 정보를 읽을지가 자동으로 반영된다.
예를 들어, `./test-db100.sqlite` 라는 DB 인스턴스에 우리 애플리케이션이 사용하는
DB 인스턴스를 만들고 싶다면 (이것이 schema migration이 제공하는 기능의 일종이다),
아래와 같은 커맨드를 실행할 수 있다:

```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel setup
Creating database: sqlite://./test-db100.sqlite
```
사실, 위처럼 `diesel setup`으로 DB 인스턴스를 생성하는 과정은 optional이다.
추후 실행될 `migration run` 커맨드가 동일한 효과를 줄 수 있기 때문이다. 그래도,
diesel이 제공하는 다양한 기능들을 파악하기 위한 좋은 예제가 될 것으로 기대한다.

그 다음은, schema migration 코드를 작성하는 것이다. migration을 위해서는
양방향의 코드를 모두 작성해야 하는데, 1) `up.sql`이라는 SQL 코드에 DB에 원하는 변경점을
기술하고 (roll-forward), 2) `down.sql` 이라는 SQL 코드에 `up.sql`에서 가해졌던 변경 사항을
원래대로 되돌리는 코드를 작성하도록 되어 있다 (roll-back). 
그 두 파일을 손으로 생성할 필요 없이, 적당히 uniqueness가 보장되도록 timestamp를
붙여서 diesel이 생성해 줄 수 있다:
```bash
$ diesel migration generate create_mini_tectonic
Creating migrations/2022-09-26-141831_create_mini_tectonic/up.sql
Creating migrations/2022-09-26-141831_create_mini_tectonic/down.sql
```

`up.sql`에는 최초의 schema migration 인 만큼, `CREATE TABLE` DML이 들어가게
될 것이다. 이 시점에, tectonic 논문을 펴고 *Table 1*을 펼쳐보도록 한다:

| Layer | Key                | Value                  | Sharded By | Mapping                               |
|-------|--------------------|------------------------|------------|---------------------------------------|
| Name  | (dir_id, subdir)   | subdir_info, subdir_id | dir_id     | dir -> list of subdirs (expanded)     |
|       | (dir_id, filename) | file_info, file_id     | dir_id     | dir -> list of files (expanded)       |
| File  | (file_id, blk_id)  | blk_info               | file_id    | file -> list of blocks (expanded)     |
| Block | blk_id             | list<disk_id>          | blk_id     | block -> list of disks (i.e., chunks) |
|       | (disk_id, blk_id)  | chunk_info             | blk_id     | disk -> list of blocks (expanded)     |


일단은, 페이스북은 RocksDB의 분산 클러스터 버전인 `ZippyDB`를 이용하여 
위 메타데이터 스토어를 구현하였다 하니, SQL로 표현하기 최적의 모습은 아닐 수 있겠다.
Row의 type을 runtime에 추정하여 고정된 column의 데이터를 적당히 해석하던지,
아니면 sparse table을 만들어서 특정 column의 값을 바탕으로 몇몇 column들만
해석에 사용하는 방식을 써야할 수 있겠다. 어쨌든, 생각이 너무 많아지면 일을 시작할 수가
없으니, 아주 단순하게 모든 column을 string 타입을 만들어 테이블을 생성해보도록 하자.
`sharded_by`는 테이블에 포함될 필요가 없어 보이기도 하지만, 일단 두도록 한다.
Tectonic 메타데이터를 SQL로 표현하는 좋은 방법을 추후에 찾아, schema migration으로
원하는 대로 바꿀 수 있게 될 것이다.

`up.sql`, `down.sql` 파일들을 각각 열어, 아래와 같은 내용으로 넣어보자:
```bash
$ cat up.sql
CREATE TABLE tectonic (
   id INTEGER PRIMARY KEY AUTOINCREMENT,
   layer VARCHAR NOT NULL,
   key VARCHAR NOT NULL,
   value VARCHAR NOT NULL,
   sharded_by VARCHAR,
   mappings VARCHAR
)

$ cat down.sql
DROP TABLE tectonic;
```
위와 같이 schema migration 코드 작성을 마쳤다면, 이제 실제로 migration을 
수행해 보자. `migration run`은 `DATABASE_URL`이 가리키는 DB 인스턴스에
변경을 가한다. 

```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run
Running migration 2022-09-26-141831_create_mini_tectonic
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run
```

굳이 두 번을 실행한 이유는, `migration run`이 idempotent 하다는 것을 
보이기 위해서이다. 즉, 가장 최신 code를 체크아웃 한 다음에, schema migration을
할 것인지 말 것인지가 아리송하면 또 실행해 보아도 무방하다. 두 번 이상을 실행하더라도,
마치 한 번 실행한 결과처럼 보이게 되는 *idempotency(멱등성)*을 지원하고 있다.
`migration run`의 경우 roll-forward만을 테스트할 수 있고, roll-back은
테스트할 수 없기 때문에, `migration redo`를 이용해서 roll-back까지도 
테스트해볼 수 있다. 다만, 이 때 rollback으로 인한 데이터 손실이 발생할 수 있다.
테스트 환경에서라면 부담없이 실행할 수 있겠지만, 프로덕션 환경에서라면 극도의 주의를
필요로 할 것이다. 이전과 마찬가지로, migration 코드가 제대로 작성되었다면, 
`migratino redo` 도 idempotent 한 특성을 보여주어야 한다.

```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-09-26-141831_create_mini_tectonic
Running migration 2022-09-26-141831_create_mini_tectonic
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-09-26-141831_create_mini_tectonic
Running migration 2022-09-26-141831_create_mini_tectonic
```

schema migration이 제대로 수행되었다면, 생성된 DB 테이블에 대한 표현이
RUST의 매크로로 `src/schema.rs`에 생성된다. 이 RUST 파일은, 
`pub mod schema`로 injection 되어 사용될 수 있는데, 만약 `main`이 위치한
코드의 위치 때문에 `schema.rs`를 포함하는 것이 어려워진 경우 
`#[path="<path>"]` 를 통해 `schema.rs`의 위치를 지정해 줄 수 있다.
이 때 생성된 `schema.rs`는 대략 아래와 같은 모양이 된다:
```text
diesel::table! {
    tectonic (id) {
        id -> Nullable<Integer>,
        layer -> Text,
        key -> Text,
        value -> Text,
        sharded_by -> Nullable<Text>,
        mappings -> Nullable<Text>,
    }
}
```

구체적인 column type들은 아직 논의하지 않았으므로 임시적으로 `Text`로만
두고 있음에 유의할 것. 실제 DB 인스턴스에 접속하여 테이블도 제대로 생성되었는지,
확인해 볼 수 있다. sqlite의 경우에는 아래의 커맨드를 사용할 수 있다:
```bash
$ sqlite3 test-db100.sqlite
sqlite> .schema
CREATE TABLE __diesel_schema_migrations (
       version VARCHAR(50) PRIMARY KEY NOT NULL,
       run_on TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE sqlite_sequence(name,seq);
CREATE TABLE tectonic (
   id INTEGER PRIMARY KEY AUTOINCREMENT,
   layer VARCHAR NOT NULL,
   key VARCHAR NOT NULL,
   value VARCHAR NOT NULL,
   sharded_by VARCHAR,
   mappings VARCHAR
);
```
`__diesel_schema_migrations`은 diesel이 내부적으로 관리하는 메타데이터를 
담고 있고, 이 덕분에 idempotency의 지원이라던지, migration 순서의 보장 등의
기능이 구현될 수 있었다고 볼 수 있겠다.

다음의 예제는, 생성된 `schema.rs`를 어떻게 RUST 코드에
활용하여 테이블에 insert하거나 load할 수 있는지에 대해 설명하고 있다.

```rust
use std::borrow::Borrow;
use std::env;
use diesel::{Connection, ConnectionResult, SqliteConnection};
use diesel::prelude::*;
use models::*;
use schema::tectonic::dsl::*;

#[path="../schema.rs"]
pub mod schema;

fn main() {
    let mut connection = establish_connection();
    let new_tectonic = Tectonic::new("layer1", "key1", "val1");
    diesel::insert_into(schema::tectonic::table)
        .values(&new_tectonic)
        .execute(&mut connection);

    let result = tectonic.load::<Tectonic>(&mut connection).expect("Error loading tectonic rows");
    println!("displaying {} tectonic entries...", result.len());
    for t in result {
        println!("{:?}", t);
    }
}

pub fn establish_connection() -> SqliteConnection {
    let db_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set, e.g., sqlite://./test-db.sqlite");
    SqliteConnection::establish(&db_url)
        .unwrap_or_else(|_| panic!("Error connecting to {}", db_url))
}

pub mod models {
    use diesel::prelude::*;
    use super::schema;

    #[derive(Queryable, Insertable, Debug)]
    #[diesel(table_name=schema::tectonic)]
    pub struct Tectonic {
        pub id: Option<i32>,
        pub layer: String,
        pub key: String,
        pub value: String,
        pub sharded_by: Option<String>,
        pub mappings: Option<String>,
    }
    impl Tectonic {
        pub fn new(layer: &str, key: &str, value: &str) -> Self {
            Tectonic {
                id: None,
                layer: layer.to_string(),
                key: key.to_string(),
                value: value.to_string(),
                sharded_by: None,
                mappings: None
            }
        }
    }
}
```

이를 실행하면 아래와 같은 결과가 나온다:
```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" cargo run
displaying 1 tectonic entries...
Tectonic { id: Some(1), layer: "layer1", key: "key1", value: "val1", sharded_by: None, mappings: None }

$ DATABASE_URL="sqlite://./test-db100.sqlite" cargo run
displaying 2 tectonic entries...
Tectonic { id: Some(1), layer: "layer1", key: "key1", value: "val1", sharded_by: None, mappings: None }
Tectonic { id: Some(2), layer: "layer1", key: "key1", value: "val1", sharded_by: None, mappings: None }

$ DATABASE_URL="sqlite://./test-db100.sqlite" cargo run
displaying 3 tectonic entries...
Tectonic { id: Some(1), layer: "layer1", key: "key1", value: "val1", sharded_by: None, mappings: None }
Tectonic { id: Some(2), layer: "layer1", key: "key1", value: "val1", sharded_by: None, mappings: None }
Tectonic { id: Some(3), layer: "layer1", key: "key1", value: "val1", sharded_by: None, mappings: None }
```

이제 기본적인 diesel의 read/write을 마쳤다!