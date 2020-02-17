
#!title:    Scheme解释器
#!date:     2018-08-16
#!authors:  Mikukonai
#!cover:    
#!type:     原创
#!tags:     函数式编程,计算机科学,Scheme


#!content

<iframe class="MikumarkIframe" src="./html/Scheme解释器.html" height="700px"></iframe>

> **说明**：解释器项目已经冻结，原则上不再继续维护。相关的工作全部转移到[AuroraScheme](https://github.com/mikukonai/AuroraScheme)项目，该项目是对解释器项目的继承和发展。

> 尽管上面的测试用例全部运行无误，但是目前V3和V4在设计上都有重大的错误，且V4有部分特性和内置函数尚未实现。由于项目已冻结，因此不再改正。

2018年10月18日的自我评价：

> 你搞的这个解释器啊，naive！

> 首先你没有卫生宏，没有卫生宏的Scheme和咸鱼有什么区别？？？

> 其次你的词法作用域真的sound吗？不要以为你用各种说不清道不明的workaround搞定了阴阳谜题和Knuth的那个用例，就真的觉得万事大吉了。实际上你这个设计从根本上来讲就是不恰当的。你的闭包链本质上仍然是动态作用域那一套。

> 进一步说，你的这个解释器实际上是应该做形式化验证的。但是你既不懂coq也不懂haskell，所以验证不了，对不对？

> 你读得少、做得也不多。你花了两个月搞出这么一个trivial的小东西，真的很simple。

> 你的用例少得可怜，所以你没办法通过高覆盖的测试去验证你拍脑袋想出的各种小trick对不对？

> 你的代码架构脏得可以，毫无封装意识，毕竟你也没有正经学过JavaScript对不对？毕竟你连TSPL/SICP/EOPL/甚至HTDP都没读过嘛。你只是一个刚刚从The Little Schemer开始入门的FP新手菜鸟而已啊。

> 好了，不挖苦你了。王垠说解释器这东西既简单又复杂，简单就简单在它一是一、二是二，核心思想是简洁且有规律有逻辑可循的；但复杂就复杂在，作为一种虚拟机器，它的输入空间是千变万化的，你很难通过测试去彻底地验证它，同时也有工程上的复杂性。

> 本次开发的一个有益的经验是：从一开始就采取测试驱动的开发，以及相对比较好的版本控制。尽管你也没设计多少有一定复杂性的用例，但是这件事情是对的。值得鼓励。

> 接下来，你还有卫生宏和continuation两座大山需要翻越。搞定了这两个advanced的特性，即便MikuRec不能投入生产，即便MikuRec有种种缺陷，那也算是一件值得写入简历的作品了。

> 行了，就说到这里。10月18日的版本拉个基线出来，后续开发基于这个基线。祝你成功~

# 形式语法

```
         <Term> ::= <SList> | <Lambda> | <Quote> | <Quasiquote> | <Symbol>
        <SList> ::= ( <SListSeq> )
       <Lambda> ::= ( lambda <ArgList> <Body> )
     <SListSeq> ::= <Term> <SListSeq> | ε
   <ArgListSeq> ::= <ArgSymbol> <ArgListSeq> | ε
      <ArgList> ::= ( <ArgListSeq> )
         <Body> ::= <SList> | <Lambda> | <Quote> | <BodySymbol>
        <Quote> ::= ' <Term>
   <Quasiquote> ::= [ <SListSeq> ] | ε
       <Symbol> ::= IDENTIFIER
   <BodySymbol> ::= IDENTIFIER
    <ArgSymbol> ::= IDENTIFIER
```

其中IDENTIFIER是上文所述的标识符。

非终结符的FIRST集合：

|非终结符|FIRST|
|---|
|`&lt;Term&gt;`|`(` `[` `'` `SYMBOL`|
|`&lt;SList&gt;`|`(`|
|`&lt;Lambda&gt;`|`(`|
|`&lt;SListSeq&gt;`|`ε` `(` `[` `'` `SYMBOL`|
|`&lt;ArgListSeq&gt;`|`ε` `SYMBOL`|
|`&lt;ArgList'&gt;`|`(`|
|`&lt;Body&gt;`|`(` `[` `'` `SYMBOL`|
|`&lt;Quote&gt;`|`'`|
|`&lt;Quasiquote&gt;`|`[`|
|`&lt;Symbol&gt;`|`SYMBOL`|
|`&lt;BodySymbol&gt;`|`SYMBOL`|
|`&lt;ArgSymbol&gt;`|`SYMBOL`|

为简单起见，使用递归下降分析。因为已经消除了左递归，所以理论上是不会出现无限循环的。此外，最多只需要向前看两个符号，就可以决定产生式。

# 卫生宏

宏（macro）是一种元编程手段。简单地说，宏的用途是匹配代码中出现的某种模式，将其替换为事先定义好的其他结构。

宏是Scheme最强大的语言设施之一，许多语法糖（派生结构）实际上就是利用宏机制，在少数基本结构的基础上构造而成的。进一步，Continuation也可以利用宏对原始代码作CPS变换而实现。

C语言也提供了宏，但C语言的宏仅仅是字符串的简单替换，稍不注意，就会掉进坑里。例如，考虑下列代码：

```:c
#define SQUARE(x) (x*x)

SQUARE(a+1) //=>(a+1*a+1)
SQUARE(a++) //=>(a++*a++)
```

第一种情况是简单字符串替换造成的错误，但第二种情况很难说清楚，因为`++`运算符是有副作用的，这里并不知道程序员的真实意图。

简单宏替换会污染宏调用的上下文，因此是“不卫生”的。

Scheme创新性地引入了**卫生宏**的概念。所谓的卫生宏，指的是可以通过重命名等机制，保证宏替换过程不会污染宏调用所在的上下文。具体来说，是满足这样几点性质：

+ 宏定义中出现的自由变量，仍然是词法作用域的。即，在宏定义中出现的自由变量，其绑定以**定义**所在的上下文为准，而**不会**被宏调用所在上下文所覆盖。
+ 宏定义中引入的变量绑定，若宏调用所在上下文出现了同名绑定，则宏定义引入的变量会被改名，以避免污染宏调用上下文。

第一点保证了宏定义内部自由变量的词法作用域特性；第二点保证了宏展开过程不会污染宏调用上下文已有的绑定。但是，如果传入宏调用的参数是有副作用的（例如`set!`），则会出现上面C语言例子中的第二种情况，必须特别注意。

下面是一个例子：

```:scheme
(define var 100)
;; 宏定义
(macro test
  ((_ f a)
   (lambda (x) (f a x var))))
;; 宏调用
(define foo
  (lambda (x var)
    (test (lambda (x y z) (+ x y z)) x)))

#lang racket
(define var 10)
(define-syntax test
  (syntax-rules ()
    ((_ a)
     (+ a a))))
(test (begin (set! var (+ 1 var)) var))
```

每个宏都有宏关键字，用于唯一标识已定义的宏。宏和宏关键字必须在程序顶层定义，不可嵌套定义。局部变量绑定会覆盖宏关键字，而宏关键字会覆盖同级以及上级的变量绑定。

# 尾递归

发生在尾位置的递归调用，称为**尾递归**。R5RS中给出了尾位置的归纳定义。MikuRec在解释执行前，会对AST进行扫描，检测出尾递归并加以标记，在执行时进行尾递归优化。

此外，可以利用 MikuRec 提供的`while`循环，使用一种称为“[蹦床](https://en.wikipedia.org/wiki/Trampoline_\(computing\))”的方法，实现手工的尾递归优化。

# 延续传递风格（CPS）

在C等命令式语言中，可通过跳转指令实现控制流转向。在基于表达式的 Scheme 中，每个表达式都有一个对应的“上下文”。此上下文保存了表达式可见的变量绑定、函数调用栈和语法上下文等信息。在 Scheme 中，上下文用 Continuation（延续）来表示，并作为一等对象暴露给程序员。利用 continuation，程序员可以手动控制程序的控制流，实现非局部跳出、异常处理、协程、生成器等高级的控制结构。

可以使用MikuRec提供的宏机制，实现代码到代码的自动CPST。以下用S*指代S-term的CPS形式。

```:lisp
; 符号 S* =
(lambda (cont) (cont S))

; 函数 (lambda (x) M)* =
(lambda (cont)
  (cont (lambda (x)
          (lambda (k)
            (M* (lambda (m) (k m)))))))

; 调用 (M N)* =
(lambda (cont)
  (M* (lambda (m-res)
        (N* (lambda (n-res)
              ((m-res n-res)
               (lambda (a) (cont a))))))))

; 未经柯里化的多参数调用 (M0 M1 M2 M3 ...)* =
(lambda (cont)
  ; 以下仅仅是对每个AST节点进行简单的遍历CPST/重命名，并未体现求值顺序，可以理解成并行的
  (M0* (lambda (node0) ; 每个nodex的类型都是cont->...
  (M1* (lambda (node1)
  (M2* (lambda (node2)
  (M3* (lambda (node3)
  ; 从这里开始体现求值顺序，几乎等于是 A-Normal Form
  ((node0 node1 node2 node3) (lambda (res)
  ; 最后执行总的continuation
  ( cont res))))))))))))

; (if P T F)* =
(lambda (cont)
  (P*
   (lambda (p-res)
     (if p-res
         (T* cont)
         (F* cont)))))
```

# 参考资料

+ Danvy O, Filinski A. [Representing Control: a Study of the CPS Transformation](http://pdfs.semanticscholar.org/144b/7a68e040839f161ae9025e6e2c02ee4b08e2.pdf)[J]. Mathematical Structures in Computer Science, 1992, 2(4):361-391.
+ D.P. Friedman, M. Wand. [Essentials of Programming Languages](http://www.eopl3.com/).
+ 王垠. [怎样写一个解释器](http://www.yinwang.org/blog-cn/2012/08/01/interpreter/).
+ [CPS变换与CPS变换编译](https://zhuanlan.zhihu.com/p/22721931).
+ [Matt Might 的博客](http://matt.might.net/)
