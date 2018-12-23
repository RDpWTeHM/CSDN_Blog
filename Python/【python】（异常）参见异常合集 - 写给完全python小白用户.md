# 【python】（异常）参见异常合集 - 写给完全python小白的用户

> 欢迎转载，请保留本段：版权声明：本文为博主原创文章，转载请注明出处。	https://blog.csdn.net/qq_29757283/article/details/85225386

**内容说明**
这一篇博客是最简单的异常情况，基本上是发生在脚本“编译成字节码”过程中就抛出异常的情况。
（大部分都是语法问题）

**本文背景**
我个人目前是不需要这东西啦。
但是像我自己也是其它领域（比如其它编程语言）的小白
（比如 前端（bootstrap），node.js 和 其它等等 ）。
偶尔就会需要用到其它的编程语言来做事情，但是拿到的，搜到的资料中的代码可能并没有经过严格的测试之后贴上去（我个人偶尔也会按思路简要写些代码贴出来，不过没有测试运行过，测试过我一般会说明，不然既是 CV 即用的），所以照抄运行就会有一些低级错误发生。

但是正如我说的，有时候不得不使用到其它编程语言，但是又不会一直在用那门语言，只是一部分需要用到，这样去学一整门语言的语法等等成本还是有点高的。

所以我个人乐见大家用 Python 来实现一些东西，即使不懂 python，只是临时用一下也 OK。

这篇 blog 就来提供你们可能遇到的一些很通常的，常见的问题的原因和解决方式。

**本博客不定期更新，欢迎在评论区贴出你们的问题，在我知道原因和答案的情况下就会解答，并且可能加入成为本文的一部分**

<u>希望本文在未来能够囊括一切 Python 的基本异常问题，</u>
<u>希望你想要解决的基本异常问题都能在本文找到。</u>

@[toc]( )

## 版本语法异常

Python3.x 的版本在语法上做了相当大的改动。很多 Python2 版本的代码没有修改一般都会报一些错误。
但是还好，大部分代码都能改改常见的地方就能在 python 3 中运行。
而对于移植难度很大的代码，个人建议不如直接用 python 2 解释器运行。
（如果有修改和定制的需要，你已经脱离了本文面向的读者群，这里提供一个使用进程间通信来使用 python 2 代码的建议。相信你会知道如何写 Python 的进程间通信的。）

### 表达式改为函数造成的一系列异常

如果不明白标题没有关系，看到标题留个印象；未来有意学习 Python 的时候，你就有了看到类似名词的实例场景；有助于将概念和实例/应用结合起来理解。

找到你对应的异常，对上号，按 solution 修改一下，即可。

#### python2 和 python3 print 不同造成的异常

在 Python 2 中，`print` 是一个“表达式”，在 Python 3 中，`print` 是一个函数。
所以将 print 改成函数形式即可：

##### 复制 `print` 语法异常(`SyntaxError`) 现象

![print 该为函数问题](https://img-blog.csdnimg.cn/20181223172856761.png)

```shell
  File "<ipython-input-2-20f88f935616>", line 5
    print "IndexError: ", e
                       ^
SyntaxError: Missing parentheses in call to 'print'
```

##### solution

改成函数形式的意思就是 `print()` :point_left: 这么使用 print 即可。

```python
try:
    raise IndexError("out of range")
except IndexError as e:
    print("IndexError: ", e)
```

运行的到预期输出：
![python print 版本问题 - solution 运行输出](https://img-blog.csdnimg.cn/20181223173253594.png)

#### raise 语法变化后的异常

> 虽然这个问题本质上不适合放在 **“表达式改为函数造成的一系列异常”** 这个标题下，
> 但是因为和 `print` 的代码写法形式很相像，并且上一节刚好使用到了它，
> 所以将这个问题放在这个位置。

可能有部分读者发现了：

![N/A](https://img-blog.csdnimg.cn/20181223181547412.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5NzU3Mjgz,size_16,color_FFFFFF,t_70)

上面“复制现象”中的 `raise IndexError("out of range")` 和手上的代码长的不太一样。
> **上面的 `raise IndexError("out of range")` 写法在 Python 2 中也是正确的！！！**

##### 复制 raise 语法变化后的问题现象
如果你使用的是一份 Python 2 的源码，那么它可能更像这么编写：

```python
try:
    # ...run many things...
    _var = []  # consider "[]" as a function to return a list
    if not _var:
        del _var
        raise RuntimeError, "no result"
except RuntimeError as e:
    print "RuntimeError", e
    raise
else:
    pass # ... more code, deal with "_var"...
```

> 正如我所说的，这是一份 Python 2 的代码，所以我将 line 8 按照 Python 2 的语法书写。
> 
> **当然，这段代码在 Python 2 的解释器中是可以正常运行的。** 它会抛出一个 `RuntimeError`  *而非语法错误。*

在 Python 3 的解释器中运行：
```shell
  File "<ipython-input-13-0b37b768c203>", line 6
    raise RuntimeError, "no result"
                      ^
SyntaxError: invalid syntax
```

和预期的一样，它给出了一个语法错误： `SyntaxError: invalid syntax`

##### Solution

"异常" 都是“类”，raise 语法的行为是抛出异常的实例。
所以不要使用 `,` 的写法，像前面的显式地实例化异常类，然后抛出即可：
```python
try:
    # ...run many things...
    _var = []  # consider "[]" as a function to return a list
    if not _var:
        del _var
        raise RuntimeError("no result")
except RuntimeError as e:
    print ("RuntimeError", e)
    raise
else:
    pass # ... more code, deal with "_var"...
```
修改（line 6 和 line 8）后的运行结果：
![N/A](https://img-blog.csdnimg.cn/20181223184736953.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5NzU3Mjgz,size_16,color_FFFFFF,t_70)

**注意，这个异常是正确的行为。因为它已经不是语法错误了。**
> 注意到上面运行结果的截图中，有 ”RuntimeError no result“ 这一行输出，
> 这是在 line 7 捕获异常之后，line 8 的输出。
> 之后这段代码又将这个异常继续抛出。
> 所以修改之后的代码行为和预期一致。

对于本文给出的该段 demo 程序，你可以修改 line 3，使 `_var = [1, 2, 3, ]`，只要列表不为空就可以。

然后运行一下，这样就不会抛出 `RuntimeError` 了。


## Note: 更多 Python 程序上的基础问题（非代码本身bug）收集补充中...

#### 加入本文的编辑？

在 GitHub 上 [fork 这个地址 :link:](https://github.com/RDpWTeHM/CSDN_Blog)，

修改 Python 目录下的
[本文]: `【python】（异常）参见异常合集 - 写给完全python小白用户.md`

然后push 到你自己的 fork 仓库上，最后向我发起 pull request 即可



#### 你也可以在 GitHub 上的 issue 板块贴出你遇到的问题

**issue 板块跳转 [链接:link:](https://github.com/RDpWTeHM/CSDN_Blog/issues/1)**



