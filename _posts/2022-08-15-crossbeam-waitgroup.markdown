---
layout: post
title:  "crossbeam <3> - WaitGroup"
date:   2022-08-15 08:15:00 +0930
categories: actix rust crossbeam coordination
---

Multi-thread application을 작성하다 보면, 다음과 같은 상황을 마주할 때가 있을 것이다:
*모든 쓰레드들이 동일 선상에 도착했을 경우, 특정 연산을 수행하도록 하고 싶다!* 예를 들면, storage perf benchmark를
작성한다고 해보자. 사용자에게 주어진 concurrency 설정만큼 thread를 생성하고 있는 도중, spawned thread가
곧바로 target에 workload를 주게 되면, 그래프 상으로 실험 초반의 특이한 
fluctuation으로 나타나면서 실험 결과의 해석을 잘못된 방향으로 이끌게 될 가능성이 있다. 이 경우는, 
실험 시작전의 setup phase에서 모든 thread들이 준비 완료가 됨을 확인한 이후에, workload를 생성시키는 것이
일관적이고 재현가능한 실험 결과 확보에 도움을 줄 것이다.

이럴 때 사용할 수 있는 라이브러리가, `crossbeam` crate에 포함된 `WaitGroup`이다 
([crates.io](https://crates.io/crates/crossbeam)). 
2022/08/15를 기준으로 1400만+ 다운로드를 기록한 인기 라이브러리이다.
영화 '한산'의 1400만 관객을 바라면서 학익진의 예로 `WaitGroup`의 사용법을 살펴보면, 아래와 같다.

우선, 학익진에서 공격력을 극대화하려면 모든 판옥선들이 자신의 위치에 도달한 뒤
집중 포화를 시작해야만 한다. 극 중에서, 원균의 판옥선이 뒤쳐져 진영이 완전히
갖추어 지지 않는 장면이 나오는데, 이순신 장군은 초인적인 인내를 보여주며 학익진이
완성될 때를 기다려 전투를 승리로 이끈다. 
여기서, **모든 배들이 자신의 위치에 도착하는 것**을 어떻게 구현할 것인지가 
고민될 수 있다. Java 프로그래머라면 자연스럽게 `Barrier`를 떠올릴 것이다
(참고: [Java Concurrency in Practice](https://www.amazon.com/dp/0321349601/?tag=javamysqlanta-20), 
[Baeldung](https://www.baeldung.com/java-cyclic-barrier)).
비슷한 기능을 가진, 하지만 거기에 RUST 스러움을 약간 입힌 것이 `WaitGroup`이라
보면 되겠다. 가령, `Barrier`와는 달리 `WaitGroup`은 생성 시점에 몇 개의 
thread를 기다릴 것인지를 미리 정해둘 필요가 없다. 필요한 thread로 변수를
move 시키기 전에, `clone()`을 하는 시점에 자동으로 `WaitGroup`에 등록이 된다
(이것이 idiomatic RUST 방식이 아닐까 생각한다). 
또 다른 차이점은, `wait()`이 한번 호출된 `WaitGroup`은 재사용할 수가 없고,
만약 그런 시도를 하게 되면 use-after-move 류의 컴파일러 에러가 발생하게 된다.
이 차이점들은 [crossbeam 문서](https://docs.rs/crossbeam/0.8.2/crossbeam/sync/struct.WaitGroup.html)에
잘 소개되어 있다.

```rust
use std::thread;
use crossbeam::sync::WaitGroup;

fn main() {
    let hak_ik_jin = WaitGroup::new();
    let no_more_cannon = WaitGroup::new();
    for ship_num in 0..10 {
        let hak_ik_jin = hak_ik_jin.clone();
        let no_more_cannon = no_more_cannon.clone();
        thread::spawn(move || {
            println!("Ship {} is holding its fire...", ship_num);
            hak_ik_jin.wait();
            println!("Ship {} is attacking!", ship_num);
            no_more_cannon.wait();
        });
    }
    hak_ik_jin.wait();
    println!("Open Fire!");
    no_more_cannon.wait();
    println!("Victory!");
}
```
이를 실행하면, 아래와 같은 결과가 나온다:
```rust
Ship 0 is holding its fire...
Ship 1 is holding its fire...
Ship 2 is holding its fire...
Ship 3 is holding its fire...
Ship 4 is holding its fire...
Ship 5 is holding its fire...
Ship 6 is holding its fire...
Ship 7 is holding its fire...
Ship 8 is holding its fire...
Ship 9 is holding its fire...
Ship 9 is attacking!
Ship 0 is attacking!
Ship 1 is attacking!
Ship 2 is attacking!
Ship 3 is attacking!
Ship 4 is attacking!
Open Fire!
Ship 5 is attacking!
Ship 6 is attacking!
Ship 7 is attacking!
Ship 8 is attacking!
Victory!
```

`Open Fire`의 실행 시점은 async runtime에 특성에 기인한 것으로, 
`WaitGroup`의 정확성과는 크게 관계가 없으므로 무시할 수 있다.
여기서 중요한 것은, 

* 모든 배들이 준비가 된 이후에야, "attacking"이 시작될 수 있다는 점
* 모든 배들이 "attacking"을 한 이후에야, "Victory"가 될 수 있다는 점

이다. 만약 `attacking`이 `holding its fire` 이전에 한번이라도 찍혔다던지,
`Victory`가 `attacking`보다 먼저 찍혔다던지 하면, 예제 프로그램의 버그를
의심해 보아야 할 것.