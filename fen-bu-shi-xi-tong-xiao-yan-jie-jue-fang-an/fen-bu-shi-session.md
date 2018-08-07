# 分布式Session

## 方案一（最佳）

可以把用户的Session放在缓存服务器中   最好的解决方案是:放在缓存服务器中推荐二种缓存服务器:Memcached/redis

要求:Memcached/redis 必须是集群

## 方案二（下下策）

可以把用户的Session放在Cookie中

优点:解决了Session没有的问题

缺点:Session放在了用户的浏览器中,是不安全的

## 方案三（性能）

可以把用户的Session放在数据库中

优点:解决了Session没有的问题

 缺点:网站是一个成千上万用户的网站,如果把Session放到数据库中,会造成数据库压力太大,从而使的网站不能正常运转



