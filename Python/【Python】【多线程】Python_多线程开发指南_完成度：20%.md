# Python 多线程开发指南

[TOC]

## 写在前面的话

本文大部分内容基于：
1. 《Python 核心编程》
2. 《Python3 cookbook》
3. 《流畅的 Python》

三本书。

### i. 用与不用多线程
- 首先我们需要明确一点 :point_right: 多线程肯定是有用的。
- 然后确认一个老生常谈的问题：Python 中的多线程对 *计算能力* 没有多大的提升（GIL 的限制）。
  > 另一方面，如果 Python 程序的 <u>多线程代码设计合理</u> 的话，（至少在 Python 这门语言中）是可以很简单地转变为多进程程序，而多进程程序是可以提高 **计算能力** 的。
  >
  > > 对于线程可以共享的变量，进程间也可以使用 multiprocessing.connection 模块中提供的工具来很容易的传递。

- 最后明确的一些 Python 多线程能起大作用的用途：重 IO 的程序中，GUI 程序中。
另外我个人比较喜欢 Python 多线程的用途：在需要进行多种工作的程序中，多线程可以简化软件结构。
	> 我个人觉得简化软件结构很实用。
	> 但是我不是指简化软件逻辑，因为多线程的引入本身就会造成一定程度上的逻辑变复杂。
	> 但是从我个人角度来看，通过程序的线性处理来实现一些多种不同的工作任务，需要一些软件设计上的技巧性，可能得不偿失。
	> 最后，或许等未来我了解到更多的编程 *技巧*，或许简化结构这一说就成了伪命题（比如可能最开始设计的软件结构就有些问题等）。
	> 总的来说，我个人还是认为多线程是很实用的！

最后的最后，引用 《流畅的 Python》中引用的对 Python 线程的总结：
来自 [“Generators: The Final Frontier”](http://www.dabeaz.com/finalgenerator/) :point_down:
![n/a](https://img-blog.csdnimg.cn/20190224144617574.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190224144702277.png)
> 图中三点我就不多做说明了，但是对于图中第一点，我认为还是有必要强调一下的，
> 因为在《流畅的 Python》一书中，对第一点的 “**翻译**” 如下：
> ![n/a](https://img-blog.csdnimg.cn/2019022414522137.png)
> 实际上，如果你英语 OK 的话，并且结合 `sleep` 代码，你应该知道 *great at doing nothing!* 这段直译并不是 *毫无作用*，而是一句双关语（Python 线程在不做任何事的时候作用最大）。
> 那么译者为什么这么翻译？
> > 可能译者结合了 105 张幻灯片的 **Answer: Nothing! That's what!** 并且“简化”了双关语之后这么翻译的。
> > 也可能是因为：实际上，如果仔细分析第一点示例的 `time.sleep(2)` 代码的话，了解过“协程”的朋友就会知道其实它可以通过编程协程的方式重新调整一下代码，然后就不需要线程，就能达到一样的效果。
> > 不过我个人认为协程比线程还要难理解那么一丢丢，并且协程在整个软件的设计上需要更多一点的技巧性（我是指一个进程需要多个在后台运行的不退出的函数/线程）。所以我还是更喜欢线程一点。
> >
> > 当然，如果你的某几个线程本身代码中有“sleep”，并且这个线程在执行完任务之后就会退出，那么，如果掌握了协程之后，就可以很容易的取代线程。**从这个角度而言**，线程确实是“毫无作用”。
> > 只不过我个人仍然不赞同 “python 线程毫无作用” 这个说法。
> > 因为这就相当于将一个软件简单的用它运行地快不快，性能好不好来定性这个软件有没有用途。
> 
> 总而言之，Python 线程很有用，只要不以错误的方式使用它。

### ii. 线程和进程，并发与并行
#### ii-1. 线程运行在进程内
一个计算机系统运行多个进程(processing)，一个进程中**可以**运行多个线程(threading)
（一个进程中至少有一个线程）。
> 如果编写一个程序(program) 没有用到多线程，
> 那么这个程序“启动”之后，成为系统中的一个进程，该进程上运行一个线程，这个线程就是主线程。
> 换句话说，一个程序/进程从一个主线程开始运行，代码中如果创建了新线程，创建的新线程就是“副线程”，以下是简单的图示：
> > ```
> >  run program => [processing]
> >                        ||
> >                        \/
> >                main thread
> >                     create thread
> >                     run the created thread ----> other thread
> >                    ||                                   ||
> >                    \/                                   \/
> >               keep running              running the thread-function
> > ```
> 
> 主线程（main thread）和副线程（other thread）的更多关系会在下文一一提及。

#### ii-2. 并发不是并行
Concurrency Is Not Parallelism (It's Better)[“并发不是并行（并发更好）”][^1]

理解 “并发不是并行” 这句话很简单，只要想象并发是在一个运行地非常非常快的 CPU 上交替地执行代码就可以了（即这是一个时分复用的设计）；而并行就是几个 CPU 同时运行地运行代码（因为有多个 CPU 可以用于执行代码）。
> 并发的作用的至关重要的，想象一下如果你的电脑只有一个 CPU，没有并发的话，它一次只能执行一个程序：
> 在桌面 -> 启动程序（退出桌面），运行程序，退出程序（自动启动桌面） -> 回到桌面 -> 启动另一个程序（退出桌面），运行程序，退出程序 -> 回到桌面 ...
> 而在程序内部也只能按顺序执行一句一句代码... 那样的生活不要太美好 :smile:

所以在 Python 上，编写多线程就是并发，编写多进程就是并行（在有多个 CPU 的主机上运行的条件下）。


[^1]: 《流畅的 Python》CH18: 使用 asyncio 包处理并发

## 1 入门 Python 多线程
试想一下你用了这么多年各种软件，遇到过多少次“点击”了之后，等了半天程序没有反应的情况 ？
是不是数不过来。
没有给用户一种反馈，用户体验真的很糟糕，如果你要写一个软件，你肯定不想成为写用户体验很糟糕的软件的一份子。
现在有了多线程（并发），你就可以和写出没有任何反馈的软件说拜拜了。

现在假设我们有一个命令行程序，这个程序在功能上看过去可能有点傻，但是我认为它还是很能说明问题的：
```python
#!/usr/bin/env python3
"""filename: thread0.py
Usage: $ python3 thread0.py <int>
"""
import sys
import time
def timeconsuming_job(cnt_time):
    time.sleep(cnt_time)

def main():
    try:
        print("current time: ", time.ctime())
        print("="*10 + "  start  " + "="*10)
        cnt_t = sys.argv[1]
        timeconsuming_job(int(cnt_t))
        print("end time: ", time.ctime())
    except (IndexError, ValueError):
        print("Usage: python3 {} <int>".format(sys.argv[0]))
        print("\tex: python3 {} 5".format(sys.argv[0]))
        sys.exit(1)
if __name__ == "__main__":
    main()
```

这个程序中假设了一个耗时的任务 `timeconsuming_job`。但是为了简单起见，我们设定这个任务运行消耗的时间在启动这个程序的命令行中指定了。
当我们运行 `$ python3 thread0.py 10` 则这个程序就会耗时 10s 结束，也就是说，这 10s 中，我们看不到任何现象，看过去就像程序卡住了(hang)一样。

我们使用多线程来优化这一点，现在有一个函数 `spin`, 调用它就会在终端中转圈圈；
`spin` 函数的基本逻辑仅此而已：
```python
class InternalSignal():
    quit = False

def spin(internalsignal):
    import itertools
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + "thinking"
        write(status); flush()
        write('\x08' * len(status))
        time.sleep(.1)
        if internalsignal.quit:
            break
    write(' ' * len(status) + '\x08' * len(status))
```
> InternalSignal 这个类定义一个简单的可变对象；其中有个 quit 属性，用于从外部控制线程退出。

于是我们可以非常容易地在开始一个耗时的操作之前，先创建一个线程:
`t = threading.Thread(...)`；

这个线程执行 `spin` 函数: 
`t = threading.Thread(target=spin, args=(sig))`
`t.start()`；
> 在程序中，可以简单地理解为线程等同于绑定的函数：
> 当线程启动时，即启动该函数；
> 当该函数结束时，也是线程退出。

然后再在主线程启动那个耗时的操作；

在主线程的耗时操作结束之后，停止原先启动的 `spin` 函数线程 
`sig.quit = True`；

综上，实现这样功能的代码很简单：
```python
sig = InternalSignal()
t = threading.Thread(target=spin, args=(sig, ))
t.start()

cnt_t = sys.argv[1]
ret = timeconsuming_job(int(cnt_t))

sig.quit = True
t.join()
```




综上代码的逻辑基本上是这样的：
```
主线程
  ||  副线程（spin）停止信号 # sig = InternalSignal() (sig.quit==False)
  \/
+----------+
| 创建新线程 |     # t = threading.Thread(target=spin, args=(sig, ))
+----------+
    ||
    \/
+----------+
| 启动该线程 |    # t.start()
+----------+
   ||└------------------ 副线程
   \/                      || 运行 spin 函数
cnt_t = sys.argv[1]        || （在终端中打印转圈）
运行 timeconsuming_job      ||
  ||                       ||
  \/                       || 线程间可以共享变量
sig.quit = True ------>    || sig 作用在副线程上
  ||                       \/
  \/                   spin 函数内部代码 -> 函数结束退出 -> 线程结束
+--------------+
| 等待副线程退出 |  # t.join()
+--------------+
  ||
  \/
执行剩下的程序代码（比如打印消息）
```

完整的代码如下：
```python
#!/usr/bin/env python3
"""filename: thread1.py
Usage: $ python3 thread1.py <int>
"""
import sys
import time
import threading


class InternalSignal():
    quit = False

def spin(internalsignal):
    [... 已在前面定义 ...]

def timeconsuming_job(cnt_time):
    time.sleep(cnt_time)
    return time.ctime()

def main():
    try:
        print("current time: ", time.ctime())
        print("="*10 + "  start  " + "="*10)
        cnt_t = sys.argv[1]

        sig = InternalSignal()
        t = threading.Thread(target=spin, args=(sig, ))
        t.start()

        ret = timeconsuming_job(int(cnt_t))

        sig.quit = True
        t.join()

        print("end time: ", ret)
    except (IndexError, ValueError):
        [...同 thread0.py ...]

[...同 thread0.py ...]
```

运行效果如下：
![thread1.py 运行效果](https://img-blog.csdnimg.cn/20190224182210611.gif)
**总结**
在 Python 中创建和运行一个线程是非常简单的。
假设你有一个 `foo` 函数想要在一个新线程中运行，同时这个函数有一些参数需求，比如： `def foo(arg1, arg2, arg3)`；那么创建一个运行 foo 函数的线程只需要这样一行而已：
`thr = threading.Thread(target=foo, args=(param1, param2, param3))`;

> 注意将 threading 模块 import 进去。

创建完成了一个线程之后，只需调用 `.start()` 就可以运行该线程，如：
`thr.start()`; 同时，因为是并发的，所以主线程依然会往下执行。

如果主线程继续往下运行依赖副线程的运行结果，那么可以使用 `.join()` 等待指定的线程结束， 如： `thr.join()`。


## 2 编写线程类
当我们确认有一个“函数”会以线程的形式执行的时候，只要这个函数稍微复杂一点点，
> 这里的“复杂”并不是指算法复杂，或者逻辑复杂。
> 往往更多时候是一个想要以线程运行的函数内部调用其它库会发生的各种情况的复杂，即使是只有一行代码，往往也会发生各种不同的现象，继续往下看。

前面 `t=Thread(target=func); t.start()` 的方式往往不能够满足需求，所以将该函数放在一个类中，辅以其它线程关联的功能就相当有必要了。

### 2.1 非继承方式
使用非继承方式编写线程类在创建线程和启动与前文的方式基本一样。不过因为到了“类级”，所以可以为该任务（线程）添加更多的控制（比如上文的退出线程功能）。

在开始之前，这里有一段简单的代码来做对比，使后文更容易理解：
![n/a](https://img-blog.csdnimg.cn/20190224195110735.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5NzU3Mjgz,size_16,color_FFFFFF,t_70#pic_center)
其中第一个方块/代码段是自定义的一个 `__FUNCTION__` 函数，用于返回当前运行的函数名，在调试的时候使用，所以不必太过在意。
第二个方块/代码段定义了一个简单的类，没有太多特殊之处。
第三个方块初始化了该类的一个实例；
第四个方块很常规的调用了该类中的 `.run` 函数。

下面，我们可以将该实例对象的函数绑定在一个新的变量上：
![n/a](https://img-blog.csdnimg.cn/20190224195613489.png)
然后看过去就像调用普通函数一样调用该类实例对象中的函数：
![n/a](https://img-blog.csdnimg.cn/20190224195909397.gif)
看到这里，很容易得出以下将类方法作为线程的代码：

```python
func_as_thr = task.run
t = Thread(target=func_as_thr, args=(2, ))
t.start()
```
但是我们当然不需要这么麻烦，实际上这么写即可：
```python
t = Thread(target=task.run, args=(2, ))
t.start()
```

#### 2.1.1 带退出功能的线程类
> 除了判断线程是否正在执行/执行完毕和等待线程结束之外，并没有其它操作可以作用在线程的操作。
> > 注：后台线程[^2]无法等待
> 
> 你无法结束一个线程，无法给它发送信号[^3]，无法调整它的调度，也无法执行其它高级操作。
> 如果需要这些特性，你需要自己添加。[^4]

现在我们就为一个非继承方式的线程类添加终止线程的功能。
要实现这个功能，线程必须通过编程在某个特定点轮询来退出：

```python
class SpinTask:
    def __init__(self):
        self._running = True
    def terminate(self):
        self._running = False
    def run(self):
        import itertools
        write, flush = sys.stdout.write, sys.stdout.flush
        for char in itertools.cycle('|/-\\'):
            status = char + ' ' + "thinking"
            write(status); flush()
            write('\x08' * len(status))
            time.sleep(.1)
            if not self._running:
                break
        write(' ' * len(status) + '\x08' * len(status))
```

只需要稍微修改一下上文的代码，就能够使用这个线程类了：
```python
# filename: thread2.py
import sys, time, threading

class SpinTask:
    [...参见上文定义...]

def timeconsuming_job(cnt_time):
    [...同 thread0.py ...]

def main():
    try:
        print("current time: ", time.ctime())
        print("="*10 + "  start  " + "="*10)

        spin = SpinTask()
        t = threading.Thread(target=spin.run, )
        t.start()

        cnt_t = sys.argv[1]
        ret = timeconsuming_job(int(cnt_t))

        spin.terminate()
        t.join()

        print("end time: ", ret)
    except (IndexError, ValueError):
        [...同 thread0.py ...]

[...同 thread0.py ...]
```

使用 `$ python3 thread2.py 3 ` 运行查看现象和上面 `thread1.py` 运行现象一致。


[^2]: 后台线程是指 `thr.setDaemon(True)` 的线程，后台线程会在主线程终止时自动销毁。所以对于“后台线程”，可以用“不重要”的线程来记住它的定义，因为无需也无法等待它的工作完成（后台线程退出），所以它是“不重要”的。

[^3]: 这里的信号和 [1 入门 Python 多线程](#1 入门 Python 多线程) 中的示例的信号不同。这里的信号指的就是系统层面的“信号”（如 SIGTREM, SIGINT, SIGALRM 等）。

[^4]: 《Python3 cookbook》CH 12: 并发编程 - 12.1 启动和停止线程。

#### 2.1.2 编写带超时功能的线程
正如前面提到的，当一个“函数”以线程的形式执行的时候，只要这个函数稍微复杂一点点，就会发生很多意外情况。
比起没有终止功能的线程（等它自动退出）更加恐怖的事情是线程阻塞，因为很多情况下，线程无法退出的原因就是永远阻塞在那里（线程无法检测自己是否已经被结束了）。
另外，当一个线程阻塞住了，即使是程序不需要线程退出的用例，但是线程间的协调 也会 hang 住，这样的线程往往会造成一系列连锁阻塞（在没有很好考虑阻塞的多线程程序中）。

下面是一个利用超时循环来小心操作线程的例子[^5]：
```python
class IOTask:
    def terminate(self):
        self._running = False
    def run(self, sock):
        # sock is a socket
        sock.settimeout(5) # Set timeout period
        while self._running:
            # Perform a blocking I/O operation w/ timeout
            try:
                data = sock.recv(8192)
                break
            except socket.timeout:
                continue
            # Continued processing
            [...]
        # Terminated
        return
```

注意，自己编写的线程类没有通用的超时方案，而是必须自己在会发生阻塞的 IO 调用上设置超时。
> 还记得前面说过不能对线程使用信号吗？
> 所以如果你想到了定时器，那么很不幸的告诉你： SIGALRM(定时器信号)不能作为线程超时通用的解决方案。[^6]

基于这个示例，我将修改上面的 `timeconsuming_job()` 函数到一个新的线程中，并且为它增加一个更加实质性的操作 - 获取网页代码，这个操作基于复杂的网络环境会产生多种不同的结果。
我将分别使用 Baidu 和 Google 的查询 API 来查询孙艺珍生日（其结果会在网页代码中），但是由于众所周知的原因，这个行为将会在使用 Google 时被阻塞住：
![n/a](https://img-blog.csdnimg.cn/20190224213759307.gif)
将这段查询及获得结果的代码写在 `timeconsuming_job` 函数中:

```python
-[o] 待续
```

## 待续


[^5]: 本例来自 《Python3 cookbook》CH 12: 并发编程 - 12.1 启动和停止线程。

[^6]: 这是为什么呢？
	> Pyhon Docs:
	> 18.8.1.2. Signals and threads
	> > Python signal handlers are always executed in the main Python thread, even if the signal was received in another thread. This means that signals can’t be used as a means of inter-thread communication. You can use the synchronization primitives from the threading module instead.
	> > Besides, only the main thread is allowed to set a new signal handler.

------

## Reference





