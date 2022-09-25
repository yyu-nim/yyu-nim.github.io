---
layout: post
title:  "lazy_static <3> - lazy eval. with async/await"
date:   2022-09-25 22:10:00 +0930
categories: rust lazy_static async_once
---

`lazy_static!{}` 내부에 싱글턴 생성 로직을 넣다보면 궁금할 때가 있을 것이다; 
async/await을 쓰고 싶으면 어떻게 하지? tokio 기반의 application에 싱글턴을 도입하려 한다면
꼭 한번 묻게되는 질문일 것이다. 단순하게, 새로운 async runtime을 생성해서 (e.g., tokio::runtime::Runtime::new()), 
`block_on(async {})` 로 감싸는 것은 그럴듯 해 보이나, 싱글턴의 생성 시점에 이미 존재하고 있었던
async runtime과 충돌이 발생하는 "nested runtime" 문제가 발생할 수 있다. 물론 이전 포스트에서 
언급한 `lazy_static::initialize()`를 적당한 시점으로 당겨 호출하는 우회책도 있을 수 있지만...

`AsyncOnce`가 여기서 해결사로 등장한다 [(async_once)](https://crates.io/crates/async_once). 
async {}를 인자로 주어 객체를 생성할 수 있기 때문에, async 블럭 안에서 마음껏 .await을 호출할 수 있다.
이에 대한 간단한 실험을 아래와 같이 수행해 볼 수 있다.
`LazyObjectSingleton` 생성 시점에 tokio가 제공하는
async sleep()을 5초로 호출하여, 정말 5초 뒤에 객체 생성이 완료되는지 표준 출력의 timestamp를 통해
확인해보자. 또한, 이에 대한 대조군으로 앞서 언급한 nested runtime 문제도 직접 재현해 보기로 하자.

```rust
use std::time::Duration;
use lazy_static::lazy_static;
use async_once::AsyncOnce;
use tokio::time::Instant;

struct LazyObject(u32);

lazy_static! {
    static ref LazyObjectSingleton : AsyncOnce<LazyObject> = AsyncOnce::new(async {
        // 아래와 같이 .await이 필요한 경우, async block 내에서 호출되어야 함
        tokio::time::sleep(Duration::from_secs(5)).await;
        LazyObject(10)
    });

    static ref LazyObjectSingletonWithError : LazyObject = {
        let rt = tokio::runtime::Runtime::new().unwrap();
        // 실행 시점에 에러가 발생할 수 있는 코드이다
        rt.block_on(async {
            tokio::time::sleep(Duration::from_secs(5)).await; 
        });
        LazyObject(10)
    };
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {

    let instant = Instant::now();
    println!("{:10} us, Started", instant.elapsed().as_micros());

    let _v = LazyObjectSingleton.get().await.0;
    println!("{:10} us, First  read from LazyObjectSingleton.0", instant.elapsed().as_micros());

    let _v = LazyObjectSingleton.get().await.0;
    println!("{:10} us, Second read from LazyObjectSingleton.0", instant.elapsed().as_micros());

    let _v = LazyObjectSingletonWithError.0; // => nested runtime 에러 발생 시킴
    Ok(()) // => not reachable because of the error above
}
```

이를 실행하면 다음과 같다:

```bash
$ cargo run
         0 us, Started
   5002398 us, First  read from LazyObjectSingleton.0
   5002446 us, Second read from LazyObjectSingleton.0
thread 'main' panicked at 'Cannot start a runtime from within a runtime. This happens because a function (like `block_on`) attempted to block the current thread while the thread is being used to drive asynchronous tasks.', /Users/yyu-nim/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.20.0/src/runtime/thread_pool/mod.rs:89:25
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

예상했던 대로, `LazyObjectSingleton`의 경우 tokio의 async sleep()이 의도대로 잘 호출되어
5초 sleep을 유발한 것을 확인할 수 있다.
`AsyncOnce`를 안 쓰고 대신 tokio runtime을 생성했던 `LazyObjectSingletonWithError`의
경우에는 *"Cannot start a runtime from within a runtime"* 라는 (유명한) 에러가 발생했음을
눈여겨 볼 것.