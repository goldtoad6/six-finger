# 前言
>文本已收录至我的GitHub仓库，欢迎Star：https://github.com/bin392328206/six-finger                             
> **种一棵树最好的时间是十年前，其次是现在**   
>我知道很多人不玩**qq**了,但是怀旧一下,欢迎加入六脉神剑Java菜鸟学习群，群聊号码：**549684836** 鼓励大家在技术的路上写博客
## 絮叨 

我们继续来探索mysql。前面我们了解了mysql的索引的一些基础知识，今天我们来康康B+树索引
- [🔥Mysql从入门到入神之（一）Schema 数据类型优化 和索引基础](https://juejin.im/post/5e40c87b518825494905b7ac)
- [🔥Mysql从入门到入神之（二）Select 和Update的执行过程](https://mp.weixin.qq.com/s?__biz=MjM5OTA0MjE5Mg==&mid=2247483752&idx=1&sn=84a50fd9197aca938c1b6144b843f3d6&chksm=a6c0cc9791b745811fb8aa889990b66bc13acdfa3493f843bedf80b64ed8d40191961c8ac849&token=1932512762&lang=zh_CN#rd)
- [🔥Mysql从入门到入神之（三）InnoDB的存储结构](https://juejin.im/post/5e8191b4e51d4546d961e674)


> 来复习一下一下昨天的 首先是InnoDB的页存储结构，我们知道 多个不同的页组成的是一个双向链表，而每个页里面的数据行会按主键的大小组成一个单向链表，并且每4到8个数据组成一个槽，每个槽存储在pageDirectoy里面 ，当我们要查询页的行数据的时候，可以先定位到页，然后用2分法定位到槽，然后遍历槽，来定位到当前行的数据。（大佬画的图，大家可以好好理解一下）

![](https://user-gold-cdn.xitu.io/2020/4/2/17139eec9aff2635?w=1092&h=340&f=png&s=90312)
其中页a、页b、页c ... 页n 这些页可以不在物理结构上相连，只要通过双向链表相关联即可。

## 没有索引下的查找数据的方式

- 第一种，查询的是id主键的一个确定值，这个好像还不是那么难，首先遍历所有的页，定位到页，从页里面找到槽，从槽里面找到当前行，所以这样说的话，这种如果页数比较多的话，查询也会很慢
- 第二种，也就是我们说的全表扫描，一个个去遍历，最后来找到这一行数据，因为这种查询的会非常的慢，所以呢我们的索引就派上用场了


### InnoDB中的索引方案

- InnoDB是使用页来作为管理存储空间的基本单位，也就是最多能保证16KB的连续存储空间，而随着表中记录数量的增多，需要非常大的连续的存储空间才能把所有的目录项都放下，这对记录数量非常多的表是不现实的。
- 我们时常会对记录进行增删，假设我们把页中的记录都删除了，页也就没有存在的必要了，那意味着目录项也就没有存在的必要了，这就需要把目录项后的目录项都向前移动一下，这种牵一发而动全身的设计不是什么好主意～

它是怎么来实现 ，页记录 和用户用户记录 ，它每个行数据中又一个record_type 这个既可以表示，页记录 和用户用户记。它有以下的4种取值方式
- 0：普通的用户记录
- 1：目录项记录
- 2：最小记录
- 3：最大记录

![](https://user-gold-cdn.xitu.io/2020/4/2/1713a054901499f9?w=1136&h=533&f=png&s=102978)

不论是存放用户记录的数据页，还是存放目录项记录的数据页，我们都把它们存放到B+树这个数据结构中了，所以我们也称这些数据页为节点。从图中可以看出来，我们的实际用户记录其实都存放在B+树的最底层的节点上，这些节点也被称为叶子节点或叶节点，其余用来存放目录项的节点称为非叶子节点或者内节点，其中B+树最上边的那个节点也称为根节点。


## 聚簇索引
上面的B+数 本身就是一个主键索引 我们也叫聚簇索引，它有两个特点
- 使用记录主键值的大小进行记录和页的排序，这包括三个方面的含义：
    - 存放目录项记录的页分为不同的层次，在同一层次中的页也是根据页中目录项记录的主键大小顺序排成一个双向链表。 （树的每一层都是一个双向链表）
    - 各个存放用户记录的页也是根据页中用户记录的主键大小顺序排成一个双向链表。（最后一层的用户数据层也是一个双向链表）
    - 页内的记录是按照主键的大小顺序排成一个单向链表。（页内是一个单休链表，和一个有着顺序的槽目录）


- B+树的叶子节点存储的是完整的用户记录。所谓完整的用户记录，就是指这个记录中存储了所有列的值（包括隐藏列）。

我们把具有这两种特性的B+树称为聚簇索引，所有完整的用户记录都存放在这个聚簇索引的叶子节点处。这种聚簇索引并不需要我们在MySQL语句中显式的使用INDEX语句去创建（后边会介绍索引相关的语句），InnoDB存储引擎会自动的为我们创建聚簇索引。另外有趣的一点是，在InnoDB存储引擎中，聚簇索引就是数据的存储方式（所有的用户记录都存储在了叶子节点），也就是所谓的索引即数据，数据即索引。

## 二级索引
大家有木有发现，上边介绍的聚簇索引只能在搜索条件是主键值时才能发挥作用，因为B+树中的数据都是按照主键进行排序的。那如果我们想以别的列作为搜索条件该咋办呢？难道只能从头到尾沿着链表依次遍历记录么？

不，我们可以多建几棵B+树，不同的B+树中的数据采用不同的排序规则。比方说我们用c2列的大小作为数据页、页中记录的排序规则，再建一棵B+树，效果如下图所示：


![](https://user-gold-cdn.xitu.io/2020/4/2/1713a12f5b58ae53?w=1112&h=586&f=png&s=158954)
其实这个呢，和上面的也差不都就是说这个说子节点存放的是我们的索引列+我们的主键的数据。如果我们想要当前那一行的所有数据的话，我们是需要做一次回表操作的。


## 联合索引
我们也可以同时以多个列的大小作为排序规则，也就是同时为多个列建立索引，比方说我们想让B+树按照c2和c3列的大小进行排序，这个包含两层含义：

- 先把各个记录和页按照c2列进行排序。
- 在记录的c2列相同的情况下，采用c3列进行排序


![](https://user-gold-cdn.xitu.io/2020/4/2/1713a16284606dc1?w=1153&h=617&f=png&s=353890)

类似于这种，就是先把第一个列做好索引，然后再排第二个列，必须按先后顺序来，所以我们所说的前缀索引就是这样来的。

## 索引的代价
在熟悉了B+树索引原理之后，本篇文章的主题是唠叨如何更好的使用索引，虽然索引是个好东西，可不能乱建，在介绍如何更好的使用索引之前先要了解一下使用这玩意儿的代价，它在空间和时间上都会拖后腿：
- 空间上的代价
    - 这个是显而易见的，每建立一个索引都要为它建立一棵B+树，每一棵B+树的每一个节点都是一个数据页，一个页默认会占用16KB的存储空间，一棵很大的B+树由许多数据页组成，那可是很大的一片存储空间呢。

- 时间上的代价
    - 每次对表中的数据进行增、删、改操作时，都需要去修改各个B+树索引。而且我们讲过，B+树每层节点都是按照索引列的值从小到大的顺序排序而组成了双向链表。不论是叶子节点中的记录，还是内节点中的记录（也就是不论是用户记录还是目录项记录）都是按照索引列的值从小到大的顺序而形成了一个单向链表。而增、删、改操作可能会对节点和记录的排序造成破坏，所以存储引擎需要额外的时间进行一些记录移位，页面分裂、页面回收啥的操作来维护好节点和记录的排序。如果我们建了许多索引，每个索引对应的B+树都要进行相关的维护操作，这还能不给性能拖后腿么？





## 创建高性能索引原则

### 独立的列
什么意思呢？就是我们where = 后面的条件，必须是独立的一个列，不能是id+1,这种计算，所以有一个原则就是始终将索引列单独放在比较符合的一侧。

### 前缀索引

比如一个字符串很长，然后你要给这个字段建立索引，如果说他们前几个字段的识别度很高了话，就建议建立一个前缀索引。这样就可以大大的节省索引空间


### 多列索引
一个常见的错误就是，给每个列都建立一个索引，这样是错误的，还有就是建立联合索引的时候的顺序是随便填的，这种方式也是错误的。如果你用explain 关键字看到了 索引合并的信息，就说明你这个索引看是否能否优化。


### 选择合适的索引顺序

假设你有2个列要建立组合索引，那么这个组合索引的列的字段到底是哪个先 哪个后呢？这个是没有一定的标准的，但是默认的条件是如果你有数量少的字段尽量是放到前面，在不考虑，分组的条件下，这种情况确实是比较快的。

### 覆盖索引

覆盖索引的意思就是我们建立索引的时候，我把需要查询的条件一起建立一个联合索引，那么查询这些数据的时候，我们就不需要回表操作了。


### 尽量用索引扫描来排序
如果explain中type的结果是index,就说明mysql使用了索引扫描来做排序，

### 未使用的索引
如果发现有些索引是一直不会使用的索引，建议删除它。


### 不可以使用索引进行排序的几种情况
- ASC、DESC混用
对于使用联合索引进行排序的场景，我们要求各个排序列的排序顺序是一致的，也就是要么各个列都是ASC规则排序，要么都是DESC规则排序。
- WHERE子句中出现非排序使用到的索引列


## 总结
- 索引并不是想建就建，凡事都是有代价的吗，我们只能说权衡利弊
- 索引通用的一些场景
    - 等值查询
    - 匹配组合索引左边的索引
    - 匹配组合索引的左边的索引的范围查询
    - 匹配等值查询和范围查询
    - 分组查询
    - 排序
- 索引的一些注意事项
    - 只为搜索，分组，排序的列建立索引
    - 只为数据的识别度高的列建立索引（例如性别就不建议建立索引）
    - 对于字符串的列，如果它的能建立前缀索引，最好就建立前缀索引
    - 为了让页尽量减少页分裂的情况，最好给主键建立自增
    - 删除不必要的索引
    - 如果能用覆盖索引的尽量用覆盖索引，减少回表的次数。


## 结尾
我们下章继续再战。
文章部分内容出自 [MySQL 是怎样运行的：从根儿上理解 MySQL](https://juejin.im/book/5bffcbc9f265da614b11b731),
![](https://user-gold-cdn.xitu.io/2020/4/7/1715405b9c95d021?w=900&h=500&f=png&s=109836)

## 日常求赞
> 好了各位，以上就是这篇文章的全部内容了，能看到这里的人呀，都是**真粉**。

> 创作不易，各位的支持和认可，就是我创作的最大动力，我们下篇文章见

>六脉神剑 | 文 【原创】如果本篇博客有任何错误，请批评指教，不胜感激 ！
