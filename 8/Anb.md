## Anb系统设计

### RSS
RSS reader 设计database schema, 具体到怎么index，follow up: 需要纪录post read/unread; 需要记录feed read/unread; 怎么scale..本文原创自1point3acres论坛
我的答案仅供参考：
user table: {user_id, user_name, user_info etc}. visit 1point3acres for more.
posts table: {post_id, feed_id, publish_time, post_title, post_content}
feeds table: {feed_id, feed_url, create_time}
subscription table: {subcription_id, user_id, feed_id, last_read_time} last_read_time 用于纪录feed read/unread,我一开始加了一个新的table，经面试官提示合并到这个table里面。
read status table: {post_id, user_id} 只有看过的post会放到这个table里面


设计：订阅系统需要考虑如果停止订阅要从当前Feed中删除对应条目先设计存储schema，不考虑具体数据库类型假设单机，然后问如何加速查询如何对每个table进行index，需不需要建多个，是否需要复合。然后就是典型的NewsFeed套路了。特别问到了如果某个发布者有海量订阅者怎么办。

这轮主要关注数据库的优化和系统的规模拓展性

Google reader design. 问了如何提高database读取的效率，详细问了denormalization, 以及瓶颈在哪，怎么解决。
这个人明显对index 很感兴趣。顺着他意思说就好，有多少说多少。他很感兴趣。还引进他所遇到的问题，甚至都没有说push 或者pull model。

Design news feed system like Google Reader，貌似是面经，就是用户订阅了好几个news provider比如人民日报、新浪财经，然后用户登录后需要把他订阅的provider的最新news按照时间逆序显示出来。这一轮问的比较细，从最开始的data table schema design，到怎么实现top k算法把news merge；之后问了一个小的feature就是mark news as read，需不需要再加一个table；最后问了一下master slave，类似于怎么sharding

### 翻译系统
翻译系统，我看了所有气床的面经包括他公司blog上关于翻译系统的设计，感觉都没讲很清楚，所以虽然准备了但是感觉还是有点迷糊，当然我也没在气床工作过不具体了解他们的翻译系统，只是刚好我曾经做过app的localization（就是翻译各种语言给app store这样不同国家的人下载的版本就是那个语言的版本），所以面试的时候我是结合自己工作经验做的介绍，以及一些如何做网站翻译的扩展设计（气床的产品主要还是网站，当然现在也有手机端app，手机端的方法跟我做过的应该是一样的），以下只是我个人的理解，大部分也得到面试官的认可所以应该比较接近他们的系统。面试之后还跟他聊了一些他们翻译系统平时的用法和我给的方法的差别（下面会说）

翻译系统大体分为三个部分: 1. UI (front end)， 2. translation service (middle layer), 3. translator portal，这三部分在一起最后会做出完整的version (web/app)发给client，要么是给app store的app package，要么是转换成网页发给用户（或者CDN for scalability reason).

#### UI部分
front-end developer和designer通常不用去理会如何翻译，他们主要做的是修改ui，这个过程中可能会增加，修改，删除一些文字（比如一个按键的label，比如增加新的网页和目录的标题etc)，他们的任务是确定网页和app这些UI用主语言（比如英语）是没问题的，通常这些修改之后，会提交给translation service...提交的东西你可以理解成是一整套resource文件，里面包含所有的ui相关的文字资料，标注了这是哪个Version，有哪些ui (key-value, where key represent ui variable name like "agreeButtonText", value represents actual text in English "Agree"），通常每个修改还会带一条介绍帮助翻译更好的理解语境，比如叫description里面会写这句话是当用户同意某个条款的时候点击同意的，这样翻译的人就会翻译成（同意，认同之类的），有些语言会有多义词所以这条解释很重要

#### translation service
当文字修改发给这个service，他负责三个部分，一个从resource文件中找出新版本中被改变的文字信息，比如有几行条款文字改了，或者一些button的文字改了，第二部分是把这些信息汇总（tokenlize)然后发给 Translator portal，可以理解成一个接受各种key-value pair的工具或者翻译平台，实际翻译发生在那里， 第三部分是当信息被翻译之后，汇总并声称新的resource文件给每个需要的语言，比如一开始只有en-us.resource，翻译之后这个Service会生成zh-cn.resource (中文），es-es.resource （西班牙语）。汇总后会连同这些语言一起发布这个版本的网页或者app (之后就是发布端了这里就不多讲）

#### translator portal
每个公司做多语言产品都必然会雇佣一些翻译来翻译它的产品，所以会有一个翻译平台，翻译的人不需要真实了解或者使用这个产品，他们需要的只是拿到一些词汇和语境介绍，然后给出相关语言的翻译，当translation service给了一系列需要翻译的词汇之后，这个平台会收录整理排序这些词汇（有些词汇可能是一样的，有些词汇可能更重要更急需比如一些legal文件更新），然后翻译的人从这个平台拿词，根据介绍翻译好然后发回去，通常可能会有一些PM或者designer会尝试把翻译好的版本自己测试一遍确定读起来通顺没问题，这要看公司的要求和产品重要性了。

上面这三部分基本构成了一套翻译系统，其实没太多系统设计的概念，关于手机端，最后会生成一个主的package 已经所有语言的Resource file发给store然后store负责转换语言生成新的package给不同语言的用户。网络段会有区别，可以理解成这个网页会根据不同的域名比如airbnb.jp跟Server索取日语版的网页，这时候server会有自己的一套生成系统自动提取翻译好的日语词汇安在对应的ui上 (eg.一个button叫agreButton, text = "$agreeButtonText"，他会自己找jp resource里agreeButtonText对应的值，日语的“同意”），生成网页后发给Client；Scale的考虑可以有cache这些网页到CDN, browser cache or even server cache，来缓解read-load...这里就用基本系统设计概念扩展就好了。.

面经上的翻译系统。主要问的是整体的System Design, 我的解法是先估算出所有翻译的容量，估算出来才几百兆，所以都可以存到Cache里面。然后Frontend server每个instance用memcache, Backend Server用shared Redis，Frontend 如果cache里找不到再call backend。面试官问的最tricky的是如果翻译更新了，如何迅速更新cache，我说用pubsub先broadcast给Backend，Backend更新好了Redis再broadcast给Frontend。面试官感觉对我的design非常满意，半个小时不到就结束了，然后开始愉快的吹水。. F

设计是地里的经典老题做翻译系统，要求之前地里同学也提过，就是有三个人，前端工程师，翻译官和用户的体验都要做好，他用了很久来解释这个题。这题我也是做了很久的准备，包括他们的blog和自己公司的经验。但这个题目和面试官完全无法交流的感觉。可能是因为我没有前端经验，他上来就问在这个html里怎么加一个service call然后说不用管syntax，我想那这不就一行code call后端API不就行了么，然后给了点参数，他说要包含可翻译的和不可翻译的部分，那我就加上去了，然后就在API应该怎么写这讨论了五分钟，不是说系统怎么design，就是这一行code要怎么写。有一种和小学生讨论的错觉，他问了很多 “这还用说” 这种类型的问题。做过design的人都知道，API设计是一个整体，前中后都要考虑，光说前端应该要什么怎么能design好呢？面试官草草说赶快到下一个环节。下一个环节他说你设计一下schema好了。说有一个网站是显示需要翻译的东西的，问你怎么设计table。那我就说他需要什么我就给什么就好了，直接一个表里需要的column都加上，加了index和composit primary key。他也没问别的。最后就说怎么能让user看翻译的东西更快，假设network很慢怎么办。这个时候已经没什么时间了，我也没考虑太多，给webserver后面加了个cache，然后让internal network可以去更新这个cache然后external user只看cache不看别的。现在想想都不知道应该怎么办，当时说network再慢也要make call吧，我们可以在浏览器cache一下翻译的东西能避免新的call，其他的怎么样都要call 一次吧。他还是没说什么，然后就结束了。

面试官问的比较细，web server怎么分工，数据库schema怎么设计，memcache放哪儿，是怎么工作的，要不要设置TTL，网站响应时间大概是什么量级，等等。
让design他们家的搜索页，就是user给一些filter，显示filter后的listings，然后每个listing还可以再点进去看listing的细节。没太搞懂到底想问什么，扯了些db schema design，sharding， denormalization。。。后来提示问用过elastic search没，确实没用过，可能是想问index table? 挂点二
感觉像是inverted index

### KV Store
先设计单机版本，内存快速存取且排序（说的跳表或treeset好像不太靠谱），内存满了dump到硬盘，dump的时候创建新表不block新进来的请求，内存存储对文件的index快速定位需要搜索的文件（忘了说boom filter了……），硬盘二分搜索，定期合并避免文件太多，更改删除元素直接append新元素，查询时先内存文件再倒时序查，多次更改删除后浪费空间-同一个key出现的文件数达到某一阈值后在下一次的非高峰时段进行文件合并每个key保留最近值。拓展到多机器，数据迁移一致性哈希，一致性哈希的查询表可以缓存在client端减轻master压力，上传文件时需要返回client完成信息-选择captain上传一由captain复制备份有一个备份传输完成则返回成功信息避免client等太久，保存不同replica的访问顺序避免复制完全结束前收到请求。




