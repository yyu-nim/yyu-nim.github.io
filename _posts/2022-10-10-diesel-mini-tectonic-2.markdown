---
layout: post
title:  "diesel <2> - mini tectonic in rust - name layer"
date:   2022-10-10 08:00:00 +0930
categories: rust diesel facebook tectonic metadata name
---

지난 포스트의 메타데이터 스키마를 다시 한번 살펴 보고, schema migration을 해보도록 하자.

| Layer | Key                | Value                  | Sharded By | Mapping                               |
|-------|--------------------|------------------------|------------|---------------------------------------|
| Name  | (dir_id, subdir)   | subdir_info, subdir_id | dir_id     | dir -> list of subdirs (expanded)     |
|       | (dir_id, filename) | file_info, file_id     | dir_id     | dir -> list of files (expanded)       |
| File  | (file_id, blk_id)  | blk_info               | file_id    | file -> list of blocks (expanded)     |
| Block | blk_id             | list<disk_id>          | blk_id     | block -> list of disks (i.e., chunks) |
|       | (disk_id, blk_id)  | chunk_info             | blk_id     | disk -> list of blocks (expanded)     |

일단은 `Layer`라는 컬럼이 눈에 띄는데, 이에 따라 뒤 따르는 `Key`, `Value` 등이
포함하는 정보들이 매우 달라지는 것을 볼 수 있다. 우리는 RDBMS + diesel를 쓸 예정이니,
table normalization, index 등을 고려할 때 `Layer` 별로 table을 하나 두는 것이
괜찮아 보인다. 만약 `Layer`가 가질 수 있는 값이 동적으로 변할 수 있다던지 (예를 들면 매 
릴리즈마다 여러 값들이 추가), 혹은 그 값이 지나치게 많아져 너무 많은 테이블을 관리하게
될 수 있다고 하면, 거대한 하나의 테이블을 디자인하여 Layer Id를 모든 row에 넣는 방식으로
일종의 multi-tenant 스타일을 고려해 볼 수도 있을 것이다. 비슷한 맥락에서, RDBMS가
아닌 schema에 대한 제약이 덜한 NoSQL을 사용한다던지, RocksDB와 같은 key-value 
라이브러리를 사용하는 경우에는, key에 ID 정보를 최대한 인코딩하여 단일 테이블이나 
DB 인스턴스에 메타데이터를 전부 담을 수도 있을 것이다. 어쨌든, 본 mini tectonic 과제에서는
Layer 별로 테이블을 분리하여 각 테이블 별로 컬럼을 디자인 하려 한다.
Diesel의 schema migration이라는 강력한 툴을 사용해서, 테이블 수가 많아지거나
컬럼이 복잡해져도, 복잡한 DB 스크립트 없이도 쉬운 개발이 가능해 질 수 있을지 
확인해보는 것도 의미가 있을 것이다.

`Name` 이라는 레이어를 우선 살펴보도록 하자.
[Tectonic](https://www.usenix.org/system/files/fast21-pan.pdf) 의
"3.3 Metadata Store" 를 살펴보면 아래와 같은 예제가 있다:

>  the Name and File layer and disk
to block list maps are expanded. A key mapped to a list is
expanded by storing each item in the list as a key, prefixed
by the true key. For example, if directory d1 contains files
foo and bar, we store two keys (d1, foo) and (d1, bar) in d1’s
Name shard.

논문에 쓰인 *"expanded"*의 뜻이 예제로 설명되어 있다. 즉, parent directory가
가진 subdirs이나 files가 있으면, 그것들을 하나의 value로 표현하는 대신, 각각 새로운
row를 생성하라는 의미로 해석된다.

```text
/A/B1/
/A/B1/F1
/A/B2/C1/
/A/F1
```
위와 같은 directory hierarchy 가 존재하는 경우,
```text
# (Layer, Key, Value)
(Name, (A, B1), 디렉토리 B1의 정보)
(Name, (A, B2), 디렉토리 B2의 정보) 
(Name, (A, F1), 파일 F1의 정보)
(Name, (B1, F1), 파일 F1의 정보)
(Name, (B2, C1), 디렉토리 C1의 정보)
```
위와 같이 5개의 row로 표현이 되어야 할 것 같다. `/A/F2` 라는 새로운 파일이 생성되는 경우,
기존 row에 locking을 한다던지 할 필요없이, 단지 새로운 row로 (Name, (A, F2), 파일 F2의 정보)를
추가해주면 되므로, "expanded"라는 용어를 사용한 것이 어느 정도 이해가 간다.

여기서 **dir_id**는 어떤 type을 주는 것이 좋을까? 위 예제에서는 그냥 편하게 dir_name (e.g., A, B1, ...)을
썼지만, 이 경우 서로 다른 디렉토리에 위치한 동일한 이름을 갖는 두 디렉토리간에 구분이
어려울 것이다. 가령, `/A/B/`와 `/C/B/`가 존재하는 경우, `B`라는 dir_name 만으로는
유일한 하나의 디렉토리를 지정할 수 없는 문제가 있다. 
얼핏 떠오르는 아이디어는, 루트 디렉토리로 부터의 절대 경로를 dir_id로 삼는 것이다. 
이 경우 uniqueness를 확보할 수 있다. 또한 이를 hashing 하여 shard key로 삼는 것도
문제가 없을 것이다. 다만 이 경우 문제는, 디렉토리 이름을 반복적으로 DB에 저장해야 하는 문제가
있고 디렉토리 이름 길이에 따라 용량적으로나 index 구성 측면에서 부담이 될 수 있다. 가령,
아래의 경우를 보자.

```text
/SupercalifragilisticexpialidociousDirectory/A1
/SupercalifragilisticexpialidociousDirectory/A2
...
/SupercalifragilisticexpialidociousDirectory/A10000
```

`SupercalifragilisticexpialidociousDirectory`라는 디렉토리가 어떤 이유로
매우 많은 파일을 가진 디렉토리라 한다면, 이를 굳이 매번 저장할 필요 없이, 여기에 dir_id를 부여해서
`Name` 테이블에는 dir_id를 저장하도록 하고, 그에 해당하는 dir_path를 별도의 테이블에 
저장하는 편이 합리적으로 보인다.
이러한 normalization을 추후에 schema migration으로 해결할 것이냐, 
초반부터 정하고 가느냐에 따라 개발 부채등이 달라질 것이고, 여기에 개발자의 판단이 필요한 영역이다. 

여기에서는 dir_id -> dir_path 에 대한 매핑을 별도의 테이블인 `Dentry`에 유지하고,
`Name` 테이블에는 `dir_id`를 `Dentry`로의 foreign key로 사용하도록 한다. 
이를 위해 다음과 같은 schema migration을 디자인해 볼 수 있을 것이다:

1. `Dentry` 테이블을 생성한다. `dir_id`는 primary key로 하고, `dir_path`에는
string을 저장하도록 한다. mini tectonic의 입장에서, 주어진 `dir_path`에 해당하는
`dir_id`를 reverse lookup 하고 싶을 가능성이 높으므로, `dir_path`에도 index를 생성한다.
2. `Name` 테이블을 생성하고, 네 개의 컬럼을 생성한다: `Parent`, `Child`, `ChildType`, and `Value`.
(Parent, Child)로 composite key를 정하고, ChildType은 `Dir` 혹은 `File` 이라는 
값만을 가질 수 있도록 제약을 둔다. `Name` 테이블의 primary key는 auto increment id로
하도록 해서, row id를 직접 사용하지는 말고 그냥 고유의 키를 주는 것으로만 쓰도록 한다.

이를 diesel schema migration을 위해 template을 생성해보자:

```bash
$ diesel migration generate name-layer
Creating migrations/2022-10-10-034509_name-layer/up.sql
Creating migrations/2022-10-10-034509_name-layer/down.sql
```

아래와 같이 migration 코드를 작성할 수 있다:
```bash
$ cat migrations/2022-10-10-034509_name-layer/up.sql
CREATE TABLE dentry (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    dir_path VARCHAR NOT NULL
);
CREATE INDEX pathindex ON dentry (dir_path);

CREATE TABLE name (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    parent VARCHAR NOT NULL,
    child VARCHAR NOT NULL,
    child_type VARCHAR CHECK ( child_type IN ('Dir', 'File') ) NOT NULL,
    value VARCHAR,
    FOREIGN KEY(parent) REFERENCES dentry(id)
);

$ cat migrations/2022-10-10-034509_name-layer/down.sql
drop table name;
drop index pathindex;
drop table dentry;
```

이전 포스트에서 사용했던 **DATABASE_URL**을 그대로 사용해서, migration을 (두번) 수행해보자:
```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run   
Running migration 2022-10-10-034509_name-layer
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run
```

마찬가지로 idempotency 테스트를 위해, 그리고 rollback 테스트를 위해 redo를 (두번) 수행해보자:
```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-10-10-034509_name-layer
Running migration 2022-10-10-034509_name-layer
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-10-10-034509_name-layer
Running migration 2022-10-10-034509_name-layer
```

이제 `sqlite3` cli로 스키마를 확인해보면 아래와 같을 것이다:
```bash
% sqlite3 ./test-db100.sqlite
SQLite version 3.37.0 2021-12-09 01:34:53
Enter ".help" for usage hints.
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
CREATE TABLE dentry (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    dir_path VARCHAR NOT NULL
);
CREATE INDEX pathindex ON dentry (dir_path);
CREATE TABLE name (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    parent VARCHAR NOT NULL,
    child VARCHAR NOT NULL,
    child_type VARCHAR CHECK ( child_type IN ('Dir', 'File') ) NOT NULL,
    value VARCHAR,
    FOREIGN KEY(parent) REFERENCES dentry(id)
);
```

두번째 DB migration을 마쳤다! 아직은 코드를 짤 때가 아니고, File / Block 레이어까지
스키마가 만들어지면, 그것들을 사용해보면서 tectonic API를 만들어보도록 하자.