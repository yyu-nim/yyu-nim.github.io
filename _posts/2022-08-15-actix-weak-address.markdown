---
layout: post
title:  "actix <7> - weak address"
date:   2022-08-15 00:30:00 +0900
categories: actix rust actor weak address
---

Actix examples 중에 [weak address](https://github.com/actix/actix/blob/master/actix/examples/weak_addr.rs) 에 대한
재밌는 예제가 있어 포스팅한다. 

**weak** 은 원래 RUST standard library 에서 쓰이는 용어로, 
쉽게 말하면 memory ownership 없이
memory 주소를 기억할 수 있게 해주는 방식이라 볼 수 있다. 
따라서, weak memory reference를 얻어올 때 reference count의 증가가 일어나지 않고, 그 반대의 경우인
weak memory reference가 scope을 벗어나 drop이 될 때에도 reference count의 감소가 일어나지 않는다.
이것은 암묵적으로, weak memory reference가 가리키는 영역은 해당 영역의 memory owner가 
마음대로 해제할 수 있음을 의미한다.

그렇다면, weak address를 사용하는 것은 memory safety를 포기한, 일종의 *unsafe* feature로 
보아야 하는지 의문이 생길 수 있는데, 결론적으로는 그렇지 않다. weak address reference는 그대로 접근할
수가 없고, 반드시 **upgrade()** 를 호출해서 아직 해당 영역이 유효한지를 확인 후 곧바로 reference count를
증가시킨 후 메모리 주소를 리턴해 주기 때문에, RUST의 memory safety 보장을 받을 수 있다고 볼 수 있다.
[actix example](https://doc.rust-lang.org/std/rc/struct.Weak.html) 에서는 
weak address를 다음과 같은 방식으로 upgrade하여 사용한다:

```rust
if let Some(client) = client.upgrade() {
    client.do_send(TimePing(Instant::now()));
    println!("⏰ sent ping to client {:?}", client);
} else {
    println!("⏰ client can no longer be upgraded");
}
```

개념적으로는 고개가 끄덕여지지만, 실제로 actix actor address의 weak reference가 의도대로 동작하는지
확인해보고 싶다면 아래의 예제를 확인해보도록 하자. 두개의 MyActor를 만들 때, 그 중 하나는 HashMap에
`Addr`을 값으로 저장하고, 다른 하나는 `WeakAddr`을 값으로 저장한다.   
전자의 경우에는, `Addr`의 lifetime이 HashMap의 lifetime과 같아지게 되는 반면 후자의 경우에는 
block scope 을 벗어나는 순간 lifetime이 다하여 actix runtime에 의해 적절한 시점에 drop이 호출된다.


```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use actix::prelude::*;

struct MyActor(&'static str);
impl Actor for MyActor {
    type Context = Context<Self>;

    fn stopped(&mut self, ctx: &mut Self::Context) {
        println!("MyActor({}) has stopped...", self.0);
    }
}

fn main() {
    let mut sys = System::new();
    let arc_map = Arc::new(Mutex::new(HashMap::new()));
    let cloned_arc_map = arc_map.clone();
    let weakrc_map = Arc::new(Mutex::new(HashMap::new()));
    let cloned_weakrc_map = weakrc_map.clone();
    sys.block_on(async move {
        {
            let addr = MyActor("arc_addr").start();
            cloned_arc_map.lock().unwrap().insert("actor_to_live", addr);
        }
        println!("addr is not out of scope since it has moved in to cloned_arc_map whose lifetime is the same as main's...");
        {
            // actor address 로부터 weak address를 얻는 방법은 downgrade()를 호출하는 것
            let weak_addr = MyActor("weak_addr").start().downgrade();
            cloned_weakrc_map.lock().unwrap().insert("actor_to_die", weak_addr);
        }
        println!("weak_addr is now out of scope...");
    });

    sys.run();
}
```
이를 실행해보면 아래와 같다:
```rust
$ cargo run
addr is not out of scope since it has moved in to cloned_arc_map whose lifetime is the same as main's...
weak_addr is now out of scope...
MyActor(weak_addr) has stopped...
```
`MyActor(weak_addr)`만 멈추고, `MyActor(arc_addr)`은 여전히 살아있음을 확인할 수 있다.
둘 중에 어느 쪽이 낫다라고 주장하는 것이 아니니 오해하지 마시길. RUST의 memory safety 보장을 만족하기 위해,
혹은 단지 컴파일러를 기쁘게 하기 위해,
<u>그다지 중요하지 않은 (즉, 나중에 on-demand로 validity를 체크해서 cleanup 해도 무방한)</u> 
메모리 영역에 대한 ownership을 불필요하게 오랫동안 가지는 경우를 막을 방법이 있다 정도로 해석하면 될 듯.
가령, Actor간의 wiring을 쉽게 하기 위해 모든 actor address를 임시로 담아두고 있는 lookup 레이어를 둔다던지 할 때,
weak address라는게 있으니 활용할 수도 있겠다 정도로 생각하면 되겠다.