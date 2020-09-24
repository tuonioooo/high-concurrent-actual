# 分布式锁的应用场景

## 场景1：

清明节前，团队要求我们在 Wiki 登记自己的休假情况，假设我们在 id=1 这个文档上记录我们的休假时间和联系电话。

A、C 两个同学同时开始编辑，并且 A 和 C 在同一时间提交了结果，他们在提交前文档是空的。

服务需要如何处理这两个请求呢？以谁的为准呢？会不会产生覆盖现象导致 A 的记录丢失了？

## 场景2：

另一个 case，我是 Z 同学，在我前面别人都已经填完了，我有一个陋习，喜欢在保存的时候连续按 3-5 下 Ctrl+s，而每一个 Ctrl+s 都会触发一个请求，但是每个请求处理大概1s钟，但是实际请求都在 20ms 内发出去了。

## 参考

[http://www.sohu.com/a/121511814\_464071](http://www.sohu.com/a/121511814_464071)

[http://baijiahao.baidu.com/s?id=1598278413455081624&wfr=spider&for=pc](http://baijiahao.baidu.com/s?id=1598278413455081624&wfr=spider&for=pc)

[https://www.jianshu.com/p/2d22df6eccf8](https://www.jianshu.com/p/2d22df6eccf8)

