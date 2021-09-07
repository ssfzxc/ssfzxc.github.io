# 个人觉得的协程



# 简单说说，我觉得的协程 (Coroutine)

----------

> coroutine async await
> 
> Coroutine单从 IO 方面, 浅谈个人的看法和想法, 并非专业角度
>
> 此为杂谈, 内容有点乱, 后期有空再优化(开发者都懂的有空再说^ ^)


## Coroutine

> - bio 同步阻塞 IO
> - nio 同步非阻塞 IO
> - aio 异步 IO


协程, 我最早开始知道这个概念是从 Goroutine, 然后各大语言开始出现这个 Feature (特性)(但是 Java 居然不跟进),
关键词 async 和 await 成为 coroutine 的标配. 如 python 的 gevent (底层实现 io 的上下文切换),
asyncio 以及其配套 lib(aiohttp, aiomysql), javascript 的 promise 的概念拓展.


## Coroutine 协程

再说协程之前, 先谈下 IO, IO 作为 Web开发者来说是一个熟悉的名词, io 模型主要有 blocking io (阻塞IO)和 nonblocking io (非阻塞 IO )以及 asynchronous io (异步 IO ).
还有 IO multiplexing (io多路复用)等等.
Java 的 Tomcat7 之前的版本使用的都是阻塞 IO(bio), 都实现其高并发的方式是多线程, 单位时间内一个线程对应一个IO,
会导致连接数有限, 并发量上不去(生产环境通常会有负载均衡 nginx/f5/.. +多机部署解决问题), 而 Tomcat7 之后加入了非阻塞 IO(nio), 其性能不会随线程增加而导致并发量的减少,
以及平均响应时间的恶化. 而 Java 的 nio 使用的模型是 io 多路复用.

通常使用 bio 进行读写 io ,阻塞的过程包括等待可读写状态和读写 io 数据流,而 io 多路复用将省去我们等待可读写状态的时间,通过注册消息交由系统内核去完成,如 select/poll/epoll,
而 epoll 相对于前两者来说,优化了设置队列对象直接放于内核空间(前两者在用户空间创建队列而后拷贝至内核空间),
和返回就绪的队列而不是全部队列(全部队列需要遍历查询具体哪个为就绪).


## Python/JavaScript's Coroutine

相对来说, Python/Javascript 的 coroutine 从实际应用来看, 主要应用于 io 方面,
不但是 aio 还使代码逻辑没有断续感.

如下面的 python 代码,从 http 的请求到最后 responsebody 的读取,乃至于 body 转化为 json 对象都由 aiohttp 完成.

```python
# 非完整代码
import aiohttp

client = aiohttp.ClientSession()

async def post():
  async with client.post(url, request) as response:
    # await response.text()
    json = await respnse.json()
    print(json)
    return json


# asyncio.run(post())

tasks = [post for 1 in range(10)]
asyncio.run(asyncio.gather(*tasks))
```

Coroutine/async/await 的魅力所在,就 python 而言多说几句,相对于 requests , 其在单一请求状态下就用户态 cpu 占用率的减少,
开发者对这些感知是没有,再用 gevent 的存在,让 asyncio 系列 lib 的存在意义没有那么重大

> gevent 相对于 asyncio,无需关键词 async 和 await, gevent 在底层实现 io 阻塞时切换运行代码段,
> 简单来说,一个是自动档(gevent),一个是手动档(asyncio)
>
> 而 gevent 对社区乃至整个生态来说使更优解,少量的修改代码甚至无需修改代码只需在生产环境多安装几个环境即可体验到 aio? 的快乐,
> 其意义是重大的, asyncio 需要对整个系统代码进行修改和替换一定数量的第三方依赖,这对于企业来说代价是巨大的.
> (当然引入 gevent 也是需要代价的)

再看看 Javascript ,其 async/await 使作为 Promise 的语法糖, 其本质还是 callback.


## Coroutine IO

上面说到的Coroutine, 从体验上来说就可以说使aio的吧,反正个人是这么认为的.
但再细究下, 就会想到其就是nio的语法糖?,
由三方库和Interpreter实现对状态的监听以及数据的读写,完成后返回给脚本逻辑.

对于单一io处理来说,
nio是否太过于麻烦,首先从业务处理逻辑来说,其断续的代码无形的增加了我们规划和编写代码的心智负担,
(懒惰是人类发展的动力,引入async和await后代码逻辑开始连续了)
而我们该等待的时间并未缩短,
所以对于简单的单一的io处理使用coroutine并非优解,但是处理多io的高并发系统来说,这将是多份的快乐.
corouine调度器将多个coroutine处理过程集中管理,io阻塞的将被挂起.

TODO: coroutine & thread 对比


