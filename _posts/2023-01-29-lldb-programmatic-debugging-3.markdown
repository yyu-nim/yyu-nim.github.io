---
layout: post
title:  "lldb <3> - extracting values from various types"
date:   2023-01-29 08:00:00 +0930
categories: rust lldb debugging primitive types
---

이전 포스트에서 `SBValue`를 분해하여 탐색하는 법을 소개하였는데, 이번에는 `SBValue`에서 값을
추출하는 내용을 다루어 보려 한다. `println!("- LLDB variable: {:#?}", my_class_instance)`를 써서
같이 전체 구조를 확인해 볼 수 있다:
```text
- LLDB variable: SBValue { (MyClass) ::my_class_instance = {
  member1 = size=3 {
    __tree_ = {
      __begin_node_ = 0x00006000000b5180
      __pair1_ = {
        std::__1::__compressed_pair_elem<std::__1::__tree_end_node<std::__1::__tree_node_base<void *> *>, 0, false> = {
          __value_ = {
            __left_ = 0x00006000000b51a0
          }
        }
      }
      __pair3_ = {
        std::__1::__compressed_pair_elem<unsigned long, 0, false> = (__value_ = 3)
      }
    }
  }
  member2 = false
  member3 = 40
  member4 = {
    std::__1::__atomic_base<unsigned short, true> = {
      std::__1::__atomic_base<unsigned short, false> = {
        __a_ = {
          std::__1::__cxx_atomic_base_impl<unsigned short> = (__a_value = 50)
        }
      }
    }
  }
  member5 = {
    std::__1::__atomic_base<bool, false> = {
      __a_ = {
        std::__1::__cxx_atomic_base_impl<bool> = (__a_value = true)
      }
    }
  }
  member6 = size=3 {
    std::__1::__vector_base<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, std::__1::allocator<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > > > = {
      __begin_ = "A"
      __end_ = ""
      __end_cap_ = {
        std::__1::__compressed_pair_elem<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > *, 0, false> = (__value_ = "")
      }
    }
  }
  member7 = "D"
  member12 = {
    member8 = 60
    member9 = "E"
    member10 = 0x00000000ffffffff
    member11 = nullptr
  }
}
 }
```
여기에서 값들을 추출해서 이걸 RUST struct로 만들어 주는 것이 목표이다. 그래야 RUST 프로그래밍으로
상태 정보들을 조합해서 디버깅을 자동화할 수 있을 테니까. 
SBValue의 string-formatted debug 정보를 보고, 굳이 `children()`을 사용해서 
destructuring을 해야 하는지, 단지 string parsing 으로 필요한 정보를 추출하면 되지 않는지
의문이 들 수 있는데, 

* collection type에 대해서는 정보가 충분히 표시 안된다는 점 (e.g., vector, set),
* stdlib 구현에 따라 내부 변수명이 다르게 표시될 수 있다는 점 (e.g., string, atomic),
* 예외 처리가 귀찮을 수 있다는 점 (e.g., string member1 = "member1") 

등으로, string parsing은 가능한 피하고, strongly-typed 객체들을 LLDB API를 사용하여 값을
추출하는 방식을 권장한다. 아래는, std::vector 타입의 `member1`이 가진 아이템들을 하나씩 순회하면서,
값을 추출하여 `val`에 저장하는 코드이다.

```rust
            match my_class_member.name() {
                "member1" => {
                    for member1_item in my_class_member.children() {
                        let val : u32 = member1_item.value().parse().unwrap();
                        ... /* do something with 'val' */
                    }
                }
```
이전에 언급했듯, `my_class_member`가 std::vector 콜렉션 타입인 경우, children() 으로
순회를 할 수 있고 그 각각의 아이템은 (여기에서는 `member1_item`) SBValue 타입을 가진다.
원래의 cpp 애플리케이션에서 std::vector의 원소가 uint32_t라는 것을 우리는 이미 알고 있으므로,
이제 `member1_item`은 uint32_t 타입일 것이라 추측할 수 있다. 
`value()` 메서드는 SBValue가 가리키는 값을 string으로 리턴해주기 때문에, 우리가 이미 알고
있는 타입으로 casting 해주어야 한다. RUST의 타입 추론 기능을 이용하면, 편한데, `let val` 오른쪽에
타입을 명시적으로 적어주면 (i.e., `u32`), `value().parse()`가 자동으로 string -> 해당 타입으로
parsing 한 다음 Option<T>으로 리턴해준다 (i.e., Option<u32>). 

마찬가지의 방법을 `member2`, `member3`에 적용해 볼 수 있다. 우리는 이미 이것들이 
collection type이 아닌, primitive type임을 알고 있으므로 타입 추론 기능을 활용한 
string parsing을 수행한다.
```rust
                "member2" => {
                    let val : bool = my_class_member.value().parse().unwrap();
                    ...
                }
                "member3" => {
                    let val : i32 = my_class_member.value().parse().unwrap();
                    ...
                }
```

std::atomic 타입은 약간 특이하다. `member4`, `member5`의 debug output을 보면, 
"50", "true"라는 값들이 다른 구조체들 깊숙이 들어가 있는 것을 볼 수 있다. 이걸 끄집어 내려면, 
`children()`으로 순회하여 첫번째 아이템에 대해 primitive type 추출 방법을 적용해야 한다.
여기서는 `for ... in` 대신, children()의 iterator에 직접 next()를 호출하여 첫번째 아이템을
선택하도록 하였다.
```rust
                "member4" => {
                    let val: u16 = my_class_member.children().next().unwrap().value().parse().unwrap();
                    ...
                }
                "member5" => {
                    let val: bool = my_class_member.children().next().unwrap().value().parse().unwrap();
                    ...
                }
```

`std::vector<std::string>` 타입은 현재 문제가 있다. `member6`의 debug output 출력된 결과를 
위에서 확인해 보면, "A"만 제대로 나타나고 나머지 "B", "C" 는 표시가 되지 않는다. 이를 children()을
통해 아이템들을 순회하면 조사를 해보아도 마찬가지이고, 이는 std library & OS 에 따라 결과가 다르게
나타나는 듯 하여, 추가 조사가 필요하다. 참고를 위해 debug output 결과만 남겨둔다:
```rust
                "member6" => {
                    for member6_item in my_class_member.children() {
                        println!("- member6_item: {:?}", member6_item);
                    }
                }
```
```text
- member6_item: SBValue { (std::basic_string<char, std::char_traits<char>, std::allocator<char> >) [0] = ""
 }
- member6_item: SBValue { (std::basic_string<char, std::char_traits<char>, std::allocator<char> >) [1] = ""
 }
- member6_item: SBValue { (std::basic_string<char, std::char_traits<char>, std::allocator<char> >) [2] = ""
 }
```

std::string 타입도 비슷한 문제가 있어 해결책을 구하는 중이다. 현재는 debug output 출력을
단순히 string parsing 하여 값을 추출하는 방법 밖에는 찾지 못하였다. 아래 코드는, 
member7의 debug output에서 첫/끝 따옴표 사이에 존재하는 character 들을 string 으로 만드는 
naive approach를 택하였다. 이는 앞서 언급한 string parsing의 단점들이 그대로 존재하게 된다.
```rust
                "member7" => {
                    let val_str = format!("{:#?}", my_class_member).to_string();
                    let pos_start = val_str.find("\"").unwrap() + 1;
                    let pos_end = val_str.rfind("\"").unwrap();
                    let val : String = (&val_str[pos_start..pos_end]).parse().unwrap();
                    ...
                }
```

MyInnerClass 타입의 `member12`의 경우, `children()`으로 하위 SBValue들을 한번 더
순회하여 값을 추출할 수 있다:
```rust
        for my_inner_class_member in my_class_member.children() {
            match my_inner_class_member.name() {
                "member8" => { ... }
                "member9" => { ... }
                "member10" => { ... }
                "member11" => { ... }
                unknown_name => { ... }
            }
        }
```

지금까지의 추출 방법을 바탕으로 MyClass 라는 RUST struct를 만들어
`println!("- RUST my_class: {:#?}")`로 찍어보면 아래와 같을 것이다:
```rust
- RUST my_class: MyClass {
    member1: [
        10,
        20,
        30,
    ],
    member2: false,
    member3: 40,
    member4: 50,
    member5: true,
    member6: [],
    member7: "D",
    member12: MyInnerClass {
        member8: 60,
        member9: "E",
        member10: "0x00000000ffffffff",
        member11: "0x0000000000000000",
    },
}
```

현재까지의 논의 내용을 종합하면, 아래와 같은 코드가 될 것이다:
<script src="https://gist.github.com/yyu-nim/5542e6c8d23fef532d63df83e58fe42c.js"></script>

여기까지 기본적인 값 추출 방법에 대해 살펴보았다. 다음에 커버해볼 내용은,
포인터 변수를 활용하여 배열 접근하는 방법, SBValue/SBValueRef/SBData/SBDataRef/SBType/SBTypeRef 의
관계에 관한 것이다. 