---
layout: post
title:  "diesel <3> - mini tectonic in rust - file layer"
date:   2022-10-16 23:00:00 +0930
categories: rust diesel facebook tectonic metadata file
---

이번에는 `File` 레이어의 스키마를 구현해보도록 하자. tectonic 논문의 테이블을 다시 살펴보면,

| Layer    | Key                   | Value                  | Sharded By  | Mapping                               |
|----------|-----------------------|------------------------|-------------|---------------------------------------|
| Name     | (dir_id, subdir)      | subdir_info, subdir_id | dir_id      | dir -> list of subdirs (expanded)     |
|          | (dir_id, filename)    | file_info, file_id     | dir_id      | dir -> list of files (expanded)       |
| **File** | **(file_id, blk_id)** | **blk_info**           | **file_id** | **file -> list of blocks (expanded)** |
| Block    | blk_id                | list<disk_id>          | blk_id      | block -> list of disks (i.e., chunks) |
|          | (disk_id, blk_id)     | chunk_info             | blk_id      | disk -> list of blocks (expanded)     |

이와 같고 여기서 `File` 레이어에서의 Key는 (file_id, blk_id) 로 되어 있고,
Value는 (blk_info)로 되어 있다. `blk_info`에 구체적으로 어떤 내용이 들어가야 하는지는
논문의 내용을 바탕으로 추후 유추해 보도록 하고, 일단 이 레이어에서 관리하는 Mapping도 "*expanded*"로
표시되어 있음을 유의해서 볼 필요가 있다. 이전 포스트의 *expanded* 에 대한 설명을 바탕으로,
File 레이어에 들어갈 메타데이터는 대략 아래와 같을 것이라 볼 수 있다:
```text
# (Layer, Key, Value)
(File, (/A/B, Blk_1), Blk_1_info)
(File, (/A/B, Blk_2), Blk_2_info)
(File, (/A/B, Blk_3), Blk_3_info)
(File, (/A/C, Blk_4), Blk_4_info)
(File, (/A/C, Blk_5), Blk_5_info)
(File, (/A/D/E, Blk_6), Blk_6_info)
```

편의상 `file_id` 대신 file full path를 사용했지만, 이것은 이전 포스트에서의 `Dentry` 테이블을
통해 다시 표현할 수 있을 것이다.
간단히 말하면, 주어진 `File`을 구성하는 각 블럭마다 하나의 row 로 표현을 하겠다는 것이다.
위 예제에서는, `/A/B`라는 파일은 3개의 블럭으로 구성되어 있고, 각 블럭들의 id는 각각
`Blk_1`, `Blk_2`, `Blk_3`라고 볼 수 있다. 
유사하게, `/A/C`라는 파일은 2개의 블럭으로 구성되어 있고, 각 블럭들의 id는 각각
`Blk_4`, `Blk_5`로 되어 있다.
여기 적혀 있는 블럭 id들은 다음 포스팅에서의 block layer 메타데이터가 가진 매핑 정보를
활용하여 실제 디스크에서 읽어올 수 있게 될 것이다. 

그러면, `blk_info`에는 어떤 정보들이 포함되어야 할까? 이 구조체에 대해 논문에 명확히 설명된 바
없으므로 (혹시 제가 놓친 것이면 알려주시길), 일단 추측을 해보자면, 

1. Block 이 fixed- 혹은 variable-sized 인지 나타내는 flag
2. Block 이 차지하는 영역을, start offset (in bytes) 과 length (in bytes) 로 표현
3. Block 의 인코딩 정보 (예를 들면, k-m erasure coded, 혹은 replicated, ...)
4. Block 의 checksum

1, 2번은 tectonic 논문에서 언급된 것 처럼 태생적으로 Facebook이 보유하고 있는,
Haystack, F4, HDFS 스토리지 클러스터 등을 통합하여 운영을 효율화하고자 하는 목적이
컸기 때문에, 이 정도는 넣지 않았을까 싶다. 논문에서도 다음과 같은 언급이 있다: 
> Tectonic uses tenant-specific optimizations to match the performance
> of specialized storage systems.

상상을 해보자면, 일종의 mapping size를 응용 프로그램에 맞게, 
혹은 File에 맞게 동적으로 조절하던지 아니면 말던지 하여 다양한 요구 사항을 유연하게 
맞추려 하지 않았을까.

3번은 tectonic 논문에 언급된 다음의 문구로부터 추측하였다:
> Tectonic provides per-block durability to allow tenants to
tune the tradeoff between storage capacity, fault tolerance, and
performance

다시 상상을 해보자면, File이 한개의 blk id로 이루어져 있고, 이 내용을 읽어 client에게
리턴을 해주는 상황이라 가정하자. (다음 포스팅에 있겠지만) 분산 시스템에서 다수의 노드/디스크에
퍼져있는 데이터를 읽어오는 경우, 한 디스크에서만 데이터를 읽으면 될지 아니면 여러 디스크에서 모두
데이터를 읽어와야 할지? 이는 전적으로 데이터를 어떤 방식으로 썼느냐에 달려 있을 것이다.
만약 Replication을 선택하여 동일한 4 KB 블럭 내용을 3개의 디스크에 복제하여 쓴 경우라면,
3개의 디스크 중 어느 하나만 읽어서 리턴해주어도 될 것이고 (혹시 성능 최적화가 들어간다면, 3개를
모두 async read하여 제일 빨리 오는 응답을 리턴할 수도 있을 것이고), Erasure-coding을
선택하여 4 KB 블럭 1개를 블럭 2개로 표현하였다면, 2개의 디스크로부터 모두 읽어와서 decoding을
해주어야 할 것이다. 아마도, tectonic은 두가지 모드를 모두 지원하고 있을 것 같고 이를 위해
각 블럭의 `blk_info`에는 위 3번 항목의 내용을 담고 있어야 하지 않을까 생각한다.

4번은 integrity check 혹은 deduplication fingerprint 등의 용도로 뭔가 해당 블럭을
대표할 수 있는 "고유의 값"에 대한 것들도 `blk_info`에 들어가면 좋지 않을까 하여 적어보았다.
참고로, `blk_info`와 더불어 Value 컬럼에 있는 정보들은 block layer 논의 이후 한꺼번에
schema migration으로 추가해볼 예정이다.

위 내용을 바탕으로, 다음과 같은 schema migration을 계획해보면 되겠다:
1. `name` 이라는 테이블을 생성한다. 그 테이블에는, `file_id`, `blk_id`, `value`라는 컬럼을 만든다.
2. `name` 의 primary key로는 auto-increment integer를 준다. 여러 process들이
동시에 new file을 생성하려고 하는 경우에도, DB의 auto-increment에 의존하여 unique ID를 
쉽게 줄 수 있게 될 것이다. 
3. `name` 의 `file_id`에 index 를 생성하도록 한다. 예상해보면, tectonic은 주어진
`file_id`에 대해 모든 `blk_id`를 얻어오고 싶은 경우가 많을 것 같다. 혹시 추후에 특정
`blk_id`만을 얻어온다던지, blk_id가 갖는 속성에 order by를 준다던지 하는 쿼리가 나타난다면,
그 때는 (file_id, blk_id.SomeProperty)로 composite key를 만들어야 할 수도 있겠다.
4. `name` 의 `file_id`는 `Dentry.dir_id` 의 foreign key 제약을 걸도록 한다. 이는,
새로운 파일을 생성할 때 `Dentry` 에 먼저 insert를 하고나서 `file` 에 insert를 해야
constraint violation이 발생하지 않음을 의미한다.

diesel cli를 사용해서 schema migration 템플릿을 생성해보도록 하자. 이 migration의
이름은 `file-layer`이다:

```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration generate file-layer
Creating migrations/2022-10-16-143645_file-layer/up.sql
Creating migrations/2022-10-16-143645_file-layer/down.sql
```

이제 생성된 두 파일에, 위에 논의한 내용 바탕으로 
아래와 같이 roll-forward/back 스크립트를 작성해보자: 

```bash
$ cat migrations/2022-10-16-143645_file-layer/up.sql
CREATE TABLE file (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_id INTEGER NOT NULL,
  blk_id INTEGER NOT NULL,
  value VARCHAR,
  FOREIGN KEY(file_id) REFERENCES dentry(id)
);
CREATE INDEX file_blk_id ON file (file_id);

$ cat migrations/2022-10-16-143645_file-layer/down.sql
drop index file_blk_id;
drop table file;
```

이제 schema migration을 (또 역시 2번) 수행해보자:
```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run                
Running migration 2022-10-16-143645_file-layer
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run
```

roll-back 스크립트가 정상 작동하는지 확인차 redo를 (또 역시 2번) 수행해보자:
```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-10-16-143645_file-layer
Running migration 2022-10-16-143645_file-layer
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-10-16-143645_file-layer
Running migration 2022-10-16-143645_file-layer
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
CREATE INDEX parent_child ON name (parent, child);
CREATE TABLE file (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  file_id INTEGER NOT NULL,
  blk_id INTEGER NOT NULL,
  value VARCHAR,
  FOREIGN KEY(file_id) REFERENCES dentry(id)
);
CREATE INDEX file_blk_id ON file (file_id);
```

세번째 DB migration을 마쳤다!