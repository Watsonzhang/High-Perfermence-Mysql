### 第一章 Mysql架构与历史
    1.1 mysql 逻辑架构
    1.2 并发控制
    1.3 事务
        1.3.1 隔离级别
        1.3.2 死锁
        1.3.3 事务日志
            使用事务日志，存储引擎在修改表数据的时候只需要修改其内存拷贝，再把修改行为持久到硬盘上事务日志中，
            而不用每次都将修改数据本身持久化到磁盘。由于事务日志是采用追加的方式，因此写日志的操作时磁盘上一
            小块区域的顺序IO。事务日志持久后，内存中被修改的数据在后台中可以慢慢刷回到磁盘。也就是说修改数据需要
            写两次磁盘。
        1.3.4 mysql中事务    
            自动提交 autocommit
            如果不显示开启一个事务，则每个查询都被当做一个事务执行提交操作。
            set session transaction isolation level committed  
            在事务中混合使用存储引擎
            隐式和显示锁定
                Innodb会根据隔离级别在需要的收货自动加锁
                所有锁都是在同一时刻被释放
                select lock in share mode 共享锁定
                select for update
                mysql也支持lock tables 和unlock tables 但是是在服务层实现，与存储引擎无关
                但是不能代替事务处理 如果引用需要用到事务 还是应该选择事务型存储引擎
        1.4 多版本并发控制
            mysql大多数事务型存储引擎都不是简单的行级别锁 基于性能的考虑都是实现了多版本并发控制mvcc 
            可以认为mvcc是行级锁的变种 很多情况下避免了加锁
            mvcc的实现 是通过保存某个时间点的快照来实现的
             比如说可重复读这个隔离级别：
                不管该事务执行多久，事务内看到的数据都是一致的。
                也就是说根据事务开始的时间不同 每个事务对同一张表 ，同一时刻看到的数据可能是不一样的
            例如innodb的MVCC
             每行记录后面保存的两个隐藏的列来实现。
             一个列保存了行的创建时间
             一个时保存行的过期时间  
             优点：保存这两个额外的系统版本号 大读书读取操作都可以不用加锁
             缺点：每行记录都需要额外的存储空间 需要做更多的行检查工作
             注：mvcc只在repeatable read read committed 两个隔离级别下工作
        1.5 mysql的存储引擎     
            1.5.1 Innodb存储引擎
            用来处理大量的短期事务  
            1.5.2 Myisam存储引擎
             不支持事务和行级锁
             表锁问题 所有查询长期处于locked状态
            1.5.3 其他存储引擎
        1.6 mysql的时间线
        1.7 mysql的开发模式
        1.8 总结     
### 第三章 服务性能剖析

### 第四章 Schema与数据类型优化
    4.1 选择优化的数据类型
        更小的通常更好
        简单就好
        尽量避免null
        4.1.1 整数类型
            tinyint samllint mediumint int bigint
        4.1.2 实数类型
            Decimal
        4.1.3 字符串类型
            varchar 变长 需要1到2个字节额外存储字符串的长度 当update时变的比原来更长 则需要分裂页放进页里(会保留空格)
            char 定长 mysql总是根据定义的字符串分配足够的空间 会删除所有的末尾空格 char值根据需要填充
            Blob和Text类型 当Blox和text值太大时 InnoDB会使用专门的“外部存储区域来进行存储”
        4.1.4 日期和时间类型
            datetime 1001年到9999年精度为秒 与时区无关 8个字节
            timestamp 保存了从1970年1月1日午夜以来的秒数 4个字节 1970-2038 依赖时区
        4.1.5 位数据类型
        4.1.6 选择标识符
        4.1.7 特殊类型数据 
    4.2 mysql schema设计中的陷阱
        太多的列
        太多的关联
        全能的枚举
        变相的枚举
    4.3 范式和反范式
        4.3.1 范式的优点与缺点
        4.3.2 反范式的优点与缺点
        4.3.3混用范式化与反范式化
    4.4 缓存表和汇总表
        4.4.1 物化视图
        4.4.2 计数器表
    4.5 加快alter table的操作的速度
        通过都是创建新表 然后插入数据 然后重命名
        modify表转换成alter column
        例子：alter table sakila.film alter column rental_duration set default 5;
        这个语句只修改.frm结构 不涉及数据 所以操作非常快
        4.5.1 只修改.frm文件
        4.5.2 快速创建myisam索引
    4.6 总结
        尽量避免过度设计
        使用小而简单的合适数据类型
        尽量使用相同数据类型存储类似的值
        注意可变长字符串
        尽量使用整型定义标识列
        小心使用eum和set



### 第五章 创建高性能索引
    索引时存储引擎快速涨到记录的一种数据结构.
    5.1 索引基础
        5.1.1 索引的类型
            B-Tree索引(B+Tree：每一个叶子节点都包含指向下一个叶子节点的指针)
                加快访问数据的速度 因为存储引擎不需要全表扫描来获取需要的数据。
                叶子节点指针指向被索引的数据 而不是其他的节点页
                索引有效的情况：
                    全职匹配
                    匹配最左前缀
                    匹配列前缀
                    匹配范围值
                    精确匹配某一列并范围匹配另外一列
                    只访问索引的查询
                索引限制的情况
                    如果不是最左列
                    不能跳过索引的列
                    如果查询某个列的范围
                哈希索引
                    innodb的哈希自适应索引：
                        当前innodb注意到某些索引新使用非常频繁时 它会在内存中基于B-tree索引之上再创建哈希索引，隐式的
                空间数据索引
                全文索引
    5.2 索引的优点
        1 索引大大减少看服务器u需要扫描的数据量
        2 索引可以帮助服务器避免排序和临时表
        3 索引将随机IO编程顺序IO
    5.3 高性能的索引策略
        (三星系统：索引将相关记录放到一起则获得一星  如果索引中数据顺序和查找中的排列顺序一致则获得二星 如果缩影中的列包含了查询汇总需要的全部列则获得“三星”)
        5.3.1 独立的列   索引列不能时表达式的一部分
        5.3.2 前缀索引和索引选择性 
        5.3.3 多列索引
        5.3.4 选择合适的索引列顺序
            选择缩影顺序的经验法则： 将选择性最高的列放到索引最前列
        5.3.5 聚簇索引
            聚簇索引不是一种单独的索引类型 而是一种数据存储方式 具体的细节依赖与其实现方式，innodb聚簇索引实际上在同一个结构中保存了B-tree索引和数据行
            两次Btree查找
            Innodb的二级索引的叶子节点存储的不是行指针而是主键值
        5.3.6 覆盖索引
            索引的叶子节点已经包含要查询的数据
            由于二级索引包含了主键的值 可以利用这一特性
        5.3.7
            使用索引扫描来做排序
            mysql有两种方式生成有序的结果： ①通过排序操作 ②按照索引扫描 explain type=index 说明使用了索引扫描来做排序
            order by字段为第一张表 才能用索引做排序
        5.3.8 压缩(前缀压缩)索引
        5.3.9 冗余和重复索引
            索引越多导致插入速度越慢 一般来说增加索引导致insert update delete等操作速度变慢
        5.3.10 未使用的索引
            建议删除 查询每个索引使用的状态
        5.3.11 索引与锁
            Innodb的行锁效率跟高 内存使用少 锁定超过的行需要
            如果不能使用索引查找和锁定行的话 mysql会做全表扫描并锁定所有的行 不管是否需要
        Innodb在二级索引中使用共享锁 但访问主键索引时使用排他锁 
    5.4 索引案例学习
        5.4.1 支持多种过滤条件
        5.4.2 避免多个范围条件
        5.4.3 对于翻页 可用延迟关联技术解决
    5.5 维护索引和表
### 第六章 查询性能优化
    6.1 为什么查询会很慢
    6.2 慢查询基础：优化数据访问
        查询不需要的记录(增加网络开销和CPU消耗和内存资源)
        多表关联时返回全部列
        总是去除全部列
        重复查询相同的数据
        6.2.1 是否向数据库请求了不需要的数据
        6.2.2 mysql是否存在扫描额外的记录
    6.3重构查询的方式
        6.3.1 一个复杂查询时多个简单查询
        6.3.2 切分查询
        6.3.3 分解关联查询
    6.4 查询执行的基础
            1 客户端发起查询给服务器
            2 服务器查询缓存 命中缓存则返回 否则下一阶段
            3 服务器sql解析 预处理优化器产生执行计划
            4 存储引擎api执行查询
            5 返回结果到客户端
        6.4.1 mysql客户端/服务器通信协议
            半双工，任意时刻要么客户端发送数据给服务器还是服务器发送数据给客户端，这两个动作不可能同时发生
            
            多数链接mysql库函数偶读可以通过获得全部结果并缓存到内存中。
            查询状态 show full processlist
                sleep 线程正在等待客户端发送新的请求
                query 线程正在执行查询或者正在将结果发送给客户端
                locked 等待资源锁
                analyzing and statistics 线程正在收集存储引擎的统计信息 并生成查询的执行计划
                copying to tmp table 复制到临时表
                sorting result 线程对结果进行排序
                sending data 向客户端发送数据
        6.4.2 查询缓存
        6.4.3查询优化处理
            语法解析器和预处理 
            查询优化器 选择成本最小的那个
              
                
        
                            
                            
     
                            
                            
                                              
                                                    
        
           