最好不要在主库上进行数据库备份，大型活动前取消这类活动，因为会造成IO量激增，容易使数据库性能下降，引起堵塞。
影响数据库性能的因素：SQL查询速度，服务器硬件，网卡流量，磁盘IO

超高的QPS TPS
风险：效率低下的sql
QPS：每秒钟处理的查询量
	10描述ms处理1个sql
	1s处理100个sql
	QPS<=100
 	大多数的数据库问题都是因为慢查询导致的

大量的并发和超高的CPU使用率
风险：数据库连接数被占满，因CPU资源耗尽而出现宕机

磁盘IO
风险：磁盘IO性能下降（热数据不足）-->使用更快的磁盘设备ssd fashio

网卡流量
风险：网卡IO被占满
如何避免：减少从服务器数量，分级缓存，避免select *，分离业务和管理网络
==
大表
记录行数超千万行
表的数据文件超过10g
慢查询：
建立索引很慢，主从延迟
修改表结构需要长时间的锁表，数据库连接数被占满
如何处理大表
分库分表
	分表关键键的选择
	分表后的跨分区数据的查询和统计
	大多数不需要这种方式
大表的历史数据归档
	减少对前后端的业务影响
	难点：归档时间点选择，如何进行归档操作
什么是事务
	原子性：一个事务必须被视为一个不可分割的最小工作单元，要不全部成功要不回滚。
	一致性：一种状态转到另一种状态但是数据的完整性没有被破坏
	隔离性：要求一个事务在对数据库中的数据的修改，在未完成提交之前对其它的事务时不可见的
		未提交读READ UNCOMMIT脏读
		已提交读READ COMMIT 默认级别
		可重复读REPEATABLE READ 
		可串行化SERILIZEABLE 一般不用
什么是大事务
	定义：运行时间长，操作的数据比较多的
	风险：锁定大量的数据，造成大量的堵塞和锁超时，回滚所需要的时间很长，主从延迟
影响性能的地方
	服务器硬件
	服务器系统
	存储引擎选择
	数据库参数的配置影响最巨大
	SQL语句编写，数据库结构设计
CPU用64位
	mysql现在的版本也开始支持多核cpu了，而且web项目必须用多核cpu，因为并发量很大
存储引擎
	innodb会将索引和数据都缓存到内存中
磁盘的配置和选择
	使用传统机器磁盘
		1.移动磁头到磁盘表面上的正确位置
		2.等待磁盘旋转，使所需的数据在磁头之下
		3.等待磁盘旋转过去，所有所需的数据都被磁头读出
	使用raid增强传统机器硬盘的性能
		多个容量较小的磁盘组成一组容量更大的的磁盘，并提供数据冗余来保证数据的完整性
		RAID 0 磁盘串联，不具备数据冗余恢复能力
		RAID 1 磁盘镜像，写的时候会将数据镜像到另一个磁盘，但是如果有一个坏了，再重新镜像会占据大量的资源
		RAID 5	分布式奇偶校验，一般用在从服务器上
		RAID 10	
	使用固态存储ssd(闪存)和pcie卡
		相比机械盘固态磁盘由更好的随机读写性能，支持更好的并发，固态磁盘容易损坏（因为频繁的擦除）
		SSD
			使用SATA接口
			支持RAID技术
		PCI-E SSD
			无法使用SATA接口
			价格贵，性能好
			提高了IO性能但是会损失服务器的内存
		适用于存在大量随机IO
		解决单线程的负载的IO瓶颈
		如果只有一块固态，应该把它放在从服务器上，因为Slave是单线程写入
	使用网络存储SAN NAS
		两种外部文件存储设备加载到服务器上的方法
		适合场景
			数据库备份
		性能限制
			网络带宽
			网络质量
		建议
			采用高性能，高带宽的网络接口设备和交换机
			对多个网卡进行绑定，增强可用性和带宽
			尽可能的进行网络隔离，内外网隔离，业务和管理不相互影响
服务器硬件对性能的影响
CPU
	64位的cpu一定要工作在64位的系统下
	对于并发比较高的场景cpu的数量比频率重要
	对于cpu密集性场景和复杂sql则频率越高越好
CentOS系统参数优化
	vim /etc/sysctl.conf
	net.core.somaxconn=65535
	net.core.netdev_max_backlog=65535
	net.ipv4.tcp_max_syn_backlog=65535
	加快tcp的回收
	net.ipv4.tcp_fin_timeout=10
	net.ipv4.tcp_tw_reuse=1
	net.ipv4.tcp_tw_recycle=1
	TCP连接接收发送缓冲区大小
	net.core.wmem_default=87380
	net.core.wmem_max=16777216
	net.core.rmen_default=87380
	net.core.rmen_max=16777218
	失效连接占用tcp资源的数量
	net.ipv4.tcp_keepalive_time=120
	net.ipv4.tcp_keepalive_intvl=30
	net.ipv4.tcp_keepalive_probes=3
	内存相关参数
	kernel.shmmax=4294967295
	vm.swappiness=0
	增加资源限制
	vim /etc/security/limits.conf
	* soft nofile 65535
	* hard nofile 65535
	需要重启生效
MySQL
	临时表：在排序，分组操作，当数量超过一定大小后，由查询优化器建立的临时表
====================基于日志点的复制=============================
//主库
mysql>create user repl@'192.168.0.%' identified by '123456';
mysql>grant replication slave on *.* to repl@'192.168.0.%';

mysqldump --single-transaction --master-data --triggers --routines --all-databases >> all.sql

//备份到从库
scp all.sql root@192.168.0.102:/root

//从库
mysql -uroot -p < all.sql
mysql> change master to master_host='192.168.0.100',
	 > master_user='repl'
	 > master_password='123456'
	 > master_log_file=
	 > master_log_pos=(all.sql里面有)

mysql> show slave status \G
mysql> show processlist;
	IO thread
	SQL thread

//主库
mysql> show processlist \G
	Binlog dump

==================基于GTID的复制==============================
//主库
mysql> create user repl@'192.168.0.%' identified by'123456';
mysql> grant replication slave on *.* to repl@'192.168.0.%';

配置主服务器的binglog
vim /etc/my.cnf
bin_log = /usr/local/mysql/log/mysql-bin
server_id = 100
gtid_mode = on
enforce-gtid-consistency
	非法操作
	create table .. select ..
	create temporary table
	使用关联表和非事务表
log-slave-updates = on(5.7以后不需要配置)

配置从服务器
server_id = 101
relay_log = /usr/local/mysql/log/relay.log
gtid_mode = on
enforce-gtid-consistency
log-slave-updates=on
read-only = on(建议)
master_info_repository=table
relay_log_info_repository=table

//从服务器初始化
mysqldump --master-data=2 --single-transaction
xtarbackup --slave-info

//启动GTID复制
//从库
change master to master_host= '192.168.0.100',
	master_user='repl',
	master_password=..,
	master_auto_position=1

=============================mysql的拓扑结构==================================
	master-master
两个master操作的表分开，上海的用户用上海的，北京的用北京的，这样不会发生数据冲突
auto_increment_increment = 2
auto_increment_offset = 1, auto_increment_offset = 2, 两个master设置不一样

	mysql-slave
备库只读，做热备用
主备模式下的主主复制
	确保两台服务器的初始数据相同
	确保都开启了binlog并且有不同的server_id
	在两台服务器上开启log_slave_updates参数
	在初始化的备库上开启read-only = on

==============================mysql性能优化============
主从延迟
	1.主库写入binary log时间 ==> 控制主库的事务大小，分割事务
	2.二进制日志的传输时间	==> set binlog_row_image=minimal || mixed;
	3.默认只有一个sql线程，主上的并发修改会在从上变成串行，因为避免了一些锁消耗，所以性能还不错，但是
	当高并发的话，或者存在大事务的情况 ==> 多线程复制
	stop slave;
	set global slave_parallel_type='logical_clock';
	set global slave_parallel_workers=4;
	start slave;
===========================mysql复制常见的问题==================
数据损坏引起的主从复制错误
	1.主库或从库意外down机
	2.主库的意外重启导致二进制日志损坏。从库的重启导致中继日志的损坏，这个可以通过主库恢复，change master重新开IO传输，问题不大
	3.从库的read-only没开，数据不一致
	4.不唯一的server_id,从服务器的不唯一不容易查找，server_uuid存在数据目录中的auto.cnf文件中
	5.max_allow_packet设置引起的主从复制错误
	使用跳过二进制日志事件
	注入空事务的方式先恢复的中断的复制链路
	再使用其他方法对比主从服务器的数据

==================================高可用架构===========================
磁盘空间耗尽-->备份或者各种查询日志的激增，mysql如果不能记录二进制日志，无法处理新的请求，然后挂了

1.建立监控和报警系统
2.对备份数据进行恢复测试
3.正确的配置数据库==>slave 配成read-only
4.对不需要的数据进行归档和清理
5.增加系统的冗余-->避免单点故障
6.主从切换和故障转移

如何解决单点问题？
	MMM（Multi-Master Replication Manager）
============================MMM架构=========================================
1.配置主主复制及主从同步集群
2.安装主从节点所需要的包
3.安装MMM工具集
4.运行
5.测试

//创建repl账号，授权
mysqldump --single-transaction --master-data=2 --all-databases -uroot -p > all.sql
scp all.sql 192.168.0.101:/root
scp all.sql 192.168.0.102:/root

101: /root
mysql -uroot -p < all.sql
mysql> change master to master_host='192.168.0.100',
	-> master_user='repl'
	-> master_password=..,
	-> master_log_file=..,
	-> master_log_pos=..
mysql> start slave;
mysql> show slave status \G
mysql> show master status \G
	-> mysql-bin.000002
	-> position: 1412692

100:
mysql> change master to master_host='192.168.0.101',
	-> master_user='repl',
	-> master_password=..,
	-> master_log_file='mysql-bin.000002',
	-> master_log_pos=1412692
mysql> statr slave;
mysql> show slave status \G

102: /root
mysql -uroot -p < all.sql
mysql> change master to master_host='192.168.0.100',
	-> master_user='repl',
	-> master_password=..,
	-> master_log_file=(all.sql),
	-> master_log_position=(all.sql)
mysql> start slave;

100,101,102:
yum install mysql-mmm-agent.noarch -y

102:
yum -y install mysql-mmm*

100:
mysql> grant replication client on *.* to 'mmm_monitor'@'192.168.0.%' identified by '123456';
mysql> grant super, replication client, process on *.* to 'mmm_agent'@'192.168.0.%' identified by '123456';
mysql> grant replication ....(之前建好了一个复制账号了)
vim /etc/mysql_mmm/mmm_common.conf(通用配置，每个节点都应该一样)
<host>
	replication_user		repl
	replication_password 	123456
	agent_user				mmm_agent
	agent_password			123456
</host>

<host db1>
	ip 192.168.0.100
	mode master
	peer db2
</host>

<host db2>
	ip 192.168.0.101
	mode master
	peer db1
</host>

<host db3>
	ip 192.168.0.102
	mode slave
</host>

<role writer>
	hosts db1,db2
	ips 192.168.0.90
	mode 
</role>

<role reader>
	hosts db1,db2,db3
	ips 192.168.0.91,192.168.0.92,192.168.0.93
	mode balanced
</role>
:wq
scp mmm_common.conf root@192.168.0.102:/etc/mysql-mmm/
scp mmm_common.conf root@192.168.0.103:/etc/mysql-mmm/

100,101,102:
vim mmm_agent.conf
	this db0
..	db1
..	db2

102(slave + 监控): /etc/mysql-mmm/
vim mmm_mon.conf
	<monitor>
		ping_ips 192.168.0.100 192.168.0.101 192.168.0.102 (还可以配置网关) 
	</monitor>
	<host defult>
		monitor_user		mmm_monitor
		monitor_password	123456
	</host>
:wq

100,101,102:
/etc/init.d/mysql-mmm-agent start
102:
/etc/init.d/mysql-mmm-monitor start
mmm_control show
	master/online
	master/online
	slave/online 

检查虚拟ip是否配置成功
	ip addr

TEST
100:
/etc/init.d/mysqld stop
103:
mmm_control show
	master/hard_offline
mysql> show slave status
	master_host 192.168.101
[TEST_SUCCESS]
==========================MHA架构===========================
需要对集群所有节点都开启gtid_mode
mysql> show variables like 'gtid_mode';
	gtid_mode on
建立复制用户repl
	100,101,102

从库101,102:
mysql> change master to master_host='192.168.0.100',
	-> master_user='repl',
	-> master_password=123456,
	-> master_auto_position=1;
mysql> start slave;
mysql> show slave status \G


100:
生成SSH密钥
ssh-keygen
	save the key(/root/.ssh/id_rsa)
ssh-copy-id -i /root/.ssh/id_rsa '-p 22 root@192.168.0.100'
ssh root@192.168.0.100(SUCCESS 不需要输入密码)
生成SSH密钥
ssh-keygen
	save the key(/root/.ssh/id_rsa)
ssh-copy-id -i /root/.ssh/id_rsa '-p 22 root@192.168.0.101'
ssh root@192.168.0.101(SUCCESS 不需要输入密码)
生成SSH密钥
ssh-keygen
	save the key(/root/.ssh/id_rsa)
ssh-copy-id -i /root/.ssh/id_rsa '-p 22 root@192.168.0.102'
ssh root@192.168.0.102(SUCCESS 不需要输入密码)

//在102上下载了mha.node，mha.manage
scp mha4mysql-node-0.57.noarch.rpm root@192.168.0.100:/root
scp mha4mysql-node-0.57.noarch.rpm root@192.168.0.101:/root

100,101,102:
yum -y install perl-DBD-MYSQL ncftp perl-DBI.x86

100,101,102:/root,
rpm -ivh mha4mysql-node-0.57.noarch.rpm

102:
[]yum -y install perl-Config-Tiny.noarch perl-Time-HiRes.x86_64 Perl-Parallel-ForkManager
perl-Log-Dispatch-Perl perl-DBD-MySQL ncftp
[]rpm -ivh mha4mysql-manager..noarch.rpm

100,101,102:
mkdir -p /home/mysql_mha(当master挂后需要下载它的二进制文件，这就是存放的工作目录)

//mha配置只需要在manager节点配置就好
102:
mkdir -p /etc/mha
vim /etc/mha/mysql_mha.cnf
	[server default]
	user=mha
	password=123456
		{
			//配置user账号
			100:
			mysql> grant all privileges on *.* to mha@'192.168.0.%' identified by '123456';
			(因为目前是主从，所以101，102不需要配置了)
		}
	manager_workdir=/home/mysql_mha
	manager_log=/home/mysql_mha/manager.log
	remote_workdir=/home/mysql_mha
	ssh_user=root
	repl_user=repl
	repl_password=123456
	ping_interval=1(manager线程检查master的存活状态)
	master_binlog_dir=/home/mysql/sql_log(最好将所有可以参加master选举的binlog位置配置一样，方便新主上线)
		{
			//从master上查找binlog的位置
			100:
			mysql> show variables like'%log%';
			log_bin_basename 	/home/mysql/sql_log/mysql_bin
		}
	matser_ip_failover_script=/user/bin/master_ip_failover(让虚拟ip转到新master上，ip漂移的脚本)
	secondary_check_script=/usr/bin/masterha_secondary_check -s 192.168.0.101 -s 192.168.0.102 
	-s 192.168.0.100 (最好加上网关)
	[server1]
	hostname=192.168.0.100
	candidate_master=1
	[server2]
	hostname=192.168.0.101
	candidate_master=1
	[server3]
	hostname=192.168.0.102
	no_master=1

//TEST检测
102:
masterha_check_ssh --conf=/etc/mha/mysql_mha.cnf
masterha_check_repl --conf=/etc/mha/mysql_mha.cnf
nohup masterha_manager --conf=/etc/mha/mysql_mha.cnf &
ps -ef
	mha...

//mha配置虚拟master的虚拟ip
100：
ifconfig eth0:1 192.168.0.90/24
ip addr
	etho:
	192.168.0.100/24
	192.168.0.90/24

//TEST MHA架构的故障转移
100：
/etc/init.d/mysqld stop
ip addr
	etho:
	192.168.0.100/24
101:
ip addr
	etho:
	192.168.0.100/24
	192.168.0.90
102:
mysql> show slave status \G
=================================MaxScale=============================
//主库100：
mysql> create user scalemon@'192.168.0.%' identified by '123456';
mysql> grant replication salve,replication client on *.* to scalemon@'192.168.0.%';
mysql> create user maxscale@'192.168.0.%' identified by '123456';
mysql> grant select on mysql.* to maxscale@'192.168.0.%';

//102:
/home/tools/
wget ...maxscale
yum install libaio
rpm -ivh maxscale..rpm
maxkeys
	/var/lib/maxscale/ ...
maxpasswd /var/lib/maxscale/ 123456
vim /etc/maxscale.cnf
==server
	threads=4(8)
	[server1]
	type=server
	address=192.168.0.100
	port=3306
	protocol=MySQLBackend
	[server2]
	type=server
	address=192.168.0.101
	port=3306
	protocol=MySQLBackend
	[server3]
	type=server
	address=192.168.0.102
	port=3306
	protocol=MySQLBackend
==monitor
	[MySQL Monitor]
	type=monitor
	module=mysqlmon
	servers=server1,server2,server3
	user=scalemon
	passwd=123456
	monitor_interval=1000(ms)
==Read
	[Read-Only]
	不开启删除掉这个配置
==Read-Write
	[Read-Write Service]
	type=service
	router=readwritesplit
	servers=server1,server2,server3
	user=maxscale
	passwd=123456
	max_slave_connections=100%
	max_slave_replication_lag=60
==Read
	[Read-Only Listener]
	不开启，删除掉这个配置
==Read-Write
	[Read-Write Listener]
	..
	port=4006(因为这台机器同时运行了mysql，否则最好设置为3306)
//启动
maxscale --config=/ect/maxscale.cnf
ps -ef |grep maxscale
	maxscle
netstat -ntelp
	4006
	6603
//进入监控
maxadmin --user=admin --password=mariadbb
MaxScale> list servers
	server1		192.168.0.100		3306	slave,running
	..
MaxScale> show dbusers "Read-Write Service"
MaxScale> 

=========================Index优化===============================
1.索引列上不能使用表达式或者函数
select to_days(out_date)-to_days(current_date)<=30;
select out_date<=date_add(current_date,inteval 30 day);
2.前缀索引和索引列的选择性
在大字符列的地方可以选择前缀索引,但是会降低索引的唯一性，选择性和效率
create index index_name on table(col_name(n));
3.联合索引
如何选择索引列的顺序
	经常会被使用到的列优先（状态型的，可选择性差的不建议）
	选择性高的列优先
	宽度小的列优先
4.覆盖索引
	优点
		可以优化缓存，减少磁盘IO
		可以减少随机IO
		可以避免对innodb的主键索引的二次查询
	不可用情况
		存储引擎不支持（memory）
		查询中包含太多的列 select * ..
		使用%%的like查询（mysql只能提取值到内存，然后再过滤）
..............使用索引来进行优化查询.............
使用索引扫描来优化排序
	索引列的升降规则要和order by 子句的一致
	btree联合索引左边的列使用范围查找，右边的列就不会使用索引了
	select * from user where age>'10' order by user_id,user_name;(filesort)
删除重复冗余的索引
index(a) ,index(a,b)-----当只需要a索引时，index(a,b)同样生效
primary key(id), index(a,id)-----innodb二级索引会自动优化加上主键id，所以不需要加id
定期清理从不使用的索引
======================SQl优化查询（最关键）=================================
如何获取有问题的sql
	通过用户，测试反馈
	通过慢查询日志(记录所有符合条件的sql)
		存储大量日志需要的磁盘空间
		slow_query_log=on 
		slow_query_log_file(不指定默认保存在MySQL数据目录中)
		long_query_time(s) 阈值（默认10s，too long，一般设置为0.001）
		log_queries_not_using_indexes只要没使用索引的sql都会记录到慢查询日志中
		启动停止慢查询日志(/etc/my.cnf)
		因为是动态的数据，所有需要set global
		cd /usr/local/mysql/sql_log
			slow-mysql.log
		mysqldumps slow -s r -t 10 slow-mysql.log
		通过脚本来定时的开关
			常见的慢查询日志分析工具mysqldumpslow
			mysqldumps slow -s r -t 10 slow-mysql.log
	实时获取存在性能问题的sql
		select id,user,host,db,command,time,state,info from information_schema.processlist;
为什么sql查询会出现问题
	客户端发送sql请求给服务器
	服务器会现在查询缓存中检查是否命中该sql
		对大小写敏感的hash全值匹配
		查询缓存会加锁，对于一个读写频繁的sql不建议使用查询缓存
		query_cache_type =on off demand 
		query_cache_size
		query_cache_limit(可以存储的最大值，如果特定的缓存加上sql_no_cache提升效率)
		query_cache09_wlock_invalidate
		query_cache_min_res_unit
	服务器端进行sql解析，预处理，再由优化器生成对应的执行计划
	根据执行计划，调用存储引擎的api来查询数据，返回到mysql服务器层，如果需要还会在其缓存层过滤（%%的情况）
	将结果返回给客户端

优化not in
select customer_id,first_name,last_name,email
from customer
where customer_id
not in(select customer_id from payment)
--------->
select a.customer_id,a.first_name,a.last_name,a.email
from customer a 
left join payment b on a.customer_id=b.customer_id
where b.customer_id is null

使用汇总表优化查询
select count(*) from product_comment where product_id=99;*
--------->
create table product_comment_cnt(product_id int,cnt int);(凌晨进行统计汇总)
select sum(cnt) from(
	select cnt from product_comment_cnt where product_id=99;
	union all
	select count(*) from product_comment where product_id=99
	and timestr>date(now());
)a*
