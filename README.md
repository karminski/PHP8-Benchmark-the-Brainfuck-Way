README.md
---------

# PHP8 Benchmark the Brainfuck Way

PHP8 已经发布了, 带来了全新的, 令人兴奋的 JIT 编译引擎. 根据官方文档描述: "它在综合基准测试中的性能提高了大约 3 倍, 在某些特定的长期运行的应用程序中提高了 1.5-2 倍. 典型的应用程序性能与 PHP 7.4 相当.".  

官方性能测试结果如下图:  

![PHP 8 JIT official test results](./assets/images/scheme.svg)

# 那么实际情况如何?

于是我写了个简单的跑分 [demo](./src/run.php), 是一个从 Go 语言移植过来的 [brainfuck 语言](https://zh.wikipedia.org/wiki/Brainfuck) 解释器. 

源码见: [run.php](./src/run.php). 里边顺便还包含了 brainfuck 实现的 [曼德博集合(Mandelbrot set)](https://zh.wikipedia.org/wiki/%E6%9B%BC%E5%BE%B7%E5%8D%9A%E9%9B%86%E5%90%88). 程序即用 PHP 编写的 brainfuck 语言解释器运行 brainfuck 编写的生成曼德博集合的程序. 然后计时并比较其性能.  

我的机器配置为: Intel(R) Core(TM) i5-7600K @3.8GHz.

### PHP 7.4.1 测试结果
![php7.4.1-brainfuck-interpreter-benchmark](./assets/images/php7.4.1-brainfuck-interpreter-benchmark.png)


### PHP 8.0.0 测试结果
![php8.0.0-brainfuck-interpreter-benchmark](./assets/images/php8.0.0-brainfuck-interpreter-benchmark.png)

# 结论

从测试数据看, PHP 7.4.1 耗时 14m0.138s, PHP 8.0.0 耗时 11m36.697s. 单从这一测试结果讲, 性能提升了 1.2 倍. demo 程序大部分时间都在做 ```$foo++;```, ```$foo--;```, 数组查找并赋值这样的操作, 其中最多的 ```op_inc_dp``` 逻辑, 即 ```$dataPtr++;``` 执行了 4,453,036,023 次.  

| operator executed in executeBf | execute times    |
|--------------------------------|------------------| 
| op_inc_dp                      |  int(4453036023) |
| op_dec_dp                      |  int(4453036013) |
| op_inc_val                     |  int(179053599)  |
| op_dec_val                     |  int(177623022)  |
| op_out                         |  int(6240)       |
| op_in                          |  int(0)          |
| op_jmp_fwd                     |  int(422534152)  |
| op_jmp_bck                     |  int(835818921)  |

如果 PHP 8 的确使用了 DynASM, 那么根据我对 DynASM 的实践和理解, 最简单且行之有效的优化方式是, 尽量提前初始化变量. 即:

```php
$data = array();
$dataPtr = 0;
for($i=0;$i<65535;$i++){  <--------------------------------+
    $data[$i] = 0;                                         |
}                                                          |
for($pc = 0; $pc < count($program); $pc++){                |
    // if(empty($data[$dataPtr])) $data[$dataPtr] = 0; ----+
    switch ($program[$pc][self::operator]) {
        case self::op_inc_dp:
```

将判断 ```$data``` 是否为空然后再初始化这里修改为直接初始化 ```$data``` 为一个长度为 65535 的数组. (当然本身这个判断也是很耗费性能的, 具体是否经过了 JIT 还不得而知, 我接下来会更详细的测试并阅读源码, 来了解 PHP 8 JIT 是如何运作的.).

这样修改后, 运行时间缩小到了 8m34.111s, 同样 PHP 7.4.1 耗时为 9m44.762s 性能提升了 1.137 倍. 

就本次测试结果而言, PHP 8 引入了 JIT 的性能提升是显著的. 这为 PHP 带来了新的可能性. 但同时我们也看到官方测试中, 一些复杂的应用 (比如WordPress) 提升很微弱. 目前还不知道是因为 JIT 仍处于初期阶段导致性能提升不显著, 还是因为 WordPress 由于是既存项目, 代码不可能为 JIT 专门优化过而导致性能提升不明显. 这部分还需要通过详细的 benchmark (比如使用火焰图) 来进行研判.

作为一个从 PHP 4 一路使用过来的老用户, 对 PHP 的感情是复杂的, PHP 现在十分缺乏一个强力的生态来重新唤醒. 而这需要广大 PHPer 的共同努力. 期待未来 PHP 能有更好的表现.

# Reference
- [The Original brainfuck interpreter source code](https://github.com/kgabis/brainfuck-go/blob/master/bf.go)
- The mandelbrot set fractal viewer in brainfuck written by Erik Bosman

# License
- [The MIT License (MIT)](http://opensource.org/licenses/mit-license.php)


# One more thing

顺便这个程序的 Go 测试结果为:  

![go-brainfuck-interpreter-benchmark](./assets/images/go-brainfuck-interpreter-benchmark.png)

