---
layout: post
title:  "diesel <4> - mini tectonic in rust - block layer"
date:   2022-10-23 23:00:00 +0930
categories: rust diesel facebook tectonic metadata block
---

이번에는 `Block` 레이어의 스키마를 구현해보도록 하자. tectonic 논문의 테이블을 다시 살펴보면 아래와 같다.

| Layer     | Key                   | Value                  | Sharded By | Mapping                                   |
|-----------|-----------------------|------------------------|------------|-------------------------------------------|
| Name      | (dir_id, subdir)      | subdir_info, subdir_id | dir_id     | dir -> list of subdirs (expanded)         |
|           | (dir_id, filename)    | file_info, file_id     | dir_id     | dir -> list of files (expanded)           |
| File      | (file_id, blk_id)     | blk_info               | file_id    | file -> list of blocks (expanded)         |
| **Block** | **blk_id**            | **list of disk_id**    | **blk_id** | **block -> list of disks (i.e., chunks)** |
|           | **(disk_id, blk_id)** | **chunk_info**         | **blk_id** | **disk -> list of blocks (expanded)**     |

`blk_id`가 주어졌을 때 `disk_id`의 리스트를 찾을 수 있다는 것이 기본 아이디어 이다.
한 가지 특이점은, Mapping 컬럼의 설명을 읽어보면 여기만 "expanded"가 없다는 점이다.
즉, 특정 블럭이 여러개의 디스크에 저장되어 있을 때, 이들을 각각 디스크마다 하나의 새로운 row로
저장하는게 아니라, 디스크 리스트를 컬럼 하나에 들어갈 수 있도록 인코딩해서 저장하겠다는 것이다.
예를 들면, 아래와 같이 저장이 될 것이다:
```text
# (Layer, Key, Value)
(Block, Blk1, "Disk1,Disk2,Disk3,Disk4")
(Block, Blk2, "Disk5,Disk6,Disk7")
(Block, Blk3, "Disk8,Disk20")
...
```

이것이 의미하는 바가 무엇일까?
일단은, 이 테이블에 들어있는 Value 들은 immutable 할 것이라는 점이다. 
Value가 포함된 Row가 지워지는 경우는 있을 수 있어도, Value 데이터를 일부분만 변경할
가능성은 많이 않아 보인다. 일단 정보가, 문자열로 인코딩되어 있기 때문에 Value에 쿼리를
하는 것이 비싼 연산이 될 수 있다 (e.g., string match). 만약, 부분적인 업데이트를 원하거나
쿼리를 하고 싶었다면, 다른 레이어들에서 처럼 "expanded" 방식을 택했어야 한다는 합리적인
의심을 해볼 수 있을 것이다.
이러한 immutability에 대한 제약을 논문에서는 `sealed` 라 표현하는
것이 저자의 이해이고, 이 덕분에 metadata caching의 효과를 누릴 수 있게 된다고 보이고, 이것을
논문에서도 다음과 같이 언급하고 있다:

> The contents of sealed filesystem objects cannot change; their metadata can be cached at metadata nodes and at clients 
> without compromising consistency

만약 사용자가 Blk2에 매핑된 File의 특정 영역을 변경하고자 한다면?
Tectonic은, Blk2를 업데이트 하는 대신 새로운 Blk을 할당하여 이전 포스트의 `file` 테이블을
업데이트하려 할 것이다. 
Blk2에 해당되는 디스크 중 일부가 고장났을 경우는? 데이터 복구나 migration에 대한 전략을
논문에서 찾지 못했지만 (어쩌면 놓쳤을지도), application의 redundancy SLA를 만족시킬 수 
있도록 디스크 리스트를 확보해서 (이전 리스트와 overlap이 있을수도 없을수도 있겠다),
새로운 Blk을 할당하여 역시 마찬가지로 `file` 테이블을 업데이트할 것이다.
만약, 사용자가 어떤 이유로든 오래된 블럭 정보 (여기에서는 Blk2)를 그대로 가지고 접근하려 
했다면? Tectonic이 가진 cache coherence 프로토콜에 따라 stale access로 판단하여
cache refresh를 발생시킬 것이라 생각한다. 논문에 이와 관련된 것처럼 보이는 문구가 있다:

> A stale Block layer cache can be detected during reads, 
> triggering a cache refresh.

이제 마지막으로 (disk_id, blk_id)가 주어지면, 한개의 디스크상에서 어떤 연속된 영역을 읽어야 
할지를 알 수 있게 된다. 그 단위를 `chunk`라 불리는 것으로 보인다. "expanded" 타입인 것으로
미루어 보아, 다음과 같은 식으로 테이블에 적힐 것이라 유추해 볼 수 있다:

```text
# (Layer, Key, Value)
(Block, (Disk1, Blk1), 0 ~ 8)
(Block, (Disk1, Blk2), 120 ~ 128)
(Block, (Disk1, Blk3), 800 ~ 808)
(Block, (Disk2, Blk4), 40 ~ 48)
(Block, (Disk2, Blk5), 48 ~ 56)
...
```

diesel cli를 사용해서 schema migration 템플릿을 생성해보도록 하자. 이 migration의
이름은 `block-layer`이다:
```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration generate block-layer
Creating migrations/2022-10-23-135826_block-layer/up.sql
Creating migrations/2022-10-23-135826_block-layer/down.sql
```

`Value`에 대해서는 이전 포스팅에서도 그랬지만, 아직 본격적으로 structure를 정의하지는 않았다.
이번 포스팅에서도 비슷한데, 한가지 차이점은 `block` 테이블의 Value에는 디스크의 리스트를 
인코딩해서 담아야 하는데, 일단 JSON 타입으로 저장하기로 하자. sqlite3 에서는 JSON 타입의
컬럼이 존재하지 않으므로, 일단 VARCHAR 로 두고, JSON 객체를 string으로 변환해서
저장하도록 하자.

```bash
$ cat migrations/2022-10-23-135826_block-layer/up.sql
CREATE TABLE block (
    blk_id INTEGER PRIMARY KEY AUTOINCREMENT,
    value VARCHAR NOT NULL
);

CREATE TABLE disk (
    disk_id INTEGER PRIMARY KEY AUTOINCREMENT,
    blk_id INTEGER NOT NULL,
    value VARCHAR NOT NULL,
    FOREIGN KEY(blk_id) REFERENCES block(blk_id)
);
CREATE INDEX disk_blk_id ON disk (disk_id, blk_id);

$ cat migrations/2022-10-23-135826_block-layer/down.sql
DROP INDEX disk_blk_id;
DROP TABLE disk;
DROP TABLE block;
```

이제 DB migration run/redo를 2번씩 진행해서 idempotency/rollback 을 테스트
해보도록 하자:
```bash
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run   
Running migration 2022-10-23-135826_block-layer
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration run
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-10-23-135826_block-layer
Running migration 2022-10-23-135826_block-layer
$ DATABASE_URL="sqlite://./test-db100.sqlite" diesel migration redo
Rolling back migration 2022-10-23-135826_block-layer
Running migration 2022-10-23-135826_block-layer
```

이제 `sqlite3` cli로 스키마를 확인해보면 아래와 같을 것이다:
```bash
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
CREATE TABLE block (
    blk_id INTEGER PRIMARY KEY AUTOINCREMENT,
    value VARCHAR NOT NULL
);
CREATE TABLE disk (
    disk_id INTEGER PRIMARY KEY AUTOINCREMENT,
    blk_id INTEGER NOT NULL,
    value VARCHAR NOT NULL,
    FOREIGN KEY(blk_id) REFERENCES block(blk_id)
);
CREATE INDEX disk_blk_id ON disk (disk_id, blk_id);
```

아직 Value 컬럼의 구조에 대해서는 추가 논의가 필요한 부분이 있으나,
구체적인 변경사항은 개발을 진행하면서 알아볼 수 있을 것이다.
이제 Tectonic의 read/write API를 구현해볼 준비가 되었다!