---
layout: default
title:  "Linux Kernel中对变量益处问题的处理"
categories: Linux Tcp
---

### Linux Kernel中对变量益处问题的处理



在Linux内核中比较时间时，我们基本都是取jiffies的值进行比价，但有个问题：**如果jiffies溢出了怎么办？**



在内核中我们都是使用`time_before` `time_after`来比较时间的前后，来看这两个宏定义的实现：

```c
#define time_after(a,b)     \
    (typecheck(unsigned long, a) && \
     typecheck(unsigned long, b) && \
     ((long)((b) - (a)) < 0))
#define time_before(a,b)    time_after(b,a)
```

对于`time_after`，首先检查传入两个参数的类型（必须是unsigned long），然后进行相减，如果时间a在b之后，`(b) - (a)`应该是个很大的正数，强制转化为`(long)`之后，是个负数，结果是正确的。

但如果a在取jiffies时出现了溢出，那么此时a应该是一个比较小的数，如果需要`(b) - (a)`同样是一个很大的正数（大于long类型的最大值，这样转换为long之后还是负数，保证了结果的正确），则a和b的值就不能相差太大（大于long类型的最大值），例如如果a在溢出时，b的取值是小于long类型的最大值的，那么此时使用这种方式就是不对的，所以使用这种方式来处理变量益处后的比较，必须要满足一个条件：**相比较的两个值之间不能相差太大（大于这个值所属有符号类型的最大值）**，在实际使用场景中，比如判断时间或者tcp的seq num（before 和 after两个宏也是这个原理），都不会出现相差如此之大的场景，所以可以放心使用。



