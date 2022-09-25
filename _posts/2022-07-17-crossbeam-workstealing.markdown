---
layout: post
title:  "crossbeam <1> - work-stealing queue I"
date:   2022-07-17 09:25:00 +0900
categories: rust crossbeam
---

{% highlight rust %}
use crossbeam_deque::{Steal, Worker};
let w = Worker::new_fifo();
let s = w.stealer();
w.push(1);
w.push(2);
w.push(3);
assert_eq!(s.steal(), Steal::Success(1));
assert_eq!(w.pop(), Some(2));
assert_eq!(w.pop(), Some(3));
{% endhighlight %}

[crossbeam doc](https://docs.rs/crossbeam/0.8.1/crossbeam/deque/struct.Worker.html)

