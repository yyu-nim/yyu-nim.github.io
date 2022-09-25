---
layout: post
title:  "lazy_static <2> - early loading"
date:   2022-09-25 08:10:00 +0930
categories: rust lazy_static singleton
---

`lazy_static` 매크로의 예제들을 보면, "static"이 crate 이름에 포함되어 있는 이유는 금방
수긍이 갈 것이다. 그런데, 왜 "lazy"가 붙는지 궁금할 수 있을 것이다. [문서](https://docs.rs/lazy_static/latest/lazy_static/) 를 따라 들어가보면, 
아래와 같은 문구가 있다.

> On first deref, EXPR gets evaluated and stored internally, 
> such that all further derefs can return a reference to the same object. 
> Note that this can lead to deadlocks if you have multiple lazy statics 
> that depend on each other in their initialization.


즉, 일종의 Effecitve C++의 [Meyers Singleton](https://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf)
구현에서 처럼 `Singleton::instance()`를 최초로 실행한 시점에 동적으로 싱글턴 객체를 생성해 주고, 
두번째, 세번째, ... 호출에서는 기존에 생성한 싱글턴 객체를 리턴해주는 방식을 RUST에 가져온 것이라
볼 수 있을 것이다. 물론 내부적으로 thread safety 보장을 위한 구현이 들어가 있을 것이고.

문서대로 "laziness"가 구현이 되어 있는지, 확인을 해보면 좋을 것이다. 
`lazy_static` crate 에 보면 `initialize()`가 있다고 하는데, lazy static object를 
명시적으로 곧바로 생성시켜 주는 함수로 보인다 (물론 그냥 아무 멤버 변수 접근해도 비슷한 효과이겠지만). 
이를 활용해, `initialize()`가 호출되지 않은 버전과 호출된 버전 간의 first object access latency를
측정해보는 실험을 다음과 같이 디자인할 수 있을 것이다.


```rust
use std::thread;
use std::time::{Duration, Instant};
use lazy_static::lazy_static;

struct LazyObject(u32);
struct EarlyObject(u32);

lazy_static!{
    static ref LazyObjectSingleton: LazyObject = {
        thread::sleep(Duration::from_secs(5));
        LazyObject(0)
    };

    static ref EarlyObjectSingleton: EarlyObject = {
        thread::sleep(Duration::from_secs(7));
        EarlyObject(0)
    };
}

fn main() {
    println!("======== Access Latency for Uninitialized Lazy Static Object ========");
    test_lazy_loaded_object_access_latency();

    println!("======== Access Latency for Initialized   Lazy Static Object ========");
    test_early_loaded_object_access_latency();
}

fn test_lazy_loaded_object_access_latency() {
    let instant = Instant::now();
    println!("{:10} us, Started", instant.elapsed().as_micros());

    let _v = LazyObjectSingleton.0;
    println!("{:10} us, First  read from LazyObjectSingleton.0", instant.elapsed().as_micros());

    let _v = LazyObjectSingleton.0;
    println!("{:10} us, Second read from LazyObjectSingleton.0", instant.elapsed().as_micros());
}

fn test_early_loaded_object_access_latency() {
    // 여기서 lazy_static 객체를 미리 초기화 해줄 수 있다
    lazy_static::initialize(&EarlyObjectSingleton);

    let instant = Instant::now();
    println!("{:10} us, Started", instant.elapsed().as_micros());

    let _v = EarlyObjectSingleton.0;
    println!("{:10} us, First  read from EarlyObjectSingleton.0", instant.elapsed().as_micros());

    let _v = EarlyObjectSingleton.0;
    println!("{:10} us, Second read from EarlyObjectSingleton.0", instant.elapsed().as_micros());
}
```

이를 실행하면 다음과 같다:
```bash
$ cargo run
======== Access Latency for Uninitialized Lazy Static Object ========
         0 us, Started
   5005067 us, First  read from LazyObjectSingleton.0
   5005127 us, Second read from LazyObjectSingleton.0
======== Access Latency for Initialized   Lazy Static Object ========
         0 us, Started
        67 us, First  read from EarlyObjectSingleton.0
        74 us, Second read from EarlyObjectSingleton.0
```

위 두 실험의 주요 차이는 `lazy_static::initialize()`를 미리 실행해주었냐 아니냐의 차이일 것이다.
해당 fn을 실행해 주지 않았던 `LazyObjectSingleton`의 경우에는 객체를 맨 처음 접근하는 시점에
(즉 `.0`을 읽으려는 시점), `thread::sleep(Duration::from_secs(5))`가 호출이 되면서
5초 정도가 소요되고, 2번째 접근 부터는 이미 생성되어 `LazyObjectSingleton`에 캐쉬된 객체를
리턴해주기만 하면 되기 때문에 latency가 거의 없다. 반면, `EarlyObjectSingleton`의 경우에는
객체를 맨 처음 접근하는 시점에, 이미 객체 생성이 완료되어 캐쉬된 값에 접근할 수 있어
latency가 거의 없다. 여기서 얻을 수 있는 결론은 자명하다: **lazy_static 객체의 생성  
latency를 서비스 도중에 감당할 수 있으면 그냥 쓰면 되고, tail-latency 측면에서 
부담이 되는 수준이면 서비스 시작 전에 initialize()를 써서 early loading 해둘 것**.

메서드 추출하고, 히스토그램으로 반복 실행하고 등등 코드 길이도 줄이고 과학적인 테스트 하는 방법도 
있겠으나, 이번에는 println을 주석의 의미로도 써보고 싶었고, 포스트 목적과 직접 연관없는 부분은
설명을 생략하고 싶었다.