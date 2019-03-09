=电商数据库设计及架构优化实践

==设计电商常用的模块的数据库设计

注册会员，展示商品，加入购物车，生成订单

==准备
mysql5.7
navicat
linux shell

==模块

用户模块 完成用户的注册和登陆验证

商品模块 前后台商品管理和浏览

订单模块 订单及购物车的生成和管理

仓配模块 库存和物流管理

==设计规范

数据结构：
逻辑设计->物理设计

实际工作：
逻辑设计+物理设计

物理设计：
表名 字段名 字段类型

【规范并非不可违背，但是真的有必要吗？】

数据库命名规范
	所有数据库对象名称必须使用小写字母来命名并用下划线分割单词
	所有数据库对象名称禁用MySQL保留关键字
	见名识义，最好不要超过32个字符
	所有的临时表必须以tmp为前缀并且以日期为后缀
	所有的备份表必须以bak~
	所有存储相同数据的列名和列类型必须一致

数据库基本设计规范
	所有表必须使用Innodb存储引擎
		默认引擎
		支持事务,ACID特性，行级锁，更好的恢复性，高并发下的性能更好
	数据库和表的字符集UTF-8，一定要统一
	MySQL中UTF-8字符集汉字一个占3个字节
	所有的表和字段都需要加注释 comment ‘’，这样从一开始就进行数据字典的维护
	尽量控制单表数据量的大小，简易控制在500w
		可以用历史数据归档（日志），分库分表手段（业务数据，订单表）
	谨慎使用MySQL分区表
		分区表在物理机上表现为多个文件，在逻辑上表现为一个表
		谨慎选择分区键，跨分区查询效率可能很低
		建议采用物理分表的方式管理大数据
	尽量做到冷热数据分离，减少表的宽度
		4096
		减少磁盘IO，保证热数据的内存缓存命中率
		更有效利用缓存，避免读入无用冷数据，不可避免的select * 
		经常一起使用的列放到一个表中
	·禁止·在表中建立预留字段
		预留字段很难做到见名识义
		预留字段无法确认存储数据类型，无法选择合适的类型，一般varchar
		对字段名修改倒是可以行级加锁，但是对数据类型修改必须锁定表，严重影响并发
		添加一个字段的成本远低于修改一个字段
	·禁止·在数据库中存储图片，文件，二进制数据
		图片的增量很快，严重影响IO
	·禁止·在线上做数据库压测
	·禁止·从开发环境，测试环境直接连接生产环境的数据库


数据库索引设计规范
	·不要滥用索引·，限制每张表上的索引数量，建议单张表的索引不超过5个
		增加读效率，降低写效率，有时也降低读效率
		MySQL优化器会统计索引信息，然后评估生成最好的查询计划，如果索引效率都差不多，会影响评估时间，进而影响读效率
		`禁止每列一个索引`，5.6以后虽然可以合并索引，但是效率远没有联合索引查询的效果好
	每个Innodb表必须有一个主键
		因为Innodb是按照主键这个索引顺序来组织表的
		不能使用频繁更新的为主键，不能使用多列（联合索引）主键，因为Innodb是一个索引组织表，频繁变动的主键就会增大IO操作，影响数据库性能，同时也会占用CPU资源
		不使用UUID,MD5,HASH,字符串列作为主键
		使用自增ID
	常见的索引列建议
		在SELECT, UPDATE, DELETE 从句的WHERE从句中的列
		包含在ORDER BY, GROUP BY, DISTINCT中的字段
		多表join的关联列
	如何选择索引列的顺序
		区分度最高的列放在联合索引的最左侧（主键）
		尽量把字段长度小的列放在联合索引的最左侧
		使用最频繁的列放在联合索引的左侧
	避免建立冗余，重复索引->还是增加了评估时间
		重复：primary key(id), index(id), unique index(id)
		冗余：index(a, b, c), index(a, b), index(a)
	对于频繁的查询优先考虑使用覆盖索引
	  	覆盖索引：就是包含了所有查询字段的索引（SELECT中的 WHERE中的 GROUP BY中的）绝对不能用SELECT *
	  	覆盖索引好处：Innodb先根据索引定位，然后再二次查找需要的数据，如果覆盖索引的话就一次查找了，减少了IO操作和提升了效率；可以把随机IO变为顺序IO，加快查询效率
	尽量避免使用外键
		不建议使用外键约束，但一定要再表与表之间的关联键上建立索引
		外键可用与保证数据的参照完整性，但是建议在业务端实现
		外键会影响父表和子表和写操作从而降低性能

数据库字段设计规范
	·字段类型的设计会直接影响数据库性能，大概就是影响建立索引的数目· 
	1.优先选择存储需要的最小数据类型
		VARCHAR(N),N是字符数，不是字节数
		过大的字段长度会消耗更多的内存，VARCHAR虽然说是以实际用的大小存储，但是为了效率还是用预设的大小消耗内存
		将字符串转化为数字类型存储
		(ip转化为整形数值，INET_ATON('127.0.0.1') = 3518213, INET_NTOA(3518213) = '127.0.0.1'）
	2.对于非负型的数据来说，要优先使用无符号的整型来存储
		SIGNED INT "-214783648-214783648"
		UNSIGNED INT 0-4294967295    多差不多一倍的范围
	3.避免使用TEXT, BLOB数据类型
		TinyText，Text，MediumText, LongText,mysql不支持这类数据的内存内置表，当排序时就需要使用磁盘内置表，这类数据需要二次查询
		建议把这类的列分离到单独的扩展表中 ，千万别对这个表用select * 
		TEXT,BLOB只能用前缀索引
	4.避免使用ENUM数据类型
		修改ENUM值需要使用ALTER语句（太频繁，无可忽视的元数据堵）
		ENUM是整型数据的索引值存储的，ORDER BY得先将其转化为字符串类型在排序，无法使用索引，效率低
		禁止使用整型数值作为ENUM的枚举值
	5.尽可能把所有列都定义为NOT NULL 
		索引NULL列需要额外的空间来保存“null 还是 not null”，占用额外的空间
		进行比较和计算的时候要对NULL值做特别的处理
	6.日期类型千万别设置成字符串，要使用TIMESTAMP或DATETIME类型
		字符串类型无法使用日期函数进行计算比较
		用字符串存储日期要占用更多的空间（至少16字节），而datetime只需8个字节	
		TIMESTAMP 1970-01-01 00:00:01 - 2038-0(4个字节，本身就是INT存储)
	7.金额类数据，必须使用decimal类型
		decimal精准浮点数，计算不会丢失精度
		占用空间由定义的宽度决定
		可以存储bigint更大的整数数据

数据库SQL开发规范
	建议使用预编译语句进行数据库操作
		只传参数，比传递SQL语句更高效，减少网络带宽
		相同语句一次解析，多次使用，提高处理效率，防止sql注入
	避免数据类型的隐式转换（一般出现在where从句中，列类型和参数类型不一致）
		隐式转换会导致索引失效
	充分利用表上存在的索引
		避免使用双%，前置%查询条件， a like %123%，a like %123.
		一个SQL只能利用到联合索引中的一列进行范围查询，把范围查询的放在索引右侧
		使用left join 或 not exit 来优化not in操作
	程序连接不同的数据库要使用不同的账号，·禁止·跨库查询
		为数据库迁移和分库分表留出余地
		降低业务耦合度
		避免权限过大产生的安全风险
	·禁止·SELECT * 
		消耗更多的CPU和IO以及网络带宽资源
		无法使用覆盖索引
		可以减少表结构更改带来的影响
	·禁止·使用不含字段列表的INSERT语句
		可以减少表结构更改带来的影响
	避免使用子查询，可以把子查询优化为join操作
		子查询的结果集无法使用索引
		子查询会产生临时表操作，如果子查询的数据量很大则严重影响效率
		临时表没索引，消耗过多CPU IO
	避免使用JOIN关联太多的表
		每Join一个表会多占用一部分内存（join_buffer_size）,不合适会产生服务内存溢出
		会产生临时表影响效率
		不建议超过5个
	减少同数据库的交互次数
		数据库更适合批量操作
		合并多个相同的操作到一起，可以提高处理效率
		alter table t1 add column c1 int, change column c2 c2 int...
	使用in代替or（对同一列）
		in值不要超过500个
		in可以更有效用索引
	禁止使用order by rand()进行随机排序
		会把表中所有符合条件的数据装载到内存中进行排序
		会消耗大量的CPU IO 内存资源
		推荐在程序中先获取一个随机值，然后再从数据库中获取数据
	WHERE从句中禁止对列进行函数转换和计算
		会导致MySQL优化器无法使用索引
		WHERE DATE(createtime) = '20120202'
		-> where createtime >= '20120202' and createtime < '20120203'
	在明显不会有重复值时要使用UNION ALL 而不是 UNION
		因为union会把所有数据放到临时表中后再进行去重操作
		union all不会进行去重
	拆分复杂的大SQL为为多个小SQL
		MySQL一个SQL只能使用一个CPU进行计算


数据库操作行为规范
	超过100万行的批量写操作，要分批多次进行操作
		大批量的写操作会造成严重的主从延迟
		binlog日志为row格式时会产生大量的日志
		避免产生大事务操作（堵塞）
	对于大表使用pt-online-schema-change修改表结构
		新表复制旧表，然后加旧表一个短时间锁，替换删除
		避免大表修改的主从延迟
		避免对表字段进行修改时进行锁表
	禁止为程序使用账号赋予super权限
		当数据库达到最大连接数限制，还允许一个super权限用户连接（维护用的）
	对于程序连接数据库账号，遵循权限最小原则
		只有读需求就设只读
		程序使用的数据库账号只允许在一个db下使用，不准跨库
		程序使用账号不准有drop权限

====================================================================
## 用户模型设计
（作用：管理和维护用户信息）
#### 用户实体：
*   用户真实姓名
* 登录名
*	邮箱
*	密码
*	手机号
*	证件类型
*	相应的证件号码
*	性别
*	邮政编码
*	地址
*	积分
*	注册时间
*	生日
*	用户状态
*	用户级别
 *   级别最低积分
* 级别最高积分
* 用户余额
*  ......
#### 思考：如何将用户的属性存储到表中？
        【放入一个表】
	    优点:易于数据存取
	    问题1：数据插入异常
	        例子：insert into 用户表(会员级别) values('青铜')，这条SQL执行的表没有主键啊
	    问题2：数据更新异常，要修改一行值不得不修改多行数值
	        例子：用户等级名称要从青铜->注册会员
	        update 用户表 set 等级='注册会员' where 等级='青铜'
        问题3：数据的删除异常，删除一行时不得不删除另一行的数据
	        例子：如何删除用户等级名为青铜的等级，不是用户
	        delete from 用户表 where 用户等级='青铜'
	    问题4：数据存在冗余
	    问题5：数据表过宽，会影响修改表结构的效率，还有万一不小心再用到SELECT * 
        【解决方式】
        数据库的设计范式
            * 满足第三范式就可以满足最低要求，开发一般遵循这一条就够了
            * 如果一个表的列和其他列之间既不包含部分函数依赖关系，也不包含传递函数依赖关系         i，那么这个表的设计就符合第三范式
            （只用一个表的话：用户id->用户积分->用户积分，已经违反第三范式）
        1.拆分原用户表以符合第三范式
        2.尽量做到，事无绝对，尤其是与开发要结合团队和成本问题
        3.尽量做到冷热数据分离	    
#### 合理的数据库表设计	    
	    1.用户登录表{登录名，密码，用户状态}
	    2.用户地址表{省，市，区，邮编，地址}
	    3.用户信息表{姓名，证件类型，证件号码，手机号，邮箱，性别，积分，注册时间，会员级      别,生日，用户余额}
* `用户登陆表`
	```
    create table customer_login(
		customer_id int unsigined AUTO_INCREMENT NOT NULL comment '用户ID',
		login_name varchar(20) NOT NULL comment '用户登录名',
		password char(32) NOT NULL comment 'md5加密->32位',
		user_stats tinyint NOT NULL DEFAULT 1 comment '用户状态',
		modified_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP	
		ON UPDATE CURRENT_TIMESTAMP comment '最后修改时间',
		primary key pk_customerid(customer_id)
	) engine=Innodb comment '用户登录表'
	;
	```
* `用户信息表`
	```
    create table customer_inf(
		customer_inf_id int unsigned AUTO_INCREMENT NOT NULL comment '自增主键ID',
		customer_id int unsigned not null comment 'customer_login的自增主键ID',
		customer_name varchar(20) not null comment '用户真实姓名',
		identity_card_type tinyint not null default 1
		comment '证件类型 1身份证，2军官证，3护照',
		mobile_phone int unsigned comment '手机号',
		gender char(1) comment '性别',
		user_point int not null default 0 comment '用户积分',
		register_time timestamp not null comment '注册时间',
		birthday datetime comment '生日',
		customer_level tinyint not null default 1
		comment '会员级别 1普通，2青铜，3白银，4黄金，5钻石',
		user_money decimal(8,2) not null default 0.00 comment '用户余额',
		modefied_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '最后修改时间',
		primary key pk_customerinfid(customer_inf_id)
	) engine=Innodb comment '用户信息表'
	;
	```
* `用户级别信息表`
	````
	create table customer_level_inf(
		customer_level tinyint not null auto_increment comment '会员级别ID',
		level_name varchar(10) not null comment '会员级别名称',
		min_point int unsigned not null default 0 comment '该级别最低积分',
		max_point int unsigned not null default 0 comment '该级别最高积分',
		modified_time timestamp not null default CURRENT_TIMESTAMP 
		ON UPDATE CURRENT_TIMESTAMP COMMENT '最后修改时间',
		primary key pk_levelid(customer_level)
	) engine=Innodb comment '用户级别信息表'
	;
	```
* `用户地址表`
    ```
    create table order_customer_addr(
		customer_addr_id int unsigned AUTO_INCREMENT NOT NULL
		comment '自增主键ID',
		customer_id int unsigned not null comment 'customer_login的自增ID',
		zip smallint not null comment '邮编',
		province smallint not null comment '省份id',
		city smallint not null comment '城市id',
		district smallint not null comment '区id',
		address varchar(200) not null comment '门牌号',
		is_default tinyint not null comment '是否默认',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '最后修改时间',
		primary key pk_customeraddrid(customer_addr_id)
	) engine=Innodb comment '用户地址表'
	;
	```
* `用户积分日志表`
	```
	create table customer_point_log(
		point_id int unsigned not null AUTO_INCREMENT comment '积分日志ID',
		customer_id int unsigned not null comment '用户ID',
		source tinyint unsigned not null
		comment '积分来源 0订单，1登陆，2活动',
		refer_number int unsigned not null default 0
		cooment '积分来源的相关编号',
		change_point smallint not int default 0 comment '变更积分数',
		create_time timestamp not null comment '积分日志生成时间',
		primary key pk_pointid (point_id)
	) engine=Innodb comment '用户积分日志表'
	;	
* `用户余额变动表`
    ```登录
	create table customer_balance_log(
		balance_id int unsigned not null AUTO_INCREMENT comment '余额日志ID',
		customer_id int unsigned not null comment '用户ID',
		source tinyint unsigned not null default 1
		comment '记录来源 1订单，2退货单',
		source_sn int  unsigned not null comment '相关单据ID',
		create_time timestamp not null default CURRENT_TIMESTAMP
		comment '记录生成时间',
		amount decimal(8,2) not null default 0.00 comment '变动金额',
		primary key pk_balanceid(balance_id)
	) engine=Innodb comment '用户金额变动表'
	;
	```
* `用户登陆日志表`
	```
    create table customer_login_log(
		login_id int unsigned not null AUTO_INCREMENT
		comment '登陆日志id',
		customer_id int unsigned not null comment '登陆id',
		login_time timestamp not null comment '用户登陆时间',
		login_ip int unsigned not null comment '登陆ip',
		login_type tinyint not null 
		comment '登陆类型 0未成功 1成功',
		primary key pk_loginid(login_id)
	) engine=Innodb comment '用户登录日志表'
	;
	```
#### 浅谈MySQL分区表
	1.确认MySQL服务器是否支持分区表
	    mysql> SHOW PLUGINS; --->partition
	2.在逻辑上为一个表，在物理上存储在多个文件中
	3.分区键和主键必须在一块
* HASH分区
	````
    create table customer_login_log(
		customer_id int unsigned not null comment '登陆id',
		login_time timestamp not null comment '用户登陆时间',
		login_ip int unsigned not null comment '登陆ip',
		login_type tinyint not null 
		comment '登陆类型 0未成功 1成功'
	) engine=Innodb comment '用户登录日志表'
	PARTITION BY HASH(customer_id)
		PARTITIONS 4;
	````
	```    
    按照HASH分区
	    1.根据MOD(分区键，分区数)的值把数据行存储到表的不同分区中
		2.数据可以平均的分布在各个分区中
		3.HASH分区的键值必须是一个int类型的值，或者可以通过函数转换成int，因为需要      对键值取模
	```	    
	```
	 非分区的磁盘文件
	    customer_login_log.frm 元数据信息，
	    customer_login_log.ibd innodb的数据文件
	 分区的磁盘文件 
	    customer_login_log.frm
	    customer_login_log#P#p0.ibd
	    customer_login_log#P#p1.ibd
	    customer_login_log#P#p2.ibd
        customer_login_log#P#p3.ibd
	 ```
	 ```
	    如何建立HASH分区
	        PARTITION BY HASH(customer_id)
		        PARTITIONS 4;
	        PARTITION BY HASH(UNIX_TIMESTAMP(login_time))
		        PARTITIONS 4;
	    哪些函数可以用于创建hash分区表？Google撒！
	```	
* RANGE分区
   ````
	create table customer_login_log(
		customer_id int unsigned not null comment '登陆id',
		login_time timestamp not null comment '用户登陆时间',
		login_ip int unsigned not null comment '登陆ip',
		login_type tinyint not null 
		comment '登陆类型 0未成功 1成功'
	) engine=Innodb comment '用户登录日志表'
	PAITITION BY RANGE (customer_id) (
		PARTITION p0 VALUES LESS THAN (10000), 0~9999
		PARTITION p1 VALUES LESS THAN (20000), 10000~19999
		PARTITION p2 VALUES LESS THAN (30000), 20000~29999
		PARTITION p0 VALUES LESS THAN MAXVALUE	>30000 
	);
   ````
     ```
        1.根据分区键值的范围把数据行存储到表的不同分区中
	    2.多个分区范围要连续，但是不要重叠
	    3.默认情况下使用values less than属性，如果id只有1-100，那么不会包含100这个值
	```
	```
	RANGE分区适应场景
		1.分区键为日期或是时间类型。如果是以ID，很可能会产生前10000很活跃，后面的不     活跃，失去了分区的意义，而且时间日期也便于归档。
		2.所有查询中都包括分区键，避免跨分区扫描
		3.定期按分区范围清理历史数据
	```
* LIST分区   ````
	create table customer_login_log(
		customer_id int unsigned not null comment '登陆id',
		login_time timestamp not null comment '用户登陆时间',
		login_ip int unsigned not null comment '登陆ip',
		login_type tinyint not null 
		comment '登陆类型 0未成功 1成功'
	) engine=Innodb comment '用户登录日志表'
	PARTITION BY LIST(login_type)(
		PARTITION P0 VALUES in (1, 3, 5, 7),
		PARTITION P1 VALUES in (2, 4, 6, 8)
	);
    ````
    ```
    按分区键值的列表进行分区
	    同范围分区一样，各分区的列表值不能重复
	    每一行数据必须能找到对应的分区列表，否则插入失败
    ```
#### 如何为customer_login_log表分区
* 业务场景
	    1.用户每次登陆都会产生一条日志记录
		2.用户登陆日志保存一年，然后删除	
* 使用RANGE分区
	以login_time作为分区键（因为前台肯定不需要，只是出问题手动查询）
	````
    create table customer_login_log(
		customer_id int unsigned not null comment '登陆id',
		login_time timestamp not null comment '用户登陆时间',
		login_ip int unsigned not null comment '登陆ip',
		login_type tinyint not null 
		comment '登陆类型 0未成功 1成功'
	) engine=Innodb comment '用户登录日志表'
	PARTITION BY RANGE(YEAR(login_time))(
		PARTITION P0 VALUES LESS THAN (2015),
		PARTITION P1 VALUES LESS THAN (2016),
		PARTITION p2 VALUES LESS THAN (2017)		
	)
    ````
	采用如下sql可以查看分区状态
	````
    SELECT
		table_name,
		partition_name,
		partition_description,
		table_rows
	FROM
		information_schema.`PARTITIONS`
	WHERE table_name = 'customer_login_log';
    ````
	一般不会加MAXVALUE,因为不好管理数据，我们可以每年年底加一个
	````
        ALTER TABLE customer_login_log 
		ADD PARTITION (PARTITION p3 VALUES LESS THAN(2018))
    ````	
    `分区删除,但是你必须得先对数据进行归档`
	````	
        ALTER TABLE customer_login_log DROP PARTITION p0;
    ````	
    分区数据归档迁移条件：
	* mysql>5.7
	* 结构相同
	* 归档到的数据表一定要是非分区表
	* 非临时表，不能有外键约束
	* 归档引擎要是archive  
    对这个customer_login_log进行部分数据归档
	````	
        create table arch_customer_login_log(
			customer_id int unsigned not null comment '登陆id',
			login_time timestamp not null comment '用户登陆时间',
			login_ip int unsigned not null comment '登陆ip',
			login_type tinyint not null 
			comment '登陆类型 0未成功 1成功'
		) engine=Innodb comment '用户登录日志表'
		ALTER TABLE customer_login_log 
		exchange PARTITION p0 with table arch_customer_login_log;
		ALTER TABLE arch_customer_login_log engine=archive(archive表的体积小，但是只能读操作)
    ````		
    然后可以进行分区删除了！
	使用分区表应该注意的事项！
	* 结合业务场景选择分区键，尽量避免跨分区
	* 对分区表进行查询时最好在WHERE从句中包含分区键（避免跨分区）
	* 具有primary key和unique index的表，主键和唯一索引必须是分区键的一部分，不难理解，如果按照登录年份为分区键，以id为主键，Innodb是以索引来顺序存储数据的，但是分区又是以分区键，这个例子的用户在2011年登录，2015年的登录，还分个锤子区！所以说分区表其实更适合使用在MyISAM存储引擎
=============================
商品实体
	
品牌信息表
	create table product_brand_info(
		brand_id smallint unsigned auto_increment not null comment '品牌ID',
		brand_name varchar(50) not null comment '品牌名称',
		telephone varchar(50) not null comment '联系电话',
		brand_web varchar(100) comment '品牌网站',
		brand_logo varchar(100) comment '品牌logo url',
		brand_desc varchar(150) comment '品牌描述',
		brand_status tinyint not null default 0
			comment '品牌状态 0禁用，1启用',
		brand_order tinyint not null default 0
			comment '排序',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '最后修改时间',
		primary key pk_brandid(brand_id)
	) engine=Innodb comment '品牌信息表'
;

分类信息表,一般是三级树状分类
	create table product_category(
		category_id smallint unsigned auto_increment not null 
			comment '类别id',
		category_name varchar(10) not null comment '类别名称', 
		category_code varchar(10) not null comment '类别编码',
		parent_id smallint unsigned not null default 0
			comment '父级id',
		category_level tintint not null default 1 comment '类别级别',
		category_status tinyint not null default 1 comment '类别状态',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '改变时间',
		primary key pk_categoryid(category_id)
	) engine=Innodb comment '分类信息表'
;

商品供应商信息表
	create table product_supplieer_info(
		supplier_id int unsigned auto_increment not null 
			comment '供应商id',
		supplier_code char(8) not null comment '供应商编码',
		supplier_name char(50) not null comment '供应商名称',
		supplier_type tinyint not null 
			comment '供应商类型 1自营 2平台',
		link_man varchar(10) not null comment '联系人',
		phone_number varchar(50) not null comment '联系电话',
		bank_name varchar(50) not null comment '供应商银行开户名称',
		bank_account varchar(50) not null comment '银行账号',
		address varchar(200) not null comment '供应商地址',
		supplier_status tinyint not null default 0
			comment '状态 0禁用 1启用',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '最后修改时间',
		primary key pk_supplierid(supplier_id)
	) engine=Innodb comment '供应商信息'
	;

商品信息表（不进行拆分，因为大部分字段都是在一起使用的，而且经常使用的一般都是放在缓存和页面静态化）
	create table product_info(
		product_id int unsigned auto_increment not null comment '商品自增id',
		product_code char(16) not null comment '商品编码',
		product_name varchar(20) not null comment '商品名称',
		bra_code varchar(50) not null comment '国条码',
		brand_id int unsigned not null comment '品牌表的id',
		one_category_id smallint unsigned not null comment '一级分类id',
		two_category_id smallint unsigned not null comment '二级分类id',
		three_category_id smallint unsigned not null comment '三级分类id',
		supplier_id int unsigned not null comment '商品供应商id',
		price decima(8,2) not null comment '商品销售价格',
		average_cost decimal(18,2) not null comment '商品加权成本',
		publish_status tinyint not null default 0 comment '0上架，1下架',
		audit_status tinyint not null default 0 comment '0未审核，1已审核',
		weight float comment '商品重量',
		length float comment '商品长度',
		color_type enum('红','黄','蓝','黑') comment '商品颜色',
		production_date datetime not null comment '生产日期',
		indate timestamp not null default CURRENT_TIMESTAMP comment '商品录入时间',
		modified_time timestamp not null default CURRENT_TIMESTAMP 
		ON UPDATE CURRENT_TIMESTAMP comment '最后修改时间',
		primary key pk_productid(product_id)
	) engine=innodb comment '商品信息表'
	;

商品图片表(图片一定要放到图片服务器和cdn上)
	create table product_pic_info(
		product_pic_id int unsigned auto_increment not null comment '商品图片ID',
		product_id int unsigned not null comment '商品ID',
		pic_desc varchar(50) comment '图片描述',
		pic_url varchar(200) not null comment '图片链接',
		is_master tinyint not null default 0 comment '是否主图 0非主图 1主图',
		pic_order tinyint not null default 0 comment '图片排序',
		pic_status tinyint not null default 1 comment '图片是否有效 0无效 1有效',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_picid(product_pic_id)
	) engine=innodb comment '商品图片信息'
	;
	商品评论表
	create table product_comment(
		comment_id int unsigned not null auto_increment comment '商品评论表ID',
		product_id int unsigned not null comment '商品id',
		order_id int unsigned not null comment '订单id',
		customer_id int unsigned not null comment '用户id',
		title varchar(50) comment '评论标题',
		content varchar(300) comment '评论内容',
		audit_status tinyint not null default 4 comment '默认4未审核，1已审核',
		audit_time timestamp not null default CURRENT_TIMESTAMP comment '审核通过时间',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_commentid(comment_id)
	) engine=innodb comment '评论表'
	;
=======================================
订单实体
订单主表
	create table order_master(
		order_id int unsigned auto_increment not null comment '订单ID',
		order_sn bigint unsigned not null comment '订单编号 yyyymmddnnn',
		customer_id int unsigned not null comment '下单id',
		shipping_user varchar(10) not null comment '收货人名',
		province smallint not null comment '省',
		city smallint not null comment '市',
		payment_method tinyint not null comment '支付方式',
		order_money decimal(8,2) not null comment '订单金额',
		district_money decimal(8,2) not null default 0.00 comment '优惠金额',
		shipping_money decimal(8,2) not null default 0.00 comment '运费金额',
		payment_money decimal(8,2) not null default 0.00 comment '支付金额',
		shipping_comp_name varchar(10) comment '快递公司',
		shipping_sn varchar(50) comment '快递单号',
		create_time timestamp not null default CURRENT_TIMESTAMP comment '下单时间',
		receive_time datetime comment '收货时间',
		pay_time datetime comment '支付时间',
		order_point int unsigned not null default 0 comment '订单积分',
		invoice_title varchar(100) comment '发票抬头',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_orderid(order_id)
	) engine=Innodb comment '订单主表'
	;
订单详情表（避免冗余和写数据异常）
	create table order_detail(
		order_detail_id int unsigned not null auto_increment comment '订单详情表id',
		order_id int unsigned not null comment '订单id',
		product_name varchar(50) not null comment '商品名称',
		product_cnt int not null default 1 comment '商品数量',
		product_price decimal(8,2) not null default 0.00 comment '商品单价',
		average_cost decimal(8,2) not null default 0.00 comment '平均成本价格',
		weight float comment '商品重量',
		fee_money decimal(8,2) not null default 0.00 comment '优惠分摊金额',
		w_id int unsigned not null comment '仓库id',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_orderdetailid(order_detail_id)
	) engine=innodb comment '订单详情表'
	;
购物车表
	create table order_cart(
		cart_id int unsigned not null auto_increment comment '购物车id',
		customer_id int unsigned not null comment '用户id',
		product_id int unsigned not null comment '商品id',
		product_amount int not null comment '商品数量',
		price decimal(8,2) not null comment '商品价格',
		add_time timestamp not null default CURRENT_TIMESTAMP comment '加入购物车时间',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_ordercartid(order_id)
	) engine=innodb comment '购物车表'
	;
==============================
仓库信息表
	create table warehouse_info(
		w_id int unsigned not null auto_increment comment '仓库id',
		warehouse_sn char(5) not null comment '仓库编码',
		warehouse_name varchar(10) not null comment '仓库名称',
		warehouse_phone varchar(20) not null comment '仓库联系电话',
		contact varchar(10) not null comment '仓库联系人',
		province smallint not null comment '省',
		city smallint not null comment '市',
		address varchar(100) not null comment '仓库地址',
		warehouse_status tinyint not null default 1
			comment '仓库状态 0禁用 1启用',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_wid(w_id);
	) engin=innodb comment '仓库信息表'
	;
商品的库存表
	create table warehouse_product(
		wp_id int unsigned not null auto_increment comment '商品库存id',
		product_id int unsigned not null comment '商品id',
		w_id smallint int unsigned not null comment '仓库id',
		current_cnt smallint unsigned not null default 0 comment '当前商品数量',
		lock_cnt int unsigned not null default 0 comment '当前占用数据',
		in_transit_cnt int unsigned not null default 0 comment '在途数据',
		average_cost decimal(8,2) not null default 0.00 comment '移动加权成本',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_wpid(wp_id)
	) engine=innodb comment '商品库存表'
	;
===================================
物流公司信息
	create table shipping_info(
		ship_id tinyint unsigned not null auto_increment comment '物流公司主键id',
		ship_name varchar(20) not null comment '物流公司名称',
		ship_contact varchar(20) not null comment '物流公司联系人',
		telephone varchar(20) not null comment '物流公司电话',
		price decimal(8,2) not null default 0.00 comment '配送价格',
		modified_time timestamp not null default CURRENT_TIMESTAMP
		ON UPDATE CURRENT_TIMESTAMP comment '更改时间',
		primary key pk_shipid(ship_id)
	) engine=innodb comment '物流信息表'
	;
=========================================
DB规划
	为数据库迁移更加方便
	避免跨库操作，把经常放在一起的关联查询的表放到一个db中
	为了方便识别表所在的db，在表名前增加库名前缀

用户数据库mc_customerdb
	customer_inf
	customer_login
	customer_level_inf
	customer_login_log
	customer_point_log
	customer_balance_log

商品数据库mc_productdb
	product_info
	product_pic
	product_comment
	product_category
	product_brand_info

订单数据库mc_orderdb
	order_master
	order_detail
	order_customer_addr
	warehouse_info
	warehouse_product
	shipping_info
	order_cart

=================================
进入虚拟机
mysql -uroot -p -e "create database mc_customerdb"
mysql -uroot -p mc_customerdb < mc_customerdb.sql

mysql -uroot -p -e "create database mc_orderdb"
mysql -uroot -p mc_orderdb < mc_orderdb.sql

mysql -uroot -p -e "create database mc_productdb"
mysql -uroot -p mc_productdb < mc_productdb.sql

=================================================
常见的业务处理
1.对评论进行分页展示
	执行计划分析（加EXPLAIN）
	EXPLAIN
	select customer_id, title, content
		from 'product_comment'
		where audit_status=1
		and product_id=199726
		limit 0,5;
	执行计划能告诉我们什么？
		sql如何使用索引的
		关联查询的执行顺序
		查询扫描数据行数
	执行计划结果？
	ID列
		ID列的数据为一组数字，表示执行select语句的执行顺序
		ID值相同时，执行顺序从上到下
		ID值越大优先级越高，越先被执行
	SELECT TYPE
		SIMPLE 不包含子查询和UNION操作的查询
		PRIMARY 查询中如果包含任何子查询，那么最外层的查询则被标记为primary
		SUBQUERY SELECT列表中的子查询
		DEPENDENT SUBQUERY 依赖外部结果的子查询
		UNION 当union操作的第二个或是之后的查询的值为union
		DEPENDENT UNION 当union作为子查询，第二个或是第二个后的查询的select_type
		UNION RESULT union产生的结果集
		DERIVED 出现在from子句中的子查询
	TABLE列
		输出数据行所在的表名，或者别名
		<unionM,N>由id为M N查询union产生的结果集
		<derivedN>/<subqueryN>由id为N的查询产生的结果
	PARTITIONS列
		对于分区表，显示查询的分区id
		对于非分区，显示null
	TYPE列
		可以看出mysql查询的性能，如果all这种全表扫描，考虑加索引
	EXTRA列
		Distinct 优化distinct操作，在找到第一个匹配的元组后即停止找相同的值操作
		NOT EXISTS 优化not exists
		Using filesort 使用额外排序，不是索引排序，在内存中进行排序，一般在使用group by， order by出现
		Using index 使用了覆盖索引
		Using tempoary mysql需要使用临时表来进行查询，常见于排序，子查询，分组查询
		Using where mysql在引擎层获得数据再在服务器层进行过滤
		select tables optimized away 直接通过索引来获得数据，不用访问表
	POSSIBLE_KEYS列
		指出mysql能使用到哪些索引来优化查询，但是不一定都会真的被使用
	KEY列
		mysql实际使用到的索引，如果没有使用则为NULL，但是出现的也不一定会在POSSIBLE_KEYS中出现，因为可能使用覆盖索引
	KEY_LEN列
		表示索引字段的最大长度
		key_len长度由字段长度定义，并非实际的长度
	Ref列
		利用索引查询时的值都来源于哪里
	ROWS列
		根据索引的估算返回行数，不准确
	FILTERD列
		表示返回结果行数占需要读取行数的百分比，不准确
	限制：
		无法展示存储过程，触发器，UDF对查询
	对评论分页优化？
	EXPLAIN
	select customer_id, title, content
		from 'product_comment'
		where audit_status=1
		and product_id=199726
		limit 0,5;
	1.如果要建立一个联合索引，那么得考虑把哪个列放在左侧，这个时候用以下方法
	select count(distinct audit_status)/count(*)
			as audit_rate,
		count(distinct product_id)/count(*)
			as product_rate
	from product_comment;
	--->
	audit_rate 0.0001, product_rate 0.8127
	product_rate 的区分度更好，放在左侧
	--->
	create index idx_productID_auditStatus
		on product_comment(product_id, audit_rate)
	2.sql改写
	select t.customer_id, t.titile, t.content
	from (
		select 'comment_id'
			from product_comment
			where product_id=11111 and audit_status=1 limit 0,5
	) a join product_comment t
	on a.comment_id = t.comment_id
	数据库的IO=索引的IO+索引查询对应的表数据IO
2.如何删除重复数据
	删除评论表中对同一订单同一商品的重复评论，只保留最早一条
	1）查看是否存在对同一订单同一商品的重复评论
	2）备份product_comment表
	3）删除重复
	1)
	select order_id,product_id count(*)
	from product_comment
	group by order_id,product_id HAVING count(*) > 1
	2)
	create table bak_product_comment_180802
	LIKE 
	product_comment
--
	insert into bak_product_comment_180802
	select * from product_comment
	3)
	DELETE a 
	FROM product_comment a
	JOIN (
		select order_id,product_id,MIN(comment_id) as comment_id
		from product_comment
		group by order_id,product_id
		HAVING count(*) >= 2
	) b on a.order_id=b.order_id and a.product_id=b.product_id
	AND a.comment_id>b.comment_id
3.如何进行分区间统计
	统计消费总金额大于1000的，800-1000，500-800，小于500的人数
	select count(case when ifnull(total_money),0) >= 1000 then a.customer_id end) as 	'大于1000',
		count(case when ifnull(total_money),0) >= 800 and ifnull(total_money,0) < 1000
		then a.customer_id end) as '800-1000',
		count(case when ifnull(total_money),0) < 500 then a.customer_id end ) as '<500'
	from mc_customerdb.'customer_login' a
	left join(
		select customer_id,SUM(order_money) as total_money
		from mc_orderdb.'order_master'
		group by customer_id
	) b
	on a.'customer_id' = b.'customer_id'
===============
捕捉有问题的sql
启用mysql慢查日志
	set global slow_query_log_file=/sql_log/slow_log.log
	set global log_queries_not_using_index=on
	set global long_query_time=0.001 抓取执行参数大于0.001的sql
	set global low_query_log = on
如何分析慢查询日志记录的内容
	mysqldumpslow slow-mysql.log
========================
数据库的备份
	对于任何数据库来说，备份都是很重要的
	数据库复制不能取代备份的作用。主从复制时间很短的，如果master进行了大量删除操作，slave也会很快被更新，这样想恢复master原来的数据是无法通过slave的
逻辑备份和物理备份
	逻辑备份的结果为sql语句，适合所有的引擎，但是需要花费很多时间，mysqldump
	物理备份是对数据库目录的拷贝，对于内存表只备份结构，XtraBackup物理热备工具
全量备份和增量备份
	全零备份是对整个database的一个完整备份
	增量备份是指上一次全量或者增量之后的再次增量数据的备份，mysqldump不能实现
使用mysqldump进行备份
	1>创建一个备份账号
	mysql> create user 'backup'@'localhost' identified by '123456';
	2>授权
	mysql> grant select,reload,lock,tables,replication client,show view,event,process
	on *.* to 'backup'@'localhost'
	3>备份
	[root@hufan ~ ]# mysqldump -ubackup -p --master-data=2 --single-transaction --routines --triggers --events mc_orderdb > mc_orderdb.sql
	[root@hufan ~ ]# grep "CREATE TABLE " mc_orderdb.sql
	[root@hufan ~ ]# mysqldump -ubackup -p --master-data=2 --single-transaction --routines --triggers --events mc_orderdb order_master > order_master.sql
	4>全备
	[root@hufan ~ ]# mysqldump -ubackup -p --master-data=2 --single-transaction --routines --triggers --events --all-databases > mc.sql
	5>备份到特定目录下
	[root@hufan]# mkdir -p /tmp/mc_orderdb
	[root@hufan tmp]# chown mysql:mysql mc_orderdb/ 
	[root@hufan]# mysql -uroot -p
	mysql> grant file on *.* to 'backup'@'localhost'
	mysql> exit
	[root@hufan]# mysqldump -ubackup -p --master-data=2 --single-transaction
	--routines --triggers --events --tab"/tmp/mc_orderdb" mc_orderdb
==
	案例
	[root@hufan]# cd /data/db_backup/
	[root@hufan db_backup]# mysqldump -ubackup -p --master-data=2 --single-transaction
	--where "order_id">1000 and "order_id"<1050 mc_orderdb order_master > order_master_1000.sql
==
	工作中还是主要用脚本 .sh
如何恢复mysqldump备份的数据库（速度取决于执行sql的速度和服务器的IO性能，恢复单线程）
	mysql -uroot -p -e"create database bak_orderdb"
	mysql -uroot -p bak_orderdb < mc_orderdb.sql
	INSERT INTO mc_orderdb.'order_master'(
		'order_id',
		'order_sn',
		'customer_id',
		...
	)
	select a.*
	from bak_orderdb.'order_master' a
	left join mc_orderdb.'order_master' b on a.order_id=b.order_id
	where b.order_id is null

如何将特定目录下的备份恢复
	mysql> source /tmp/mc_orderdb/regin_info.sql;
	mysql> show tables;
		...
		regin_info
		...
	mysql> desc regin_info;
		regin_id ...
		...	...
		regin_lever ...
	mysql> load data infile '/tmp/mc_orderdb/regin_info.txt' into table regin_info;
	OK;
如何进行指定的时间点的恢复
	先决条件
		具有那个时间点的全备
		具有自上次全备后到指定时间点的所有二进制日志
	[root@hufan ~ ]# mysqldump -ubackup -p --master-data=2 --single-transaction --routines --triggers --events mc_orderdb > mc_orderdb.sql
	[root@hufan ~ ]# mysql -uroot -p mc_orderdb < mc_orderdb.sql
	[root@hufan ~ ]# more mc_orderdb.sql
		84882
	[root@hufan ~ ]# cd /home/mysql/sql_log
		mysql-bin.00010
		mysql-bin.00011
	[root@hufan ~ ]# mysqlbinlog --base64-output=decode-rows -vv --start-position=84882
 	--database=mc_orderdb mysql-bin.00011 | grep -B3 DELETE | more
 		169348
 	[root@hufan ~ ]# mysqlbinlog --start-position=84882 --stop-position=169348
 	--database=mc_orderdb mysql-bin.00011 > mc_order_diff.sql
 	[root@hufan ~ ]# mysql -uroot -p mc_orderdb < mc_order_diff.sql

如何对mysql二进制日志备份
	[root@hufan ~ ]# mysql -uroot -p
	mysql> grant replication slave on *.* to 'repl'@localhost' identified by '123456'
	mysql> exit
	[root@hufan ~ ]# ll /home/mysql/sql_log
		...
		mysql-bin-000012
	[root@hufan data]# mkdir binlog_bak
	[root@hufan data]# cd binlog_bak
	[root@hufan binlog_bak]# mysqlbinlog -raw --read-from-remote-server --stop-never 
	--host localhost --port 3306 -urepl -p123456 mysql-bin.000012
	....一直在实时备份

对xtrabackup介绍
	用于在线备份innodb存储引擎的表
	支持多线程并发备份
	恢复必须把mysql实例重启
	innobackupex --user=backup --password=123456 --parallel=2 /data/db_backup/20180202 --no-timestamp
	innobackupex --apply_log /home/db_backup/20180202/
	mv /20180202 /home/mysql
	cd /home/mysql
	[root@hufan mysql]# /etc/init.d/mysqld stop
	[root@hufan mysql]# mv data data_bak
	[root@hufan mysql]# mv 20180202/ data
						chown -R mysql:mysql data
	...
增量备份
	innobackupex --user=backup --password=123456 /data/db_backup 
	...对db的更改
	innobackupex --user=backup --password=123456 --incrementl /home/db_backup --incrementl-basedir=/data/db_backup/2018-11-24_23-12-32/
增量备份恢复
	innobackupex --apply-log --repo-only /data/db_backup/2018-11-24_23-12-32
制定备份计划
	每天凌晨对数据库进行一次全备 .sh mysqldump innobackupex
	实时对二进制日志进行远程备份（在一台服务器上开启mysqlbinlog）

MySQL主从复制
	主库将变更的数据写入主库的binlog中
	从库的IO进程读取主库的binlog内容存储到Relay Log日志中
		1.二进制日志点
		2.GTID MYSQL5.7 建议
	从库的SQL进程读取Relay Log日志中内容在从库中重放
MySQL主从复制配置步骤
	1.配置主从数据库服务器参数（在上线前配置好）
	2.在master服务器创建用于中成功复制的数据库账号
	3.备份master服务器上的数据并初始化slave服务器数据
	4.启动复制链路
1>
master服务器
log_bin = /data/mysql/sql_log/mysql-bin
server_id = 100(不允许相同)
=====
slave服务器
log_bin= = /data/mysql/sql_log/mysql-bin
server_id = 101
relay_log = /data/mysql/sql_log/relay-bin
read_only=on
super_read_only=on
skip_slave_start=on
master_info_repository=TABLE
relay_log_info_repository=TABLE
2>
用于IO进程连接master服务器获取binlog日志，只需要replicetion slave权限
create user 'repl'@'ip' identified by 'password'
grant replication slave on *.* to 'repl'@'ip'
3>
建议主从服务器使用相同版本
建议使用全库备份的方式初始化slave数据
mysqldump --master-data=2 -uroot -p -A\
		  --single-transaction -R --triggers 
4>
	change master to 
		master_host='master_host_ip',
		master_user='repl',
		master_password='..',
		master_log_file='mysql_log_file_name',
		master_log_pos=...;
主从复制演示
	master放一些主要数据库
		[]# more /etc/my.cnf
		binlog 开启
		server-id 改成ip最后
		mysql> show variables like '%server_id%'
		mysql> set global server_id = 100;
	slave配置
		server-id ..
		relay_log = /home/mysql/sql_log/mysql-relay-bin
		master_info_repository=TABLE
		relay_log_info_repository=TABLE
		read-only=on
		如果是镜像安装相同的5.7以上mysql，那么每个主从服务器的mysql的uuid相同，这样会出现问题，所以我们要先删除从服务器mysql的uuid，然后重启后会生成一个新的
		[]# cd /home/mysql/data
		[]# rm -rf auto.cnf
		[]# /etc/init.d/mysql restart
	在主服务器建立登录账号
		[]# mysql -uroot -p 
		mysql> create user 'dba_repl'@'192.168.3.%' identified by '123456';
		mysql> grant replication slave on *.* to 'dba_repl'@'192.168.3.%';
		[]# cd /home/mysql/data/db_backup/
		[]# mysqldump -uroot -p --single-transaction --master-data --triggers
		--routines --all-database > all.sql
		[]# scp all.sql root@192.168.3.101:/root
	从服务器恢复备份
		[]# ll /root
		[]# more all.sql
			MASTER_LOG_FILE='mysql-bin.000017',MASTER_LOG_POS=663
		[]# mysql -uroot -p < all.sql
		[]# mysql -uroot -p 
		mysql> show databases
		...
		mysql> change master to master_host='192.168.3.100',
		-> master_user='dba_repl',
		-> master_password='123456',
		-> MASTER_LOG_FILE='mysql-bin.000017',
		-> MASTER_LOG_POS=663
		mysql> start slave;
		mysql> show slave status \G
			Relay_Master_Log_File: mysql-bin.000017
			Slave_IO_Running： YES
			Slave_SQL_Running: YES
	主服务器测试
		mysql> use mc_orderdb
		mysql> insert into t1 values(1);
	从服务器
		mysql> use mc_orderdb
		mysql> select * from t1;
		-------
		- id  -
		- 1   -
		-------
基于GDID的复制策略
	GDID:也就是全局事务ID，每一个在master上的事务都能在slave上生成一个唯一的id
	配置
	gtid_mode=on
	enforce-gtid-consistency
	log-salve-updates=on
	...
	change master to
		master_host='master_ip',
		master_user=,
		master_password=,
		master_auto_position=1
	限制
	无法再使用create table...select建立表
	无法在事务中使用create temporary table建立临时表
	无法使用关联更新同时更新事务表和非事务表
引入复制后的数据库架构
	虚拟ip（vip）：就是一个未分配给真实主机的ip，也就是说对外提供服务器的主机除了有一个真实ip外还有一个虚拟ip
	引入vip方法：脚本，MHA，MMM，keepalived
	Keepalived：
		实现主从数据库的心跳检测->健康监控
		当主DB宕机迁移vip到主备
		改主从复制为主主复制，但是只有一个主提供服务
	主主配置调整
		master配置
			auto_increment_increment=2
			auto_increment_offset=1
			1,3,5,7,9
		master备用配置
			auto_increment_increment=2
			auto_increment_offset=2
			2,4,6,8,10
	keepalived简介(主，主备都要安装keepalived)
		keepalived基于ARRP网络协议
		yum install keepalived -y
		vim /etc/keepalived/keepalived.conf
	主服务器
		vim /etc/my.cnf
			auto_increment_increment=2
			auto_increment_offset=1(动态参数，不需要重启mysql)
		mysql> set global auto_increment_increment=2;
		mysql> set global auto_increment_offset=1;
		mysql> exit;
		mysql> show variables like 'auto%'
	从服务器
		vim /etc/my.cnf
			auto_increment_increment=2
			auto_increment_offset=2(动态参数，不需要重启mysql)
		mysql> set global auto_increment_increment=2;
		mysql> set global auto_increment_offset=2;
		mysql> use mysql 
		mysql> select user,host from user
		...repl localhost
		mysql> show variables like '%read_only%';
		...read_only=on
		mysql> show master status \G
			File: mysql-bin.000003
			Position: 2542172
	主服务器
		mysql> change master to master_host='192.168.3.101',
			-> master_user='dba_repl',
			-> master_password='123456',
			-> master_log_file='mysql-bin.000003',
			-> master_log_log_pos=2542172;
		mysql> start slave;
		mysql> show slave status \G
			Slave_IO_Running:Yes
			Slave_SQL_Running:Yes
	安装keepalived
		主，主备都执行
		yum install keepalived -y
		[]# cd /etc/keepalived/
		vim keepalived.conf
		! Configuration File for keepalived
		global_defs {
			router_id mysql_ha
		}
		vrrp_script check_run {
			script "/etc/keepalived/check_mysql.sh"
			interval 2
		}
		vrrp_instance VI_1 {
			state BACKUP
			interface eth0
			virtual_router_id 200
			priority 99
			advert_int 1
			nopreemt
			authentication {
				auth_type PASS
				auth_pass 1200
			}
			track_script {
				check_run
			}
			virtual_ipaddress {
				192.168.3.99/24
			}
		}
	[keepalived]# chmod a+x check_mysql.sh
	[keepalived]# vim check_mysql.sh
		CHECK_TIME=3
		MYSQL_OK=1
	从服务器配置和主一致
	开始进程
	/etc/init.d/keepalived start
	ip addr
		inet 192.168.3.100/24
		inet 192.168.3.99/24(虚ip)
	模拟主服务器down机
		[]# /etc/init.d/mysqld stop
		[]# ip addr
			inet 192.168.3.100/24
			虚拟ip没了
		ps -ef | grep keepalived
			也没了
	主备
		[]# ip addr
			inet 192.168.3.101/24
			inet 192.168.3.99/24
			虚拟ip的切换完成，前端就用这个虚拟ip作为服务可以做到无缝切换
引入vip后的数据库架构
如何解决读负载和写负载两个问题
	1.读写分离
	2.master写操作，slave平均分摊读操作
	3.增加前端缓存服务器redis，memcache
如何进行读写分离
	由开发人员根据所执行的SQL类型连接不同的服务器
	优点：完全由开发人员控制，实现更加灵活
		  直接连接服务器，性能损耗小
	对于实时性要求比较高的数据，就不适合在从库上查询，虽然主从复制的时间只有几毫秒，但是并发量很大的话，也就时间很长了！（比如说库存超卖问题）
	缺点：增大工作量和程序代码复杂
		  人为控制，容易出错
	由数据库中间层完成读写分离
		DB Proxy
		优点：有中间层根据查询语法分析，自动完成读写分离，对程序透明
		缺点：查询效率有损耗（其实挺严重的）QPS会降低50%（基准测试）
			对延迟敏感的查询无法自动分给master查询
			但是如果无法分析就会给master，比如查询的存储过程也会给master

如何实现读服务器负载均衡方式
	数据库中间层---->实现读写分离，部分还可以对slave进行心跳检测，自动下线down机slave
	DNS轮询：对同一个域名下配置多个ip地址
	LVS/Haproxy：代理软件，不能对sql分析
	硬盘F5
使用LVS进行负载均衡
	优点：属于四层代理，只进行流量分发，不会对数据包内容解析，处理效率高，不能解决读写分离
		工作稳定，可进行高可用配置
		无流量，不会对主机的网络IO影响
	先前配置准备
		主DB ip：192.168.3.100
		主备DB ip：192.168.3.101
		SlaveDB ip：192.168.3.102
		keepalived vip：192.168.3.99
		lvs manager： 192.168.3.100/101
		lvs vip： 192.168.3.98
	主DB：
		[]# yum install -y ipvsadm.x86_64
	主备DB：
		[]# yum install -y ipvsadm.x86_64
	加载lvs模块，需要在所有参与lvs的服务器都进行这个操作
		主db 主备db SlaveDB
		[]# modprobe ip_vs
		[]# vim /etc/init.d/lvsrs
			VIP=192.168.3.98
			/sbin/ifconfig lo down
			/sbin/ifconfg lo up
			/sbin/route add -host $VIP dev lo:0
			...
		在主备DB和SlaveDB执行脚本
		[]# /etc/init.d/lvsrs start
		[]# ip addr 
			lo: 127.0.0.1/8
				192.168.3.98/32
		在keepalived配置
	主DB
		[]# mysql -uroot -p 
		mysql> grant all privileges on *.* to dba_monitor@'192.168.3.%' identified by
		123456;
	启动服务
	主DB 100
		[]# /etc/init.d/lvsdr start
			主上没有配置lo回路的虚拟ip
		[]# /etc/init.d/keepalived start
	主备DB 101
		[]# /etc/init.d/keepalived start
	主DB
		[]# ip addr
			.3.100
			.3.99
			.3.98
	SlaveDB
		[]# mysql -udba_monitor -p123456 -h192.168.3.98 -e"show variables like 'server_id'";
			server_id=3
	主DB
		[]# ipvsadm -L -n
			TCP 192.168.3.98
			 -> 192.168.3.101
			 -> 192.168.3.102
	模拟某一台从服务器down机
	SlaveDB
		[]# /etc/init.d/mysqld stop
	主DB
		[]# ipvsadm -L -n
			TCP 192.168.3.98
			 -> 192.168.3.101
	Keepalived+lvs总结
		如何利用keepalived进行写VIP的迁移
		如何利用lvs进行多个只读SlaveDB的负载均衡
		适用于开发人员认为判断读写分离的情况
==========================
MaxScale
	MaxScale节点 192.168.3.102
	MasterDB 192.168.3.100
	SlaveDB 192.168.3.101
	SlaveDB 192.168.3.192
安装
	注册下载
	安装支持库
	yum install 
	安装maxscale
	rpm -ivh 
使用
	masterDB
	mysql> create user scalemon@'192.168.3.%' identified by '123456';
	mysql> grant replication slave,replication client on *.* to scalemon@'192.168.3.%';
	mysql> create user scaleroute@'192.168.3.%' identified by '123456';
	mysql> grant select on mysql.* to scaleroute@'192.168.3.%';
maxscale密码是明文存储，不安全，需要加密
	MaxScale节点
	[]# maxpasswd /var/lib/mxscale/.secrets 123456
	[]# vim /etc/maxscale.cnf	
		threads=1(根据CPU)
		[server1]
		type=server
		address=192.168.3.100
		port=3306
		protocol=MYSQLBackend
		[server2]
		...102
		[server3]
		...103
		[MYSQL Monitor]
		type=monitor
		module=mysqlmon
		servers=server1,server2,server3
		user=scalemon
		passwd=加密后的密码
		monitor_interval=1000
		[Read-Write Service]
		type=service
		router=readwritesplit
		servers=server1,server2,server3
		user=scaleroute
		passwd=...
		max_slave_connections=100%
		max_slave_replication_lag=60
	[]# maxscale -f /etc/maxscale.cnf
	[]# ps -ef | grep maxscale
		..
	[]# maxadmin --user=admin --password=mariadb
	MaxScale> list servers
		Server   Address   Port   Connections   Status
		server1  ...100	   3306   0				Master,Running
		server2	 ...101	   ...	  0				Slave,Running
		server3  ...102    ...    0             Salve,Running
	MaxScale> show dbusers "Read-Write Service"
引入MaxScale后的架构
	单Master架构
如何解决压力问题
	MySQL复制无法缓解写压力
	利用缓存合并多次写为一次写
	对MasterDB进行拆分
		mc_orderdb
		mc_userdb
		mc_productdb
		1.按需求建立新的DB集群
		2.同步要拆分的DB到新的集群中
		3.迁移数据库账号到新的集群中
		4.修改前端应用的数据库连接
		5.删除原来的