[__toc__]


# mysql buffer pool
 [参考文章](https://www.modb.pro/db/111341)
 [参考文章2](https://www.cnblogs.com/better-farther-world2099/articles/14768929.html)
## 预读机制 & 传统LRU缺点
1. 预读：
    - 预读概念：操作系统的磁盘读写，并不是按需读取，而是按页读取，一次至少读一页数据（一般是4K），如果未来要读取的数据就在页中，就能够省去后续的磁盘IO，提高效率。
    - 有效性：数据局部性原理
    - 按页（4K）读区与缓冲池关系：缓冲池按页缓存数据，可能访问数据提前放入内存，提高效率
2. LRU：
    - 页已经在缓冲池里，那就只做“移至”LRU头部的动作，而没有页被淘汰。
    - 页不在缓冲池里，除了做“放入”LRU头部的动作，还要做“淘汰”LRU尾部页的动作。
    - 预读失效：由于预读（Read-Ahead），提前把页放入了缓冲池，但最终MySQL并没有从页中读取数据，称为预读失效
    - 缓冲池污染：当某一个SQL语句，要批量扫描大量数据时，可能导致把缓冲池的所有页都替换出去，导致大量热数据被换出，MySQL性能急剧下降，这种情况叫缓冲池污染。
## mysql采用的LRU（LRU-List）
### mysql的LRU机制优化思路：
   - 让缓冲池里于都失败的缓冲液尽快替换出
   - 将读取的热数据放到LRU头部——Mysql 中优化为热数据区的后 3/4 部分被访问后才将其移动到链表头部去，对于前 1/4 部分的缓存页被访问了不会进行移动
### 实现方法：
- 将LRU划分为两部分，新生代（new sublist）老生代（old sublist）5/8:3/8划分——————解决预读失效问题
- 新生代尾部连接老生代
- 新数据也加载时候加载进老生代（头部插入），如果超过1S后再次访问该数据页，在将其诺道新生代头部：1S后是因为再次访问表示该数据很大可能是热数据（只有满足“被访问”并且“在老生代停留时间”大于T，才会被放入新生代头部；）——————解决缓冲池污染问题，新生链条始终是热数据

## buffer pool中存在三种链表数据Free-List,LRU-List,Flush-List
三种链表数据对应着三种不同的缓冲页类型：
- Free-list：空闲页，未被使用过
- LRU-List：干净页，使用中，即当前内存中的缓冲页跟磁盘中的数据一致
- Flush-List：脏页，同时存在于LRU连表中，此时缓冲页的数据跟磁盘中的数据不一致，需要在适当时机触发flush行为
一旦你对内存中的缓冲页作出了修改，那该缓冲页对应的描述信息块就会添加进 Flush List。

## 刷脏页时机
脏页刷回磁盘的时机
当Buffer Pool不够用时，根据LRU机制，MySQL会将Old SubList部分的缓存页移出LRU链表。如果被移除出去的缓存页的描述信息在Flush List中，MySQL就得将其刷新回磁盘。

InnoDB存储引擎将脏页刷回磁盘的时机有蛮多的，你可以把它当作拓展知识大概浏览一下。

1、当MySQL数据库关闭时，会将所有的脏数据页刷新回磁盘。这个功能由参数：innodb_fast_shutdown=0控制，默认让InnoDB在关闭前将脏页刷回磁盘，以及清理掉undo log。

2、有一个后台线程Master Thread会按照每秒或者每十秒的速度，异步的将Buffer Pool中一定比例的页面刷新回磁盘中。

3、在MySQL5.7中，Buffer Pool的刷新由page cleaner threads完成。

我们可以通过innodb_page_cleaners参数控制page cleaner threads线程的数量，但是当你将这个数值调整的比Buffer Pool的数量还大时，MySQL会自动将 innodb_page_cleaners数量设置为innodb_buffer_pool_instances的数量。

Innodb1.1.x之前需要保证LRU列表中有至少100个空闲页可以使用。低于这个阈值就会触发脏页的刷新。

从MySQL5.6，也就是innodb1.2.X开始，innodb_lru_scan_depth参数为每个缓冲池实例指定page cleaner threads 扫描Buffer Pool来查找要刷新的脏页的下行距离。默认为1024，该后台线程每秒都会执行一次。

4、当脏数据页太多时，也会触发将脏数据页刷新回磁盘。该机制可由参数innodb_nax_dirty_pages_pct控制，比如将其设置为75，表示，当Buffer Pool中的脏数据页达到整体缓存的75%时，触发刷新的动作。现实情况是该参数默认值为0。以此来禁用Buffer Pool早期的刷新行为。

5、当redo log不可用时，也会强制脏页列表中的脏页刷新回磁盘。这个机制同样由一个后台线程完成。

