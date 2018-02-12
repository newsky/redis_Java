### 购物网站的redis相关实现
----------
### 本文主要内容: ###
* 1、登录cookie
* 2、购物车cookie
* 3、缓存数据库行
* 4、测试
### 必备知识点 ###

WEB应用就是通过HTTP协议对网页浏览器发出的请求进行相应的服务器或者服务(Service).

一个WEB服务器对请求进行响应的典型步骤如下：

* 1、服务器对客户端发来的请求(request)进行解析.

* 2、请求被转发到一个预定义的处理器(handler）

* 3、处理器可能会从数据库中取出数据。

* 4、处理器根据取出的数据对模板(template)进行渲染(rander)

* 5、处理器向客户端返回渲染后的内容作为请求的相应。

以上展示了典型的web服务器运作方式，这种情况下的web请求是无状态的(stateless),
服务器本身不会记住与过往请求有关的任何信息，这使得失效的服务器可以很容易的替换掉。

---------

每当我们登录互联网服务的时候，这些服务都会使用cookie来记录我们的身份。

 cookies由少量数据组成，网站要求我们浏览器存储这些数据，并且在每次服务发出请求时再将这些数据传回服务。

 对于用来登录的cookie ，有两种常见的方法可以将登录信息存储在cookie里：
- 签名cookie通常会存储用户名，还有用户ID，用户最后一次登录的时间，以及网站觉得有用的其他信息。
 - 令牌cookie会在cookie里存储一串随机字节作为令牌，服务器可以根据令牌在数据库中查找令牌的拥有者。

签名cookie和令牌cookie的优点和缺点：

```table
* ------------------------------------------------------------------------------------------------
* |  cookie类型       |                  优点                    |           缺点                 |
* -------------------------------------------------------------------------------------------------
* |    签名           |  验证cookkie所需的一切信息都存储在cookie  |  正确的处理签名很难，很容易忘记  |                      |                                      |
* |   cookie          |  还可以包含额外的信息                    |  对数据签名或者忘记验证数据签名， |
* |                   |  对这些前面也很容易                      |  从而造成安全漏洞               |
* -------------------------------------------------------------------------------------------------
* |   令牌            |     添加信息非常容易，cookie体积小。      |   需要在服务器中存储更多信息，   |                    |                                          |
* |   cookie          |  移动端和较慢的客户端可以更快的发送请求    |  使用关系型数据库，载入存储代价高 |                           |                                      |
* -------------------------------------------------------------------------------------------------
```

因为该网站没有实现签名cookie的需求，所以使用令牌cookie来引用关系型数据库表中负责存储用户登录信息的条目。
除了登录信息，还可以将用户的访问时长和已浏览商品的数量等信息存储到数据库中，有利于更好的像用户推销商品

----

#### （1）登录和cookie缓存 ####

```java
/**
 * 使用Redis重新实现登录cookie，取代目前由关系型数据库实现的登录cookie功能
 * 1、将使用一个散列来存储登录cookie令牌与与登录用户之间的映射。
 * 2、需要根据给定的令牌来查找与之对应的用户，并在已经登录的情况下，返回该用户id。
 */
public String checkToken(Jedis conn, String token) {
    //1、String token = UUID.randomUUID().toString();
    //2、尝试获取并返回令牌对应的用户
    return conn.hget("login:", token);
}

```

```java
/**
 * 1、每次用户浏览页面的时候，程序需都会对用户存储在登录散列里面的信息进行更新，
 * 2、并将用户的令牌和当前时间戳添加到记录最近登录用户的集合里。
 * 3、如果用户正在浏览的是一个商品，程序还会将商品添加到记录这个用户最近浏览过的商品有序集合里面，
 * 4、如果记录商品的数量超过25个时，对这个有序集合进行修剪。
 */
public void updateToken(Jedis conn, String token, String user, String item) {
    //1、获取当前时间戳
    long timestamp = System.currentTimeMillis() / 1000;
    //2、维持令牌与已登录用户之间的映射。
    conn.hset("login:", token, user);
    //3、记录令牌最后一次出现的时间
    conn.zadd("recent:", timestamp, token);
    if (item != null) {
        //4、记录用户浏览过的商品
        conn.zadd("viewed:" + token, timestamp, item);
        //5、移除旧记录，只保留用户最近浏览过的25个商品
        conn.zremrangeByRank("viewed:" + token, 0, -26);
        //6、为有序集key的成员member的score值加上增量increment。通过传递一个负数值increment 让 score 减去相应的值，
        conn.zincrby("viewed:", -1, item);
    }
}
```


```java
/**
 *存储会话数据所需的内存会随着时间的推移而不断增加，所有我们需要定期清理旧的会话数据。
 * 1、清理会话的程序由一个循环构成，这个循环每次执行的时候，都会检查存储在最近登录令牌的有序集合的大小。
 * 2、如果有序集合的大小超过了限制，那么程序会从有序集合中移除最多100个最旧的令牌，
 * 3、并从记录用户登录信息的散列里移除被删除令牌对应的用户信息，
 * 4、并对存储了这些用户最近浏览商品记录的有序集合中进行清理。
 * 5、于此相反，如果令牌的数量没有超过限制，那么程序会先休眠一秒，之后在重新进行检查。
 */
public class CleanSessionsThread extends Thread {
    private Jedis conn;
    private int limit = 10000;
    private boolean quit ;

    public CleanSessionsThread(int limit) {
        this.conn = new Jedis("localhost");
        this.conn.select(14);
        this.limit = limit;
    }

    public void quit() {
        quit = true;
    }

    public void run() {
        while (!quit) {
            //1、找出目前已有令牌的数量。
            long size = conn.zcard("recent:");
            //2、令牌数量未超过限制，休眠1秒，并在之后重新检查
            if (size <= limit) {
                try {
                    sleep(1000);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
                continue;
            }

            long endIndex = Math.min(size - limit, 100);
            //3、获取需要移除的令牌ID
            Set<String> tokenSet = conn.zrange("recent:", 0, endIndex - 1);
            String[] tokens = tokenSet.toArray(new String[tokenSet.size()]);

            ArrayList<String> sessionKeys = new ArrayList<String>();
            for (String token : tokens) {
                //4、为那些将要被删除的令牌构建键名
                sessionKeys.add("viewed:" + token);
            }
            //5、移除最旧的令牌
            conn.del(sessionKeys.toArray(new String[sessionKeys.size()]));
            //6、移除被删除令牌对应的用户信息
            conn.hdel("login:", tokens);
            //7、移除用户最近浏览商品记录。
            conn.zrem("recent:", tokens);
        }
    }
}

```

#### （2）使用redis实现购物车 ####

```java
/**
 * 使用cookie实现购物车——就是将整个购物车都存储到cookie里面，
 * 优点：无需对数据库进行写入就可以实现购物车功能，
 * 缺点：怎是程序需要重新解析和验证cookie，确保cookie的格式正确。并且包含商品可以正常购买
 * 还有一缺点：因为浏览器每次发送请求都会连cookie一起发送，所以如果购物车的体积较大，
 * 那么请求发送和处理的速度可能降低。
 * -----------------------------------------------------------------
 * 1、每个用户的购物车都是一个散列，存储了商品ID与商品订单数量之间的映射。
 * 2、如果用户订购某件商品的数量大于0，那么程序会将这件商品的ID以及用户订购该商品的数量添加到散列里。
 * 3、如果用户购买的商品已经存在于散列里面，那么新的订单数量会覆盖已有的。
 * 4、相反，如果某用户订购某件商品数量不大于0，那么程序将从散列里移除该条目
 * 5、需要对之前的会话清理函数进行更新，让它在清理会话的同时，将旧会话对应的用户购物车也一并删除。
 */
public void addToCart(Jedis conn, String session, String item, int count) {
    if (count <= 0) {
        //1、从购物车里面移除指定的商品
        conn.hdel("cart:" + session, item);
    } else {
        //2、将指定的商品添加到购物车
        conn.hset("cart:" + session, item, String.valueOf(count));
    }
}

```

5、需要对之前的会话清理函数进行更新，让它在清理会话的同时，将旧会话对应的用户购物车也一并删除。

只是比CleanSessionsThread多了一行代码，伪代码如下：

```java
long endIndex = Math.min(size - limit, 100);
//3、获取需要移除的令牌ID
Set<String> tokenSet = conn.zrange("recent:", 0, endIndex - 1);
String[] tokens = tokenSet.toArray(new String[tokenSet.size()]);

ArrayList<String> sessionKeys = new ArrayList<String>();
for (String token : tokens) {
    //4、为那些将要被删除的令牌构建键名
    sessionKeys.add("viewed:" + token);

    //新增加的这两行代码用于删除旧会话对应的购物车。
    sessionKeys.add("cart:" + sess);
}
//5、移除最旧的令牌
conn.del(sessionKeys.toArray(new String[sessionKeys.size()]));
//6、移除被删除令牌对应的用户信息
conn.hdel("login:", tokens);
//7、移除用户最近浏览商品记录。
conn.zrem("recent:", tokens);
```

#### (3) 数据行缓存 ####

```java
/**
 * 为了应对促销活动带来的大量负载，需要对数据行进行缓存，具体做法是：
 * 1、编写一个持续运行的守护进程，让这个函数指定的数据行缓存到redis里面，并不定期的更新。
 * 2、缓存函数会将数据行编码为JSON字典并存储在Redis字典里。其中数据列的名字会被映射为JSON的字典，
 * 而数据行的值则被映射为JSON字典的值。
 * -----------------------------------------------------------------------------------------
 * 程序使用两个有序集合来记录应该在何时对缓存进行更新：
 * 1、第一个为调用有序集合，他的成员为数据行的ID，而分支则是一个时间戳，
 * 这个时间戳记录了应该在何时将指定的数据行缓存到Redis里面
 * 2、第二个有序集合为延时有序集合，他的成员也是数据行的ID，
 * 而分值则记录了指定数据行的缓存需要每隔多少秒更新一次。
 * ----------------------------------------------------------------------------------------------
 * 为了让缓存函数定期的缓存数据行，程序首先需要将hangID和给定的延迟值添加到延迟有序集合里面，
 * 然后再将行ID和当前指定的时间戳添加到调度有序集合里面。
 */
public void scheduleRowCache(Jedis conn, String rowId, int delay) {
    //1、先设置数据行的延迟值
    conn.zadd("delay:", delay, rowId);
    //2、立即对需要行村的数据进行调度
    conn.zadd("schedule:", System.currentTimeMillis() / 1000, rowId);
}
```

```java

/**
 * 1、通过组合使用调度函数和持续运行缓存函数，实现类一种重读进行调度的自动缓存机制，
 * 并且可以随心所欲的控制数据行缓存的更新频率：
 * 2、如果数据行记录的是特价促销商品的剩余数量，并且参与促销活动的用户特别多的话，那么最好每隔几秒更新一次数据行缓存：
 * 另一方面，如果数据并不经常改变，或者商品缺货是可以接受的，那么可以每隔几分钟更新一次缓存。
 */
public class CacheRowsThread
        extends Thread {
    private Jedis conn;
    private boolean quit;

    public CacheRowsThread() {
        this.conn = new Jedis("localhost");
        this.conn.select(14);
    }

    public void quit() {
        quit = true;
    }

    public void run() {
        Gson gson = new Gson();
        while (!quit) {
            //1、尝试获取下一个需要被缓存的数据行以及该行的调度时间戳，返回一个包含0个或一个元组列表
            Set<Tuple> range = conn.zrangeWithScores("schedule:", 0, 0);
            Tuple next = range.size() > 0 ? range.iterator().next() : null;
            long now = System.currentTimeMillis() / 1000;
            //2、暂时没有行需要被缓存，休眠50毫秒。
            if (next == null || next.getScore() > now) {
                try {
                    sleep(50);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
                continue;
            }
            //3、提前获取下一次调度的延迟时间，
            String rowId = next.getElement();
            double delay = conn.zscore("delay:", rowId);
            if (delay <= 0) {
                //4、不必在缓存这个行，将它从缓存中移除
                conn.zrem("delay:", rowId);
                conn.zrem("schedule:", rowId);
                conn.del("inv:" + rowId);
                continue;
            }
            //5、继续读取数据行
            Inventory row = Inventory.get(rowId);
            //6、更新调度时间，并设置缓存值。
            conn.zadd("schedule:", now + delay, rowId);
            conn.set("inv:" + rowId, gson.toJson(row));
        }
    }
}
```
###（4）测试 ###

PS:需要好好补偿英语了！！需要全部的可以到这里下载[官方翻译Java版][1]

```java
public class Chapter02 {
    public static final void main(String[] args)
            throws InterruptedException {
            new Chapter02().run();

    }

    public void run()
            throws InterruptedException {
        Jedis conn = new Jedis("localhost");
        conn.select(14);

        testLoginCookies(conn);
        testShopppingCartCookies(conn);
        testCacheRows(conn);
        testCacheRequest(conn);
    }

    public void testLoginCookies(Jedis conn)
            throws InterruptedException {
        System.out.println("\n----- testLoginCookies -----");
        String token = UUID.randomUUID().toString();

        updateToken(conn, token, "username", "itemX");
        System.out.println("We just logged-in/updated token: " + token);
        System.out.println("For user: 'username'");
        System.out.println();

        System.out.println("What username do we get when we look-up that token?");
        String r = checkToken(conn, token);
        System.out.println(r);
        System.out.println();
        assert r != null;

        System.out.println("Let's drop the maximum number of cookies to 0 to clean them out");
        System.out.println("We will start a thread to do the cleaning, while we stop it later");

        CleanSessionsThread thread = new CleanSessionsThread(0);
        thread.start();
        Thread.sleep(1000);
        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()) {
            throw new RuntimeException("The clean sessions thread is still alive?!?");
        }

        long s = conn.hlen("login:");
        System.out.println("The current number of sessions still available is: " + s);
        assert s == 0;
    }

    public void testShopppingCartCookies(Jedis conn)
            throws InterruptedException {
        System.out.println("\n----- testShopppingCartCookies -----");
        String token = UUID.randomUUID().toString();

        System.out.println("We'll refresh our session...");
        updateToken(conn, token, "username", "itemX");
        System.out.println("And add an item to the shopping cart");
        addToCart(conn, token, "itemY", 3);
        Map<String, String> r = conn.hgetAll("cart:" + token);
        System.out.println("Our shopping cart currently has:");
        for (Map.Entry<String, String> entry : r.entrySet()) {
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }
        System.out.println();

        assert r.size() >= 1;

        System.out.println("Let's clean out our sessions and carts");
        CleanFullSessionsThread thread = new CleanFullSessionsThread(0);
        thread.start();
        Thread.sleep(1000);
        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()) {
            throw new RuntimeException("The clean sessions thread is still alive?!?");
        }

        r = conn.hgetAll("cart:" + token);
        System.out.println("Our shopping cart now contains:");
        for (Map.Entry<String, String> entry : r.entrySet()) {
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }
        assert r.size() == 0;
    }

    public void testCacheRows(Jedis conn)
            throws InterruptedException {
        System.out.println("\n----- testCacheRows -----");
        System.out.println("First, let's schedule caching of itemX every 5 seconds");
        scheduleRowCache(conn, "itemX", 5);
        System.out.println("Our schedule looks like:");
        Set<Tuple> s = conn.zrangeWithScores("schedule:", 0, -1);
        for (Tuple tuple : s) {
            System.out.println("  " + tuple.getElement() + ", " + tuple.getScore());
        }
        assert s.size() != 0;

        System.out.println("We'll start a caching thread that will cache the data...");

        CacheRowsThread thread = new CacheRowsThread();
        thread.start();

        Thread.sleep(1000);
        System.out.println("Our cached data looks like:");
        String r = conn.get("inv:itemX");
        System.out.println(r);
        assert r != null;
        System.out.println();

        System.out.println("We'll check again in 5 seconds...");
        Thread.sleep(5000);
        System.out.println("Notice that the data has changed...");
        String r2 = conn.get("inv:itemX");
        System.out.println(r2);
        System.out.println();
        assert r2 != null;
        assert !r.equals(r2);

        System.out.println("Let's force un-caching");
        scheduleRowCache(conn, "itemX", -1);
        Thread.sleep(1000);
        r = conn.get("inv:itemX");
        System.out.println("The cache was cleared? " + (r == null));
        assert r == null;

        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()) {
            throw new RuntimeException("The database caching thread is still alive?!?");
        }
    }


}
```


### 参考 ###

[Redis实战][0]

[Redis实战相关代码，目前有Java，JS，node，Python][1]

[2.Redis 命令参考][2]

### 代码地址 ###

[https://github.com/guoxiaoxu/redis_Java][10]


### 后记 ###
如果你有耐心读到这里，请允许我说明下：
- 1、因为技术能力有限，没有梳理清另外两小节，待我在琢磨琢磨。后续补上。
- 2、看老外写的书像看故事一样，越看越精彩。不知道你们有这种感觉么？
- 3、越学越发现自己需要补充的知识太多了，给我力量吧，欢迎点赞。
- 4、感谢所有人，感谢[SegmentFault][4],让你见证我脱变的过程吧。

  [0]:https://www.amazon.cn/dp/B016YLS2LM
  [1]: https://github.com/guoxiaoxu/redis-in-action
  [2]: http://redisdoc.com/index.html
  [3]:https://github.com/guoxiaoxu/redis_Java
  [4]:https://segmentfault.com/u/guoxiaoxu



























































































-
