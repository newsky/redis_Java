### 文章投票网站的redis相关Java实现
----------
### 需求: ###
1、要构建一个文章投票网站，文章需要在**一天**内至少获得200张票，才能优先显示在**当天**文章列表前列。

2、但是为了避免发布时间较久的文章由于累计的票数较多而一直停留在文章列表前列，我们需要有随着时间流逝而不断减分的评分机制。

3、于是具体的评分计算方法为:将文章得到的支持票数乘以一个常量432（由一天的秒数86400除以文章展示一天所需的支持票200得出），然后加上文章的发布时间，得出的结果就是文章的评分。

### Redis设计 ###
（1）对于网站里的每篇文章，需要使用一个**散列**来存储文章的标题、指向文章的网址、发布文章的用户、文章的发布时间、文章得到的投票数量等信息。

![记录文章内容散列][1]

为了方便网站根据文章发布的先后顺序和文章的评分高低来展示文章，我们需要两个有序集合来存储文章：
（2）**有序集合**，成员为文章ID，分值为文章的发布时间。

![文章发布时间有序集合][2]

（3）**有序集合**，成员为文章ID，分值为文章的评分。

![文章评分有序集合][3]

（4）为了防止用户对同一篇文章进行多次投票，需要为每篇文章记录一个已投票用户名单。使用**集合**来存储已投票的用户ID。由于集合是不能存储多个相同的元素的，所以不会出现同个用户对同一篇文章多次投票的情况。

![用户投票集合][4]

（5）文章支持群组功能，可以让用户只看见与特定话题相关的文章，比如“python”有关或者介绍“redis”的文章等，这时，我们需要一个集合来记录群组文章。例如 programming群组

![群组文章集合][5]

*为了节约内存，当一篇文章发布期满一周之后，用户将不能对它进行投票，文章的评分将被固定下来，而记录文章已投票用户名单的集合也会被删除。*

### 代码设计 ###
*1.当用户要发布文章时，*

（1）通过一个计数器counter执行INCR命令来创建一个新的文章ID。

（2）使用SADD将文章发布者ID添加到记录文章已投票用户名单的集合中，并用EXPIRE命令为这个集合设置一个过期时间，让Redis在文章发布期满一周后自动删除这个集合。

（3）使用HMSET命令来存储文章的相关信息，并执行两ZADD命令，将文章的初始评分和发布时间分别添加到两个相应的有序集合中。

```java
public class Chapter01 {
  private static final int ONE_WEEK_IN_SECONDS = 7 * 86400;          //文章发布期满一周后，用户不能在对它投票。
  private static final int VOTE_SCORE = 432;                        //计算评分时间与支持票数相乘的常量，通过将一天的秒数除(86400)以文章展示一天所需的支持票数(200)得出的
  private static final int ARTICLES_PER_PAGE = 25;

  /**
 * 发布并获取文章
 *1、发布一篇新文章需要创建一个新的文章id，可以通过对一个计数器(count)执行INCY命令来完成。
 *2、程序需要使用SADD将文章发布者的ID添加到记录文章已投票用户名单的集合中，
 *   并使用EXPIRE命令为这个集合设置一个过期时间，让Redis在文章发布期满一周之后自动删除这个集合。
 *3、之后程序会使用HMSET命令来存储文章的相关信息，并执行两个ZADD，将文章的初始评分和发布时间分别添加到两个相应的有序集合里面。
 */
public String postArticle(Jedis conn, String user, String title, String link) {
    //1、生成一个新的文章ID
    String articleId = String.valueOf(conn.incr("article:"));     //String.valueOf(int i) : 将 int 变量 i 转换成字符串

    String voted = "voted:" + articleId;
    //2、添加到记录文章已投用户名单中，
    conn.sadd(voted, user);
    //3、设置一周为过期时间
    conn.expire(voted, ONE_WEEK_IN_SECONDS);

    long now = System.currentTimeMillis() / 1000;
    String article = "article:" + articleId;
    //4、创建一个HashMap容器。
    HashMap<String,String> articleData = new HashMap<String,String>();
    articleData.put("title", title);
    articleData.put("link", link);
    articleData.put("user", user);
    articleData.put("now", String.valueOf(now));
    articleData.put("oppose", "0");
    articleData.put("votes", "1");
    //5、将文章信息存储到一个散列里面。
    //HMSET key field value [field value ...]
    //同时将多个 field-value (域-值)对设置到哈希表 key 中。
    //此命令会覆盖哈希表中已存在的域。
    conn.hmset(article, articleData);
    //6、将文章添加到更具评分排序的有序集合中
    //ZADD key score member [[score member] [score member] ...]
    //将一个或多个 member 元素及其 score 值加入到有序集 key 当中
    conn.zadd("score:", now + VOTE_SCORE, article);
    //7、将文章添加到更具发布时间排序的有序集合。
    conn.zadd("time:", now, article);

    return articleId;
  }
}
```

*2.当用户尝试对一篇文章进行投票时，*

（1）用ZSCORE命令检查记录文章发布时间的有序集合(redis设计2)，判断文章的发布时间是否未超过一周。

（2）如果文章仍然处于可以投票的时间范畴，那么用SADD将用户添加到记录文章已投票用户名单的集合(redis设计4)中。

（3）如果上一步操作成功，那么说明用户是第一次对这篇文章进行投票，那么使用ZINCRBY命令为文章的评分增加432(ZINCRBY命令用于对有序集合成员的分值执行自增操作)；

并使用HINCRBY命令对散列记录的文章投票数量进行更新

```java
/**
 * 对文章进行投票
 * 实现投票系统的步骤：
 * 1、当用户尝试对一篇文章进行投票时，程序要使用ZSCORE命令检查记录文字发布时间的有序集合，判断文章的发布时间是否超过一周。
 * 2、如果文章仍然处于可投票的时间范围之内，那么程序将使用SADD命令，尝试将用户添加到记录文章的已投票用户名单的集合中。
 * 3、如果投票执行成功的话，那么说明用户是第一次对这篇文章进行投票，程序将使用ZINCYBY命令为文章的评分增加432(ZINCYBY命令用于对有序集合成员的分值进行自增操作)，
 *    并使用HINCRBY命令对散列记录的文章投票数量进行更新(HINCRBY命令用于对散列存储的值执行自增操作)
 */
public void articleVote(Jedis conn, String user, String article) {
    //1、计算文章投票截止时间。
    long cutoff = (System.currentTimeMillis() / 1000) - ONE_WEEK_IN_SECONDS;
    //2、检查是否还可以对文章进行投票，(虽然使用散列也可以获取文章的发布时间，但有序集合返回文章发布时间为浮点数，可以不进行转换，直接使用)
    if (conn.zscore("time:", article) < cutoff){
        return;
    }

    //3、从articleId标识符里面取出文章ID。
    //nt indexOf(int ch,int fromIndex)函数：就是字符ch在字串fromindex位后出现的第一个位置.没有找到返加-1
    //String.Substring (Int32)	从此实例检索子字符串。子字符串从指定的字符位置开始。
    String articleId = article.substring(article.indexOf(':') + 1);
    //4、检查用户是否第一次为这篇文章投票，如果是第一次，则在增加这篇文章的投票数量和评分。
    if (conn.sadd("voted:" + articleId, user) == 1) {                       //将一个或多个 member 元素加入到集合 key 当中，已经存在于集合的 member 元素将被忽略。
        //为有序集 key 的成员 member 的 score 值加上增量 increment 。
        //可以通过传递一个负数值 increment ，让 score 减去相应的值，
        //当 key 不存在，或 member 不是 key 的成员时， ZINCRBY key increment member 等同于 ZADD key increment member 。
        //ZINCRBY salary 2000 tom   # tom 加薪啦！
        conn.zincrby("score:", VOTE_SCORE, article);

        //为哈希表 key 中的域 field 的值加上增量 increment 。
        //增量也可以为负数，相当于对给定域进行减法操作。
        //HINCRBY counter page_view 200
        conn.hincrBy(article, "votes", 1L);
    }
}

/**
 * 投反对票
 */
public void articleOppose(Jedis conn, String user, String article) {

    long cutoff = (System.currentTimeMillis() / 1000) - ONE_WEEK_IN_SECONDS;

    //cutoff之前的发布的文章 就不能再投票了
    if (conn.zscore("time:", article) < cutoff){
        return;
    }


    String articleId = article.substring(article.indexOf(':') + 1);

    //查看user是否给这篇文章投过票
    //set里面的key是唯一的 如果 sadd返回0 表示set里已经有数据了
    //如果返回1表示还没有这个数据
    if (conn.sadd("oppose:" + articleId, user) == 1) {
        conn.zincrby("score:", -VOTE_SCORE, article);
        conn.hincrBy(article, "votes", -1L);
    }
}


```

*3.我们已经实现了文章投票功能和文章发布功能，接下来就要考虑如何取出评分最高的文章以及如何取出最新发布的文章*

（1）我们需要使用ZREVRANGE命令取出多个文章ID。（由于有序集合会根据成员的分值从小到大地排列元素，使用ZREVRANGE以分值从大到小的排序取出文章ID）

（2）对每个文章ID执行一次HGETALL命令来取出文章的详细信息。

这个方法既可以用于取出评分最高的文章，又可以用于取出最新发布的文章。

```java

public List<Map<String,String>> getArticles(Jedis conn, int page) {
    //调用下面重载的方法
    return getArticles(conn, page, "score:");
}
/**
 * 取出评分最高的文章和取出最新发布的文章
 * 实现步骤：
 * 1、程序需要先使用ZREVRANGE取出多个文章ID，然后在对每个文章ID执行一次HGETALL命令来取出文章的详细信息，
 *    这个方法可以用于取出评分最高的文章，又可以用于取出最新发布的文章。
 * 需要注意的是：
 * 因为有序集合会根据成员的值从小到大排列元素，所以使用ZREVRANGE命令，已分值从大到小的排列顺序取出文章ID才是正确的做法
 *
 */

public List<Map<String,String>> getArticles(Jedis conn, int page, String order) {
    //1、设置获取文章的起始索引和结束索引。
    int start = (page - 1) * ARTICLES_PER_PAGE;
    int end = start + ARTICLES_PER_PAGE - 1;
     //2、获取多个文章ID,
    Set<String> ids = conn.zrevrange(order, start, end);
    List<Map<String,String>> articles = new ArrayList<Map<String,String>>();
    for (String id : ids){
        //3、根据文章ID获取文章的详细信息
        Map<String,String> articleData = conn.hgetAll(id);
        articleData.put("id", id);
        //4、添加到ArrayList容器中
        articles.add(articleData);
    }

    return articles;
}

```
*4. 对文章进行分组，用户可以只看自己感兴趣的相关主题的文章。*

群组功能主要有两个部分：一是负责记录文章属于哪个群组，二是负责取出群组中的文章。

为了记录各个群组都保存了哪些文章，需要为每个群组创建一个集合，并将所有同属一个群组的文章ID都记录到那个集合中。

```java
/**
 * 记录文章属于哪个群组
 * 将所属一个群组的文章ID记录到那个集合中
 * Redis不仅可以对多个集合执行操作，甚至在一些情况下，还可以在集合和有序集合之间执行操作
 */
public void addGroups(Jedis conn, String articleId, String[] toAdd) {
    //1、构建存储文章信息的键名
    String article = "article:" + articleId;
    for (String group : toAdd) {
        //2、将文章添加到它所属的群组里面
        conn.sadd("group:" + group, article);
    }
}

```
由于我们还需要根据评分或者发布时间对群组文章进行排序和分页，所以需要将同一个群组中的所有文章按照评分或者发布时间有序地存储到一个有序集合中。
但我们已经有所有文章根据评分和发布时间的有序集合，我们不需要再重新保存每个群组中相关有序集合，我们可以通过取出群组文章集合与相关有序集合的交集，就可以得到各个群组文章的评分和发布时间的有序集合。

Redis的ZINTERSTORE命令可以接受多个集合和多个有序集合作为输入，找出所有同时存在于集合和有序集合的成员，并以几种不同的方式来合并这些成员的分值（所有集合成员的分支都会视为1）。

对于文章投票网站来说，可以使用ZINTERSTORE命令选出相同成员中最大的那个分值来作为交集成员的分值：取决于所使用的排序选项，这些分值既可以是文章的评分，也可以是文章的发布时间。

如下的示例图，显示了执行ZINTERSTORE命令的过程：

![群组文章的有序集合过程][6]

对集合groups:programming和有序集合score:进行交集计算得出了新的有序集合score:programming，它包含了所有同时存在于集合groups:programming和有序集合score:的成员。因为集合groups:programming的所有成员分值都被视为1，而有序集合score:的所有成员分值都大于1，这次交集计算挑选出来的分值为相同成员中的最大分值，所以有序集合score:programming的成员分值实际上是由有序集合score:的成员的分值来决定的。

所以，我们的操作如下：

（1）通过群组文章集合和评分的有序集合或发布时间的有序集合执行ZINTERSTORE命令，而得到相关的群组文章有序集合。

（2）如果群组文章很多，那么执行ZINTERSTORE需要花费较多的时间，为了尽量减少redis的工作量，我们将查询出的有序集合进行缓存处理，尽量减少ZINTERSTORE命令的执行次数。

为了保持持续更新后我们能获取到最新的群组文章有序集合，我们只将结果缓存60秒。

（3）使用上一步的getArticles函数来分页并获取群组文章。
```java
public List<Map<String,String>> getGroupArticles(Jedis conn, String group, int page) {
    //调用下面重载的方法
    return getGroupArticles(conn, group, page, "score:");
}

/**
 * 取出群组里面的文章
 * 为了能够根据评分对群组文章进行排序和分页，网站需要将同一个群组里面的所有文章都按评分有序的存储到一个有序集合内，
 * 程序需要使用ZINTERSTORE命令选出相同成员中最大的那个分支作为交集成员的值：取决于所使用的排序选项，可以是评分或文章发布时间。
 */
public List<Map<String,String>> getGroupArticles(Jedis conn, String group, int page, String order) {
     //1、为每个群组的每种排列顺序都创建一个键。
    String key = order + group;
    //2、检查是否有已缓存的排序结果，如果没有则进行排序。
    if (!conn.exists(key)) {
        //3、根据评分或者发布时间对群组文章进行排序
        ZParams params = new ZParams().aggregate(ZParams.Aggregate.MAX);
        conn.zinterstore(key, params, "group:" + group, order);
        //让Redis在60秒之后自动删除这个有序集合
        conn.expire(key, 60);
    }
    //4、调用之前定义的getArticles()来进行分页并获取文章数据
    return getArticles(conn, page, key);
}
```
以上就是一个文章投票网站的相关redis实现。

测试代码如下：

```java
public static final void main(String[] args) {
    new Chapter01().run();
}

public void run() {
    //1、初始化redis连接
    Jedis conn = new Jedis("localhost");
    conn.select(15);

    //2、发布文章
    String articleId = postArticle(
        conn, "guoxiaoxu", "A title", "http://www.google.com");
    System.out.println("我发布了一篇文章，id为：: " + articleId);
    System.out.println("文章保存的散列格式如下：");
    Map<String,String> articleData = conn.hgetAll("article:" + articleId);
    for (Map.Entry<String,String> entry : articleData.entrySet()){
        System.out.println("  " + entry.getKey() + ": " + entry.getValue());
    }

    System.out.println();
    //2、测试文章的投票过程
    articleVote(conn, "other_user", "article:" + articleId);
    String votes = conn.hget("article:" + articleId, "votes");
    System.out.println("我们为该文章投票，目前该文章的票数 " + votes);
    assert Integer.parseInt(votes) > 1;
    //3、测试文章的投票过程
    articleOppose(conn, "other_user", "article:" + articleId);
    String oppose = conn.hget("article:" + articleId, "votes");
    System.out.println("我们为该文章投反对票，目前该文章的反对票数 " + oppose);
    assert Integer.parseInt(oppose) > 1;

    System.out.println("当前得分最高的文章是：");
    List<Map<String,String>> articles = getArticles(conn, 1);
    printArticles(articles);
    assert articles.size() >= 1;

    addGroups(conn, articleId, new String[]{"new-group"});
    System.out.println("我们将文章推到新的群组，其他文章包括：");
    articles = getGroupArticles(conn, "new-group", 1);
    printArticles(articles);
    assert articles.size() >= 1;
}
```


### 参考 ###
[1.文章投票网站的redis相关实现(python)][7]

[Redis实战相关代码，目前有Java，JS，node，Python][8]

[2.Redis 命令参考][9]

### 代码地址 ###

[https://github.com/guoxiaoxu/redis_Java][10]


### 说明 ###
如果你有耐心读到这里，请允许我说明下：
- 1、本文主题结构参考了[文章投票网站的redis相关实现(python)][7]
- 2、留下重复的注释是为了自己对比，努力让自己变得不一样
- 3、通过一天的分析、学习。越觉得需要学的东西太多了。而不只是简单的记住几个命令
- 4、感谢所有人，感谢[SegmentFault][11],让你见证我脱变的过程吧。


  [1]: https://segmentfault.com/img/bVS4cn?w=26&h=219
  [2]: https://segmentfault.com/img/bVS4de?w=26&h=219
  [3]: https://segmentfault.com/img/bVTeoa?w=266&h=147
  [4]: https://segmentfault.com/img/bVS9jg?w=263&h=149
  [5]: https://segmentfault.com/img/bVTeoA?w=269&h=150
  [6]: https://segmentfault.com/img/bVTerN?w=891&h=155
  [7]: https://segmentfault.com/a/1190000010741281
  [8]: https://github.com/guoxiaoxu/redis-in-action
  [9]: http://redisdoc.com/index.html
  [10]:https://github.com/guoxiaoxu/redis_Java
  [11]:https://segmentfault.com/u/guoxiaoxu
