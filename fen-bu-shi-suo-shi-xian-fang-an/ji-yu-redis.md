# 基于redis（一）

**如何用Redis实现分布式锁？**

Redis分布式锁的基本流程并不难理解，但要想写得尽善尽美，也并不是那么容易。在这里，我们需要先了解分布式锁实现的三个核心要素：

**1.加锁**

最简单的方法是使用setnx命令。key是锁的唯一标识，按业务来决定命名。比如想要给一种商品的秒杀活动加锁，可以给key命名为 “lock\_sale\_商品ID” 。而value设置成什么呢？我们可以姑且设置成1。加锁的伪代码如下：

setnx（key，1）

当一个线程执行setnx返回1，说明key原本不存在，该线程成功得到了锁；当一个线程执行setnx返回0，说明key已经存在，该线程抢锁失败。

**2.解锁**

有加锁就得有解锁。当得到锁的线程执行完任务，需要释放锁，以便其他线程可以进入。释放锁的最简单方式是执行del指令，伪代码如下：

del（key）

释放锁之后，其他线程就可以继续执行setnx命令来获得锁。

**3.锁超时**

锁超时是什么意思呢？如果一个得到锁的线程在执行任务的过程中挂掉，来不及显式地释放锁，这块资源将会永远被锁住，别的线程再也别想进来。

所以，setnx的key必须设置一个超时时间，以保证即使没有被显式释放，这把锁也要在一定时间后自动释放。setnx不支持超时参数，所以需要额外的指令，伪代码如下：

expire（key， 30）

综合起来，我们分布式锁实现的第一版伪代码如下：

if（setnx（key，1） == 1）{

expire（key，30）

try {

do something ……

} finally {

del（key）

}

}

**4.但是上面的伪代码中，存在着三个致命问题：**

**1.setnx和expire的非原子性**

设想一个极端场景，当某线程执行setnx，成功得到了锁：

[![](https://img.colabug.com/2018/06/ea96cc3863c45540579a0430a40082b3.png)](https://img.colabug.com/2018/06/ea96cc3863c45540579a0430a40082b3.png)

setnx刚执行成功，还未来得及执行expire指令，节点1 Duang的一声挂掉了。

[![](https://img.colabug.com/2018/06/8f38f4e788248bd70f8d21b11a2142b4.png)](https://img.colabug.com/2018/06/8f38f4e788248bd70f8d21b11a2142b4.png)

这样一来，这把锁就没有设置过期时间，变得“长生不老”，别的线程再也无法获得锁了。

怎么解决呢？setnx指令本身是不支持传入超时时间的，幸好Redis 2.6.12以上版本为**set**指令增加了可选参数，伪代码如下：

**set（key，1，30，NX）**

这样就可以取代setnx指令。

**2.del 导致误删**

又是一个极端场景，假如某线程成功得到了锁，并且设置的超时时间是30秒。

[![](https://img.colabug.com/2018/06/4c90e911220879a497ec27ca47e611d2.png)](https://img.colabug.com/2018/06/4c90e911220879a497ec27ca47e611d2.png)

如果某些原因导致线程A执行的很慢很慢，过了30秒都没执行完，这时候锁过期自动释放，线程B得到了锁。

[![](https://img.colabug.com/2018/06/2c4ea400fefd4fb38b4d3ad43625ed3d.png)](https://img.colabug.com/2018/06/2c4ea400fefd4fb38b4d3ad43625ed3d.png)

随后，线程A执行完了任务，线程A接着执行del指令来释放锁。但这时候线程B还没执行完，**线程A实际上删除的是线程B加的锁**。

[![](https://img.colabug.com/2018/06/3b81a7d9af72d99c201c5c8516de0dc3.png)](https://img.colabug.com/2018/06/3b81a7d9af72d99c201c5c8516de0dc3.png)

怎么避免这种情况呢？可以在del释放锁之前做一个判断，验证当前的锁是不是自己加的锁。

至于具体的实现，可以在加锁的时候把当前的线程ID当做value，并在删除之前验证key对应的value是不是自己线程的ID。

加锁：

**String threadId = Thread.currentThread\(\).getId\(\)**

set（key，threadId ，30，NX）

解锁：

if（threadId .equals\(redisClient.get\(key\)\)）{

**del\(key\)**    


**}**

但是，这样做又隐含了一个新的问题，**判断和释放锁是两个独立操作，不是原子性**。

我们都是追求极致的程序员，所以这一块要用Lua脚本来实现：

String luaScript = “if redis.call\(‘get’, KEYS\[1\]\) == ARGV\[1\] then return redis.call\(‘del’, KEYS\[1\]\) else return 0 end”;

**redisClient**.eval\(**luaScript**, Collections.singletonList\(key\), Collections.singletonList\(threadId\)\);

这样一来，验证和删除过程就是原子操作了。

**3.出现并发的可能性**

还是刚才第二点所描述的场景，虽然我们避免了线程A误删掉key的情况，但是同一时间有A，B两个线程在访问代码块，仍然是不完美的。

怎么办呢？我们可以让获得锁的线程开启一个**守护线程**，用来给快要过期的锁“续航”。

[![](https://img.colabug.com/2018/06/b12b9ded2ff5b8e14b38a9f6e8572edf.png)](https://img.colabug.com/2018/06/b12b9ded2ff5b8e14b38a9f6e8572edf.png)

当过去了29秒，线程A还没执行完，这时候守护线程会执行expire指令，为这把锁“续命”20秒。守护线程从第29秒开始执行，每20秒执行一次。

[![](https://img.colabug.com/2018/06/c684b30d3ca455fa1b703dd274f46d9d.png)](https://img.colabug.com/2018/06/c684b30d3ca455fa1b703dd274f46d9d.png)

当线程A执行完任务，会显式关掉守护线程。

[![](https://img.colabug.com/2018/06/9628d5b5d412e1c3926a165d80c50a0f.png)](https://img.colabug.com/2018/06/9628d5b5d412e1c3926a165d80c50a0f.png)

另一种情况，如果节点1 忽然断电，由于线程A和守护线程在同一个进程，守护线程也会停下。这把锁到了超时的时候，没人给它续命，也就自动释放了。

[![](https://img.colabug.com/2018/06/09fd395fa85067d8d50a9797d6d6ebc7.png)](https://img.colabug.com/2018/06/09fd395fa85067d8d50a9797d6d6ebc7.png)

守护线程的代码并不难实现，有了大体思路，大家可以自己尝试实现一下。

