由于公司要上redis cluster，但对于安全性要求较高，需要密码验证，此时codis2.x 并没有支持，3.0 支持，但3.0 各方面资料并不多，多方考虑，开始尝试redis cluster 。

但此时redis cluster 原生自带的redis-trib.rb 经过测试，我是发现不支持密码验证，为此对于集群维护就显得相当麻烦，而redis 原生命令却能很好的支持redis cluster密码认证，包括集群复制。

为此需一种自动管理节点数据迁移工具，所以利用perl编写自动化管理工具。

但不是很完善，希望能给大家一个借鉴。

#自动分片槽位
auto Resharding all slot to set master : 

./redis_cluster_data_move -t reshard -h host -p port

#自动迁移槽位
move slot:移动slot，此时槽位为空，也就是当cluster down 时，快速将16383槽位移走，不是涉及迁移数据，保证cluster 可用

./redis_cluster_data_move -t ms -h host -p port -d target_id-r 0-16383

#move data: 
移动槽位及数据

./redis_cluster_data_move -t md -h source_host:port-target_host2:port2 -s source_id -d target_id -r 0-16383

#删除节点
del redis node: 
在集群host：port删除 node_id 

./redis_cluster_data_move -t del -h host -p port -n node_id 

#add redis node:添加节点

在集群source_host:port 添加目标target_host:target_port

./redis_cluster_data_move -t add -h source_host:source_port-target_host:target_port 

#add redis slave node:将节点添加为从

将host:port 添加为node_id 的从节点

./redis_cluster_data_move -t add_slave -h host -p port -n node_id

#update redis slave to master :

将slave手动升级为master ，在升级时使用将host port 上级为master 

./redis_cluster_data_move -u up -h host -p port 


命令参数解释：

-t 任务类型

-h     主机

-p 端口

-d 节点id

-s 源节点id

-r 槽位范围

-n 节点id


机器环境：
    172.16.10.218 ：7000 
    172.16.10.205 ：7005

主要文件配置
masterauth "kevin"
requirepass "kevin"
cluster-enabled yes
cluster-config-file /opt/redis/etc/7000/nodes-7000.conf
cluster-node-timeout 5000
#cluster-slave-validity-factor 10
cluster-migration-barrier 2

在218 上启动redis cluster ，但是并没有slot：信息如下
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected
初始化slot分配所有的slot：
（会对当前节点上显示的所有master都要通知下slot信息）
[root@localhost ~]# ./redis_cluster_data_move -t reshard -h 172.16.10.218 -p 7000
 
auto Resharding : redis-cli -h 127.0.0.1 -p 7000 -a kevin CLUSTER SETSLOT 16378 NODE 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 : OK
 
auto Resharding : redis-cli -h 127.0.0.1 -p 7000 -a kevin CLUSTER SETSLOT 16379 NODE 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 : OK
 
auto Resharding : redis-cli -h 127.0.0.1 -p 7000 -a kevin CLUSTER SETSLOT 16380 NODE 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 : OK
 
auto Resharding : redis-cli -h 127.0.0.1 -p 7000 -a kevin CLUSTER SETSLOT 16381 NODE 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 : OK
 
auto Resharding : redis-cli -h 127.0.0.1 -p 7000 -a kevin CLUSTER SETSLOT 16382 NODE 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 : OK
 
auto Resharding : redis-cli -h 127.0.0.1 -p 7000 -a kevin CLUSTER SETSLOT 16383 NODE 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 : OK
集群状态
9688:M 28 Mar 23:41:48.133 * Background saving terminated with success
9688:M 28 Mar 23:43:01.722 # Cluster state changed: ok

查看集群信息已经分配
[root@localhost ~]# redis-cli -p 7000 -a kevin  CLUSTER nodes
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected 0-16383
在205 上启动新节点
redis-server /opt/redis/etc/7000/redis_7000.conf &

218 上添加新节点
[root@localhost ~]# ./redis_cluster_data_move.pl -t add -h 172.16.10.205 -p 7000 
redis-cli  -h 127.0.0.1  -p 7000 -a kevin cluster meet 172.16.10.205 7000 : OK
查看集群状态
[root@localhost ~]# redis-cli -p 7000 -a kevin  CLUSTER nodes
2404b44e4937c65c73cbccb778776e392619c776 172.16.10.205:7000 master - 0 1459180167113 2 connected #并没有分配slot
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected 0-16383
自动分配slot给205：7000 节点 
根据本机多个master 进行自动分配。
[root@localhost ~]# ./redis_cluster_data_move -t reshard -h 172.16.10.218 -p 7000
auto Resharding : redis-cli -h 172.16.10.205 -p 7000 -a kevin CLUSTER SETSLOT 81 NODE 2404b44e4937c65c73cbccb778776e392619c776 : OK
 
auto Resharding : redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER SETSLOT 81 NODE 2404b44e4937c65c73cbccb778776e392619c776 : OK
 
auto Resharding : redis-cli -h 172.16.10.205 -p 7000 -a kevin CLUSTER SETSLOT 82 NODE 2404b44e4937c65c73cbccb778776e392619c776 : OK
 
auto Resharding : redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER SETSLOT 82 NODE 2404b44e4937c65c73cbccb778776e392619c776 : OK
注意红色部分，显示对81槽位进行广播，在205 和218 上都进行广播，将81给node节点2404b44e4937c65c73cbccb778776e392619c776 ，节点是自行判断无需指定

查看集群信息：
[root@localhost ~]# redis-cli -p 7000 -a kevin  CLUSTER nodes
2404b44e4937c65c73cbccb778776e392619c776 172.16.10.205:7000 master - 0 1459180780937 2 connected 0-8191
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected 8192-16383

插入数据测试：
#!/bin/bash
number=100000
let i=1
while  [ $i -le $number ];do
    redis-cli -h 172.16.10.205 -c  -p 7000 -a kevin set $i test$i 
    ((i++))
done
迁移数据：
    先手动进行模拟测试：
连接205 进行节点导入数据
[root@localhost ~]# redis-cli -p 7000 -a kevin  CLUSTER nodes
2404b44e4937c65c73cbccb778776e392619c776 172.16.10.205:7000 master - 0 1459183013122 2 connected 0-8192
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected 8193-16383
将9000槽位从218 7000节点导入205：7000节点
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin CLUSTER SETSLOT 9842 IMPORTING 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32
OK
[root@localhost ~]# 
将9000槽位从205：7000节点迁出到218：7000节点
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER SETSLOT 9842 MIGRATING  2404b44e4937c65c73cbccb778776e392619c776
OK
[root@localhost ~]# 
此时状态9842已经有映射关系
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin CLUSTER nodes
2404b44e4937c65c73cbccb778776e392619c776 172.16.10.205:7000 myself,master - 0 0 2 connected 0-8192 [9000-<-5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32] [9842-<-5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32]
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 master - 0 1459183597046 0 connected 8193-16383
计算218 节点9000槽位的key
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER COUNTKEYSINSLOT 9842
(integer) 13
[root@localhost ~]# 
列出key值
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER GETKEYSINSLOT 9842 13
 1) "1"
 2) "143511"
 3) "146263"
 4) "152550"
 5) "157222"
 6) "161593"
 7) "17683"
 8) "186179"
 9) "21132"
10) "24640"
11) "30173"
12) "35601"
13) "85187"
[root@localhost ~]# 
迁移key
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin MIGRATE 172.16.10.205 7000 143511 0 1000
(error) ERR Target instance replied with error: NOAUTH Authentication required.
[root@localhost ~]# 
此时显示认证失败
解决办法：将密码重置为空
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin 
172.16.10.205:7000> config set requirepass ''
OK
172.16.10.205:7000> 
再次运行
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin MIGRATE 172.16.10.205 7000 143511 0 1000
OK
[root@localhost ~]# 
其他key同上进行迁移
获取143511key的值
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin get 143511
(error) MOVED 9842 172.16.10.218:7000
[root@localhost ~]# 
#说明集群还没有刷新此key对应的节点，在执行setslot 通知集群，此key槽位更新
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin CLUSTER SETSLOT 9842 NODE 2404b44e4937c65c73cbccb778776e392619c776
OK
[root@localhost ~]# 
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin get 143511
"test143511"
[root@localhost ~]# 
再次获取集群信息：
槽位及数据已经迁移

[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER nodes
2404b44e4937c65c73cbccb778776e392619c776 172.16.10.205:7000 master - 0 1459184382164 2 connected 0-8192 9000 9482 9842
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected 8193-8999 9001-9481 9483-9841 9843-16383

自动迁移218：7000 部分槽位及数据到 205：7000
[root@localhost ~]# ./redis_cluster_data_move -t md -h 172.16.10.218:7000-172.16.10.205:7000 -s 5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 -d 2404b44e4937c65c73cbccb778776e392619c776 -r 8193-8999
显示如下信息
。。。。。

move 79954 to 172.16.10.205:7000
echo "MIGRATE 172.16.10.205 7000 79954 0 1000"| redis-cli -h 172.16.10.218 -p 7000 -a kevinOK
 
move 80468 to 172.16.10.205:7000
echo "MIGRATE 172.16.10.205 7000 80468 0 1000"| redis-cli -h 172.16.10.218 -p 7000 -a kevinOK
 
move 91429 to 172.16.10.205:7000
echo "MIGRATE 172.16.10.205 7000 91429 0 1000"| redis-cli -h 172.16.10.218 -p 7000 -a kevinOK
在查看集群信息
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER nodes
2404b44e4937c65c73cbccb778776e392619c776 172.16.10.205:7000 master - 0 1459185086256 2 connected 0-9000 9482 9842
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected 9001-9481 9483-9841 9843-16383
[root@localhost ~]# 
随机找个key值计算槽位：

[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER KEYSLOT 80468
(integer) 8999
[root@localhost ~]# 
getkey值测试：已经move到205：7000节点，说明已经ok
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin get  80468
(error) MOVED 8999 172.16.10.205:7000
[root@localhost ~]# 
再到205：7000 节点读取已经ok
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin get  80468
"test80468"
[root@localhost ~]# 
最后别忘了给205 设置密码：
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 

172.16.10.205:7000> config set requirepass kevin
OK
172.16.10.205:7000> 
数据迁移已经ok

redis 管理工具
添加节点
[root@localhost ~]# redis-cli -h 172.16.10.218 -p 7000 -a kevin CLUSTER nodes
2404b44e4937c65c73cbccb778776e392619c776 172.16.10.205:7000 master - 0 1459187652232 2 connected 0-9000 9482 9842
5cf7ae3c2ad7de4115841e04ede03a0c3ce49e32 172.16.10.218:7000 myself,master - 0 0 0 connected 9001-9481 9483-9841 9843-16383
[root@localhost ~]# 

[root@localhost ~]# ./redis_cluster_data_move -t add -h 172.16.10.218 -p 7002
redis-cli  -h 127.0.0.1  -p 7000 -a kevin cluster meet 172.16.10.218 7002 : OK
 
[root@localhost ~]# ./redis_cluster_data_move -t add -h 172.16.10.205 -p 7002
redis-cli  -h 127.0.0.1  -p 7000 -a kevin cluster meet 172.16.10.205 7002 : OK
 
[root@localhost ~]# ./redis_cluster_data_move -t add -h 172.16.10.205 -p 7001
redis-cli  -h 127.0.0.1  -p 7000 -a kevin cluster meet 172.16.10.205 7001 : OK
 
[root@localhost ~]# ./redis_cluster_data_move -t add -h 172.16.10.205 -p 7000
redis-cli  -h 127.0.0.1  -p 7000 -a kevin cluster meet 172.16.10.205 7000 : OK
添加从节点
将205 每个端口都作为218对应端口的从（可根据自己进行定义）
[root@localhost ~]# ./redis_cluster_data_move -t add_slave -h 172.16.10.205 -p 7002 -n a1f93e3db92cbb384e1e67edfbd7d6032916ee52
redis-cli  -h 172.16.10.205  -p 7002 -a kevin cluster replicate a1f93e3db92cbb384e1e67edfbd7d6032916ee52 : OK
 
[root@localhost ~]# 
[root@localhost ~]# 
[root@localhost ~]# redis-cli -p 7000 -a kevin cluster nodes
bea712ec23b397b7f8c92b525c6e9af6367f5d22 172.16.10.205:7000 slave 29711b239305bd0cfdadaa3d17e37444be6ee705 0 1459244447444 5 connected
2cb57b0345d77e3bcb8ed6955808ef7e51aaf1ab 172.16.10.205:7002 slave a1f93e3db92cbb384e1e67edfbd7d6032916ee52 0 1459244446441 4 connected
e4d6e77ab462309680f22cedff0c1702f1bb1cfb 172.16.10.205:7001 slave 0035c65bf69cedd663ef21b5fbc9029a08ba8c34 0 1459244448445 1 connected
0035c65bf69cedd663ef21b5fbc9029a08ba8c34 172.16.10.218:7001 master - 0 1459244446943 1 connected
29711b239305bd0cfdadaa3d17e37444be6ee705 172.16.10.218:7000 myself,master - 0 0 2 connected
a1f93e3db92cbb384e1e67edfbd7d6032916ee52 172.16.10.218:7002 master - 0 1459244448445 4 connected
[root@localhost ~]# 

删除从节点
[root@localhost ~]# ./redis_cluster_data_move -t del -h 172.16.10.218 -p 7000 -n 2efd6be8decc12f19aaadbc704e1d88941327fd3
redis-cli  -h 172.16.10.218  -p 7000 -a kevin CLUSTER FORGET 2efd6be8decc12f19aaadbc704e1d88941327fd3 : OK
[root@localhost ~]# 
修改从节点
将218 7003 添加为节点29711b239305bd0cfdadaa3d17e37444be6ee705 从节点
[root@localhost ~]# ./redis_cluster_data_move -t add_slave -h 172.16.10.218 -p 7003 -n 29711b239305bd0cfdadaa3d17e37444be6ee705
redis-cli  -h 172.16.10.218  -p 7003 -a kevin cluster replicate 29711b239305bd0cfdadaa3d17e37444be6ee705 : OK
 
[root@localhost ~]#
将218 7003端口从节点提升为master
[root@localhost ~]# redis-cli -p 7000 -a kevin cluster nodes
bea712ec23b397b7f8c92b525c6e9af6367f5d22 172.16.10.205:7000 master,fail - 1459245775375 1459245772975 6 disconnected
e4d6e77ab462309680f22cedff0c1702f1bb1cfb 172.16.10.205:7001 slave 0035c65bf69cedd663ef21b5fbc9029a08ba8c34 0 1459248267228 1 connected
2cb57b0345d77e3bcb8ed6955808ef7e51aaf1ab 172.16.10.205:7002 slave a1f93e3db92cbb384e1e67edfbd7d6032916ee52 0 1459248268731 4 connected
a1f93e3db92cbb384e1e67edfbd7d6032916ee52 172.16.10.218:7002 master - 0 1459248267228 4 connected 10923-16383
0035c65bf69cedd663ef21b5fbc9029a08ba8c34 172.16.10.218:7001 master - 0 1459248268230 1 connected 0-5461
2efd6be8decc12f19aaadbc704e1d88941327fd3 172.16.10.218:7003 slave 29711b239305bd0cfdadaa3d17e37444be6ee705 0 1459248268731 7 connected
29711b239305bd0cfdadaa3d17e37444be6ee705 172.16.10.218:7000 myself,master - 0 0 7 connected 5462-10922
[root@localhost ~]# ./redis_cluster_data_move -t up -h 172.16.10.218 -p 7003
redis-cli  -h 172.16.10.218  -p 7003 -a kevin  CLUSTER FAILOVER  : OK
[root@localhost ~]# 
[root@localhost ~]# redis-cli -p 7000 -a kevin cluster nodes
bea712ec23b397b7f8c92b525c6e9af6367f5d22 172.16.10.205:7000 master,fail - 1459245775375 1459245772975 6 disconnected
e4d6e77ab462309680f22cedff0c1702f1bb1cfb 172.16.10.205:7001 slave 0035c65bf69cedd663ef21b5fbc9029a08ba8c34 0 1459248313820 1 connected
2cb57b0345d77e3bcb8ed6955808ef7e51aaf1ab 172.16.10.205:7002 slave a1f93e3db92cbb384e1e67edfbd7d6032916ee52 0 1459248313319 4 connected
a1f93e3db92cbb384e1e67edfbd7d6032916ee52 172.16.10.218:7002 master - 0 1459248313319 4 connected 10923-16383
0035c65bf69cedd663ef21b5fbc9029a08ba8c34 172.16.10.218:7001 master - 0 1459248314321 1 connected 0-5461
2efd6be8decc12f19aaadbc704e1d88941327fd3 172.16.10.218:7003 master - 0 1459248312818 8 connected 5462-10922


29711b239305bd0cfdadaa3d17e37444be6ee705 172.16.10.218:7000 myself,slave 2efd6be8decc12f19aaadbc704e1d88941327fd3 0 0 7 connected

集群信息
[root@localhost ~]# redis-cli -p 7000 -a kevin cluster nodes
bea712ec23b397b7f8c92b525c6e9af6367f5d22 172.16.10.205:7000 slave 29711b239305bd0cfdadaa3d17e37444be6ee705 0 1459244447444 5 connected
2cb57b0345d77e3bcb8ed6955808ef7e51aaf1ab 172.16.10.205:7002 slave a1f93e3db92cbb384e1e67edfbd7d6032916ee52 0 1459244446441 4 connected
e4d6e77ab462309680f22cedff0c1702f1bb1cfb 172.16.10.205:7001 slave 0035c65bf69cedd663ef21b5fbc9029a08ba8c34 0 1459244448445 1 connected
0035c65bf69cedd663ef21b5fbc9029a08ba8c34 172.16.10.218:7001 master - 0 1459244446943 1 connected
29711b239305bd0cfdadaa3d17e37444be6ee705 172.16.10.218:7000 myself,master - 0 0 2 connected
a1f93e3db92cbb384e1e67edfbd7d6032916ee52 172.16.10.218:7002 master - 0 1459244448445 4 connected

测试集群故障
1、分槽
./redis_cluster_data_move -t reshard -h 172.16.10.218 -p 7000
2、杀掉其中一个主
[root@localhost ~]# redis-cli -p 7000 -a kevin shutdown
3、查看集群状态
[root@localhost ~]# redis-cli -p 7001 -a kevin cluster nodes
a1f93e3db92cbb384e1e67edfbd7d6032916ee52 172.16.10.218:7002 master - 0 1459245337658 4 connected 10923-16383
bea712ec23b397b7f8c92b525c6e9af6367f5d22 172.16.10.205:7000 master - 0 1459245338660 6 connected 5462-10922
29711b239305bd0cfdadaa3d17e37444be6ee705 172.16.10.218:7000 master,fail - 1459245328638 1459245327636 2 disconnected
e4d6e77ab462309680f22cedff0c1702f1bb1cfb 172.16.10.205:7001 slave 0035c65bf69cedd663ef21b5fbc9029a08ba8c34 0 1459245337157 1 connected
2cb57b0345d77e3bcb8ed6955808ef7e51aaf1ab 172.16.10.205:7002 slave a1f93e3db92cbb384e1e67edfbd7d6032916ee52 0 1459245338660 4 connected
0035c65bf69cedd663ef21b5fbc9029a08ba8c34 172.16.10.218:7001 myself,master - 0 0 1 connected 0-5461
4、插入集群数据测试
脚本如下
[root@localhost ~]# cat in.sh 
#!/bin/bash
number=200000
let i=1
while  [ $i -le $number ];do
echo    ` redis-cli -h 172.16.10.205 -c  -p 7001 -a kevin set $i test$i `
    ((i++))
done
查看其中一个节点情况
[root@localhost ~]# redis-cli -p 7001 -a kevin cluster nodes|grep bea712ec23b397b7f8c92b525c6e9af6367f5d22
bea712ec23b397b7f8c92b525c6e9af6367f5d22 172.16.10.205:7000 master - 0 1459245642331 6 connected 5462-10922
29711b239305bd0cfdadaa3d17e37444be6ee705 172.16.10.218:7000 slave bea712ec23b397b7f8c92b525c6e9af6367f5d22 0 1459245643332 6 connected
运行插入数据脚本
sh in.sh
关掉master205：7000
[root@localhost ~]# redis-cli -h 172.16.10.205 -p 7000 -a kevin shutdown
查看in.sh 插入数据情况，会有一段时间205 7000 端口拒绝请求，集群不会down，一会会好，这取决于集群配置cluster-require-full-coverage no
查看切换后状态，218 已经为master，切换成功
[root@localhost ~]# redis-cli -p 7001 -a kevin cluster nodes|grep 7000
bea712ec23b397b7f8c92b525c6e9af6367f5d22 172.16.10.205:7000 master,fail - 1459245775389 1459245773489 6 disconnected
29711b239305bd0cfdadaa3d17e37444be6ee705 172.16.10.218:7000 master - 0 1459245813058 7 connected 5462-10922
[root@localhost ~]# 

