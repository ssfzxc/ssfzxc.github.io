# 个人觉得的协程


# 简单说说，我觉得的协程(Coroutine)

----------

> coroutine async await
> 
> Coroutine单从IO方面, 浅谈个人的看法和想法, 并非专业角度
>
> 此为杂谈, 内容有点乱, 后期有空再优化(开发者都懂的有空再说^ ^)


## 0. coroutine

> - bio 同步阻塞IO
> - nio 同步非阻塞IO
> - aio 异步IO


协程, 我最早开始知道这个概念是从Goroutine, 然后各大语言开始出现这个Feature(特性)(但是Java居然不跟进),
关键词async和await成为coroutine的标配. 如python的gevent(底层实现io的上下文切换),
asyncio以及其配套lib(aiohttp, aiomysql), javascript的promise的概念拓展.

## 1

再说协程之前, 先谈下IO, IO作为Web开发者来说是一个熟悉的名词, io模型主要有blocking io(阻塞IO)和nonblocking io(非阻塞IO)以及asynchronous io(异步IO).
还有IO multiplexing(io多路复用)等等.
Java的Tomcat7之前的版本使用的都是阻塞IO(bio), 都实现其高并发的方式是多线程, 单位时间内一个线程对应一个IO,
会导致连接数有限, 并发量上不去(生产环境通常会有负载均衡nginx/f5/..+多机部署解决问题), 而Tomcat7之后加入了非阻塞IO(nio), 其性能不会随线程增加而导致并发量的减少,
以及平均响应时间的恶化. 而Java的nio使用的模型是io多路复用.

通常使用bio进行读写io,阻塞的过程包括等待可读写状态和读写io数据流,而io多路复用将省去我们等待可读写状态的时间,通过注册消息交由系统内核去完成,如select/poll/epoll,
而epoll相对于前两者来说,优化了设置队列对象直接放于内核空间(前两者在用户空间创建队列而后拷贝至内核空间),
和返回就绪的队列而不是全部队列(全部队列需要遍历查询具体哪个为就绪).


## 表

相对来说, Python/Javascript的coroutine从实际应用来看,主要应用于io方面,
不但是aio还使代码逻辑没有断续感.

如下面的python代码,从http的请求到最后responsebody的读取,乃至于body转化为json对象都由aiohttp完成.

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

Coroutine/async/await的魅力所在,就python而言多说几句,相对于requests,其在单一请求状态下就用户态cpu占用率的减少,
开发者对这些感知是没有,再用gevent的存在,让asyncio系列lib的存在意义没有那么重大

> gevent相对于asyncio,无需关键词async和await,gevent在底层实现io阻塞时切换运行代码段,
> 简单来说,一个是自动档(gevent),一个是手动档(asyncio)
>
> 而gevent对社区乃至整个生态来说使更优解,少量的修改代码甚至无需修改代码只需在生产环境多安装几个环境即可体验到aio?的快乐,
> 其意义是重大的,asyncio需要对整个系统代码进行修改和替换一定数量的第三方依赖,这对于企业来说代价是巨大的.
> (当然引入gevent也是需要代价的)

再看看Javascript,其async/await使作为Promise的语法糖, 其本质还是callback.

## 内 

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


