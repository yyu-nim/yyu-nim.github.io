---
layout: post
title:  "lldb <2> - destructuring SBValue"
date:   2023-01-28 22:00:00 +0930
categories: rust lldb debugging basics
---

LLDB를 사용하면 (GDB도 마찬가지이다), 현재 실행 중인 프로세스 혹은 코어 덤프로부터
상태를 조사할 수 있다. 설명을 위해, 일단 코어 덤프를 확보해 보도록 하자. 다음과 같은
cpp 프로그램을 작성하고 컴파일 해보도록 한다:

<script src="https://gist.github.com/yyu-nim/33c392b55bc2bd178971f417a29c085e.js"></script>

`main()` 함수가 하는 일은, 무한 루프를 돌면서 5초에 한번씩 Sleeping... 이라는 메시지를
출력해 주는 일이다. 이 와중에, 우리는 `gcore` 유틸리티를 사용하여 코어 덤프를 확보할 것이다.
코어 덤프에는, 프로세스가 가진 모든 쓰레드의 모든 스택 프레임들의 정보 + 글로벌 변수에 대한 정보가
포함되어 있고, 따라서 MyClass 타입의 `my_class_instance` 라는 글로벌 변수도 마찬가지이다.
코어 덤프로부터 `my_class_instance`의 멤버 변수들을 읽어와서 RUST struct로
만들어 주는 부분까지가 오늘의 포스팅 내용이다.

이 애플리케이션을 실행하고, 별도의 터미널에서 `gcore`를 실행하여 코어 덤프를 확보하자:
```bash
$ sudo gcore <pid>
$ ls -l /cores 
total 6683744
-r--------  1 yyu-nim  staff  3422076928  1 28 01:06 main-4807-20230127T160555Z
```

이제, 애플리케이션 파일과 코어 덤프 파일을 확보하였기 때문에, LLDB 실습을 진행해보도록 한다. 
일단, LLDB 디버거 런타임을 실행한다. 애플리케이션 파일의 경로와 코어 덤프 경로는 
적절히 main() 에 넘겨주도록 한다. 여기에서는 환경 변수 `EXEC`, `DUMP`를 사용하였다:
```rust
fn main() {
    let executable = std::env::var("EXEC").expect("Please provide the application binary path");
    let dump = std::env::var("DUMP").expect("Please provide the core dump path");

    SBDebugger::initialize();
    SBDebugger::terminate();
}
```

LLDB 디버거 런타임이 살아있는 동안은 (i.e., SBDebugger::initialize() ~ SBDebugger::terminate())
사용자의 디버깅 세션을 흉내내어 connection을 맺을 수 있다. 아래는, 코어 덤프로부터 
프로세스 상태를 `p`에 복원하는 방법이다:
```rust
    let debugger = SBDebugger::create(false);
    let target = debugger.create_target_simple(executable.as_str()).unwrap();
    let p = target.load_core(dump.as_str()).unwrap();
```

프로세스 `p`로부터 글로벌 변수를 찾을 수 있는 방법이 몇 있는데, 가장 쉬운 방법은
모든 쓰레드의 모든 스택 프레임을 조사하는 것이다. 스택 프레임 타입은 `SBFrame`인데,
여기에 `all_variables()`라는 메서드가 있어 이것을 통해 로컬/글로벌 변수들을
전부 훑을 수가 있다. 이를 바탕으로, 다음과 같은 방법을 활용할 수 있다:
```rust
    let mut my_class_instance = None;
    for thr in p.threads() {
        for frame in thr.frames() {
            for var in frame.all_variables().iter() {
                if var.name() == "::my_class_instance" {
                    my_class_instance = Some(var);
                }
            }
        }
    }
```
한가지 주의할 점은, 글로벌 변수가 별도의 namespace내에 포함되지 않았다면 
`::`를 붙이면 되고, namespace에 포함되는 경우에는 `<name_space>::`를
붙이면 된다는 것 (e.g., `pos::debugInfo`). 
`my_class_instance`는 찾고자 하는 변수의 이름이다.

찾은 변수는 `SBValue` 라는 타입으로 존재한다. 이것은 내부적으로 
**unsafe** LLDB API 조합으로 구현되어 있고 이 구현 방법 및 타입 간의 관계에
대해서는 문서가 충분하지 않다. `SBValue`가 가진
메서드 중 가장 유용한 것 중의 하나는 
`pub fn children(&self) -> SBValueChildIter` 이다. 
`SBValue`가 가진 내부 멤버 변수들을 끄집어 낼 수 있도록 destructuring 해주는 
기능이라 보면 되겠다. 예를 들면, `MyClass` 를 SBValue 타입으로 가지고 있으면,
`children()` 메서드를 통해서 member1, member2, ..., member12 까지 전부
순회를 할 수 있게 된다. `children()` 으로 순회할 때 각각의 아이템 역시
SBValue 타입이고, `name()` 을 통해 변수의 이름을 출력해볼 수 있다. 이를 바탕으로,
다음의 방법으로 각각의 값들을 꺼내볼 수 있을 것이다:
```rust
        let sb_value: SBValue; // assume that it contains "my_class_instance"
        ...
        for my_class_member in sb_value.children() {
            match my_class_member.name() {
                "member1" => {}
                "member2" => {}
                "member3" => {}
                ...
                "member12" => {}
                unknown_name => {
                    eprintln!("Unknown member name! {}", unknown_name);
                }
            }
        }
```

위에 언급했던 대로, `my_class_member` 역시 SBValue 타입인데, 이 덕분에
std::vector, std::set 등의 collection 타입에 대해서는 `children()`을 통해
내부의 item 들을 하나씩 SBValue로 다시 얻어올 수 있게 된다. 즉, 여러번 nested 된
데이터 구조가 있으면, `children()`을 반복적으로 호출하여 최종적으로 primitive type
(혹은 string parsing 으로 쉽게 값을 꺼내올 수 있는 상태까지)이 보일 때 값을 
꺼내오면 된다. 이를 염두에 두고, member1과 member6는 다음과 같이 destructing 할 수 있을 것이다:
```rust
            match my_class_member.name() {
                "member1" => {
                    for member1_item in my_class_member.children() {
                        ...
                    }
                }
                ...
                "member6" => {
                    for member6_item in my_class_member.children() {
                        ...
                    }
                }
```

순서가 약간 바뀐 듯 하지만, member2, member3와 같은 primitive type들은 
어떻게 값을 추출하는 것이며, atomic data structure의 경우 (member4, member5)와, 
다른 객체 타입을 멤버로 가지고 있는 경우 (member7)는 어떻게 처리해야 할까?

포스트가 길어지는 것 같아 한번 끊고, 이어가도록 하겠다.
