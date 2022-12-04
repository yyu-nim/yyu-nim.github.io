---
layout: post
title:  "lldb <1> - programmatic debugging"
date:   2022-12-04 12:00:00 +0930
categories: rust lldb debugging
---

오랜시간 동안 사랑받아 온 디버깅 방법이 있다. 그것은 바로, 디버그 코드를 삽입하여 
`println`을 통해 콘솔이나
파일로 프로세스의 상태를 출력하는 것이다. 

왜 이 방법이 아직까지 유효한 것일까. 일단 `GDB/LLDB`와 같은 디버거를
사용하는 것은, 탐색해야 할 프로세스 "상태"가 방대하거나, 탐색 시점의 조건이
복잡할 경우, 수동적인 노력이 많이 들고 디버그 코드의 삽입이 단순하지 않다 
(물론 gdb + python 연동과 같은 방법이 존재하지만). 가령, 수천만개의 변수에 대해
corruption 여부를 조사한다던지, 여러 상태들을 조합해서 논리적으로 도달할 수 없는
상태의 여부에 대해 추론한다던지 등의 기능은 디버거의 기능을 넘어선다.
`Metric/log` 기반의 분석은, 버그를 나타내는 프로세스의 상태가 미리 전처리되어
telemetry pipeline으로 전달되지 않았다면 활용에 한계가 있고, 상태 표현에 있어
프로세스 전체 상태를 나타내기 어려우므로 디버깅 정보의 유실이 있을 수밖에 없다. 가령,
프로세스의 코어 덤프를 매 초마다 주기적으로, 혹은 메모리 상태 변경점을 실시간으로
기록하는 것은 현실적으로 어려울 것이다.

디버그 코드를 삽입하는 방식은, 앞의 두 방식보다 수월하게
디버깅 정보를 취합할 수 있도록 해주는데, 1) 프로세스의 상태에 손쉽게 접근이 가능하고 (자신의
주소 공간에 접근하면 되므로), 2) 프로세스 상태를 요약/계산하기 위해 프로그래밍 언어를
그대로 사용할 수 있다는 점 덕분이다. 다만, 디버그 코드가 삽입된 이후에 재현 평가를 통해, 
버그 상황의 프로세스 상태에 재진입해야 원하는 정보를 취합할 수 있다는 점이 큰 단점이
될 것이다. 즉, `버그 발생 -> 취합되어야 할 정보를 파악하여 디버그 코드 삽입 -> 버그 재현`의
사이클을 반복해야 하는 것이 부담이 될 것이다. 특히 재현이 어려운 버그라면 더더욱.

만약, 프로세스가 버그에 부딪힌 시점의 <u>모든 상태에 접근이 가능</u>하고, 프로그래밍을 통해
<u>상태들을 취합/요약</u>할 수도 있고, 이를 위해 prod code에 직접 
디버그 코드를 심지 않아도
되고, 따라서 <u>프로세스 런타임에 미리 정보를 취합하지 않아도 되는</u> 그런 방법이 있다고 하면
좋은 디버깅 옵션이 될 수 있지 않을지?

오늘의 주제는, `LLDB` 라이브러리를 활용해서 core dump, process, 혹은 executable을
가지고 프로그래밍을 활용하여 디버깅을 자동화 할 수 있는 방법에 대한 것이다.
앞서 잠깐 언급한 GDB + python 으로 비슷한 효과를 줄 수도 있지만, 그 방법에 비해 
좀 더 범용적인 platform에서, statically-typed language로 디버깅 자동화를 할 수 있다는데
의의가 있다고 생각한다. 구체적으로 Mac OS에서 RUST + LLDB 의 조합으로 애플리케이션 디버깅을
자동화할 수 있는 방법에 대해 살펴보려 한다. 

---
일단, LLDB를 사용하기 위해 소스 빌드가 필요하다. 다음의 절차가 필요하다.

```bash
$ git clone https://github.com/llvm/llvm-project.git
$ cd llvm-project
$ cmake -S llvm -B build -G Ninja -DCMAKE_BUILD_TYPE=debug
$ cd build
$ ninja
```

빌드를 위해 `cmake`와 `ninja`를 사용하였는데, 만약 설치한 적이 없었다면 
아래와 같이 homebrew로 설치가 가능하다.
```bash
$ brew install cmake ninja
```

설치가 완료되었다면, 아래의 두 환경 변수를 설정한다. 원한다면 .bashrc, .zshrc 등
쉘의 기본 세팅에 등록해 두도록 한다. 
```bash
$ export LLVM_ROOT=/Users/yyu-nim/llvm-project
$ export LLVM_BUILD_ROOT=/Users/yyu-nim/llvm-project/lldb
```

이제 `lldb`, `lldb-sys`라는 crate을 사용하여, 빌드가 잘 되는지를 확인할 차례이다.
아래와 같이 프로젝트를 구성해본다.

<script src="https://gist.github.com/yyu-nim/5375009f9963220d21a2a8793814f121.js"></script>

<script src="https://gist.github.com/yyu-nim/348796f48cccaa3a4b5eda109ead1a4c.js"></script>

앞의 `LLVM_ROOT`, `LLVM_BUILD_ROOT`를 제대로 설정하였다면, 일단 `cargo build`는 
잘 될 것이다. 하지만, `cargo run` 실행 시 아래와 같은 라이브러리 에러가 발생할 것이다. 
```bash
 % cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/lldb-debug1`
dyld[33438]: Library not loaded: @rpath/LLDB.framework/Versions/A/LLDB
  Referenced from: <5B405FB6-9FC2-35DE-B823-3EE5B470BF39> /Users/yyu-nim/lldb-debug1/target/debug/lldb-debug1
  Reason: tried: '/System/Volumes/Preboot/Cryptexes/OS@rpath/LLDB.framework/Versions/A/LLDB' (no such file), '/Library/Frameworks/LLDB.framework/Versions/A/LLDB' (no such file), '/System/Library/Frameworks/LLDB.framework/Versions/A/LLDB' (no such file, not in dyld cache)  
zsh: abort      cargo run
```

[lldb crate](https://crates.io/crates/lldb) 에 있는 설명대로, 
아래의 환경 변수를 set해주어 통과할 수 있다.
```bash
export DYLD_FRAMEWORK_PATH=/Applications/Xcode.app/Contents/SharedFrameworks
```

다만, 만약 `Xcode`를 설치한 적이 없다면, `/Applications/Xcode.app/Contents/SharedFrameworks`라는
디렉토리가 아예 존재하지 않아 에러가 날 것이다. 이 때는 App Store를 통해 
Xcode를 우선 설치하도록 한다. Xcode 버전에 따라 Mac OS 버전 업그레이드가 필요할 수 있음에 유의.

Xcode 설치와 `DYLD_FRAMEWORK_PATH`가 설정 완료되었다면, `cargo run`은 통과할 것이다.
```bash
% cargo run                                                                   
    Finished dev [unoptimized + debuginfo] target(s) in 0.04s
     Running `target/debug/lldb-debug1`
Hello, LLDB!
Bye,   LLDB!
```

이제 LLDB를 사용할 준비가 되었다!