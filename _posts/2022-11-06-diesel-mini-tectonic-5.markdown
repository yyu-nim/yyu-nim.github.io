---
layout: post
title:  "diesel <5> - mini tectonic in rust - DB shards"
date:   2022-11-06 23:00:00 +0930
categories: rust diesel facebook tectonic db sharding
---

이전 포스팅까지 [Tectonic](https://www.usenix.org/system/files/fast21-pan.pdf) 논문을 바탕으로 DB migration을 통해 schema를 생성해보았다.
아직 타입이 정해지지 않은 컬럼들도 있었고 (e.g., `Value`), 논리적인 오류도 몇 존재하는데
(e.g., `name.parent` foreign key constraint), 이들은 포스팅을 진행하면서 
migration을 통해 schema fix도 적용해 볼 수 있을 것이다.

아직 한가지 중요한, 다루지 못한 주제가 하나 있는데 바로 Sharding (샤딩) 이다. 논문의 Table 1에
`Sharded By` 컬럼에 언급된 것에 의하면, `dir_id`, `file_id`, `blk_id`로의 샤딩이 이루어지고
있는 것으로 보인다. 쉽게 말하면, Tectonic이 SQL 쿼리에 포함된 위 컬럼의 값에 따라 적절한 DB 인스턴스를
선택하여 쿼리를 보낸다고 이해할 수 있을 것이다. 즉, DB 인스턴스가 애초에 1개가 아니고 다수개가 존재하는
것이다. 페이스북 규모의 거대한 데이터셋을 보유한 회사라면, DB 1개로는 스토리지 공간상, 
그리고 성능상의 제약으로 Tectonic 서비스를 제공할 수 없었을 것이고, 따라서 논리적인 거대한 테이블을
쪼개어 물리적으로 분산하여 DB에 저장하는 방식은 자연스러운 선택으로 볼 수 있다.

이를 바탕으로, Tectonic의 생성자에 대해 고민을 해보자. 어떤 식의 의존성 주입을 하면 좋을까?
일단은, DB에 접속하여 쿼리를 보낼 수 있어야 하니 DB connection을 가지면 좋을 것이다.
그런데, 샤딩을 지원하기 위해서 DB endpoints/connections이 다수개가 될 수 있음을 인지해야 한다.
또한, `dir_id`, `file_id`, `blk_id` 각각이 어느 정도의 address space를 가지는지, 
이에 대한 쿼리 트래픽이 어느 정도가 될지에 따라 서로 다른 "샤드 숫자"를 가지게 될 가능성이 높음을
유의해야 할 것이다. 즉, id의 타입에 따라 별도로 DB connections 을 유지하는 전략도 필요할
것이다. "사용성"의 관점에서, Tectonic 서비스를 호출하려는 클라이언트들은 샤딩에 대해서는 알 필요
없이 Tectonic API 를 호출하기만 하면 되도록 디자인 되면 좋을 것이다.

초기 디자인으로, `DirDb`, `FileDb`, `BlkDb` 라는 세 개의 추상화된 DB accessor를 
만들어, 각각 고유의 샤드 개수를 유지하도록 생성자를 만든 뒤,
Tectonic에 주입시키는 것을 생각해 볼 수 있다.
각각의 Db 들은 1) 고유의 shard key에 따라 적절한 DB connection을 선택해주고,
2) 고유의 entity (e.g., dir, file, block)에 대한 쿼리를 제공해 줄 수 있을 것이다.
그렇다면 예상되는 단점은? 아마도 세 개의 DB 간 join 연산이 필요할 경우, DB 의 자체적인
기능을 이용하지 못하고, Tectonic이 데이터를 모아서 직접 join을 수행해주어야 할 필요가
생길 수도 있다. 다만, 이것은 꼭 Db 타입을 나눠서 뿐만이 아니라, 샤딩으로 인해서도 생길 수 있는
side effect 이므로, 감수할 만하다고 생각이 든다.

이에 따라, 다음과 같은 초기 구성이 가능할 것이다

<script src="https://gist.github.com/yyu-nim/794cd233dcfd5a6572d9cef6245429ea.js"></script>

가정은,
- DirDb는 2개의 DB 인스턴스를 운영
- FileDb는 3개의 DB 인스턴스를 운영
- BlkDb는 4개의 DB 인스턴스를 운영
한다는 점이다. 

Test-case로 `perform_migration`을 추가해 두었는데, `mod config`에 있는
모든 DB endpoints에 대해 DB migration을 수행해주는 unit test 이다. 사실, 
검증용이라기 보다는, DB migration script 용으로 작성해 두었고, 더 나은 방법은
[embedded_migration](https://docs.rs/diesel_migrations/latest/diesel_migrations/) 을
사용하는 것일텐데, 포스팅 목적의 범위를 넘어서는 관계로 다루지 않는다 (+ 사용법 연구 필요).