## 一、前期准备
### 1.1 测试机配置说明
因为没有比较合适的线上测试机，所以只能用自己之前的开发机(Deepin 15.11) 勉强测试下。

各指标如下：
- CPU   12核 Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
- 内存  16GB
- 磁盘  ST1000DM010 - hard drive - 1 TB

### 1.2 搭建TiDB集群
因为只有一台测试机，所以只能搭一个单机版的tidb集群了，

> 使用 TiUP cluster 在单机上模拟生产环境部署步骤
https://docs.pingcap.com/zh/tidb/stable/quick-start-with-tidb 

在自己电脑上使用shell连接测试机的时候，会遇到root用户权限的问题，所以得用root用户登录
> root用户ssh登录
https://blog.csdn.net/u013366098/article/details/50542418

### 1.3 小插曲
部署好之后，能用mysql连接到数据库
```
mysql --host 127.0.0.1 --port 4000 -u root
```
但是无法在自己电脑的浏览器通过 http://{grafana-ip}:3000 访问集群 Grafana 监控页面和 通过 http://{pd-ip}:2379/dashboard 访问集群 TiDB Dashboard 监控页面。

查看测试机上的服务后发现，因为这两个服务绑定端口不是主机Ip，而是127.0.0.1，不能被外部访问到。

所以按如下方式配置了nginx做转发，实现自己电脑
配置nginx转发127.0.0.1:3000端口：

 ```
 安装nginx后
 
 vim /etc/nginx/nginx.conf 

 http {

        。。。

        upstream tidb {
                server 127.0.0.1:3000;

        }

        server {
                listen       80;
                server_name  tidb.com;

                location / {
                        proxy_pass http://tidb;
                }
        }

        。。。
        
    }
        
检查NGINX配置
  /usr/sbin/nginx -t
重新加载并重启NGINX
  /usr/sbin/nginx -s reload
  
 mac  /etc/hosts 文件加上：
 10.234.11.200 tidb.com 
 ```
 之后在浏览器访问：tidb.com，出现如下报错
 ```
If you're seeing this Grafana has failed to load its application files

1. This could be caused by your reverse proxy settings.

2. If you host grafana under subpath make sure your grafana.ini root_url setting includes subpath

3. If you have a local dev build make sure you build frontend using: yarn start, yarn start:hot, or yarn build

4. Sometimes restarting grafana-server can help
```
看grafana.log有如下错误：
```
t=2020-08-21T17:11:22+0800 lvl=eror msg="Alert Rule Result Error" logger=alerting.evalContext ruleId=21 name="TiKV raft store CPU alert" error="Could not find datasource Data source not found" changing state to=alerting
t=2020-08-21T17:11:22+0800 lvl=info msg="New state change" logger=alerting.resultHandler alertId=21 newState=alerting prev state=unknown
```
发现是端口被占用了，把tidb cluster stop了，3000和9090端口的进程kill了

> tiup常用运维操作：
https://docs.pingcap.com/zh/tidb/stable/maintain-tidb-using-tiup

后面重启tiup cluster 之后，在本机127.0.0.1:3000地址打开了grafana，但是发现不是我需要想看到的监控界面，之后看文档发现我需要的是TiDB Dashboard ,而不是直接查看grafana

> TiDB Dashboard 的用法

> 部署 https://docs.pingcap.com/zh/tidb/stable/dashboard-ops-deploy

> 访问 https://docs.pingcap.com/zh/tidb/stable/dashboard-access

之后使用 IP:2379 在浏览器就能访问 TiDB Dashboard，用户是 root，不能改，**初始密码为空！！**

然后在如下界面点击右边的“查看监控”，就能查看我们想要的监控界面了。
![](https://img-blog.csdnimg.cn/20200823172207998.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


## 二、压测工具安装和熟悉
### 2.1 sysbench
#### 2.1.1 安装
```
apt-get install sysbench
```
 
#### 2.1.2 导入数据
 ```
sysbench oltp_update_non_index --config-file=config --thread=8 --tables=8 --table-size=10000 prepare
```
上面是视频中的语句，不太好用(可能是版本不一样)下面是网上Google的，可以用
```
sysbench --test=oltp --mysql-host=10.234.11.200 --mysql-port=4000 --mysql-db=test --oltp-table-size=500000 --mysql-user=root  prepare
```
  
#### 2.1.3 测试
```
sysbench --test=oltp --mysql-host=10.234.11.200 --mysql-port=4000 --mysql-db=test --oltp-table-size=500000 --mysql-user=root  run
```

### 2.2 go-ycsb
#### 2.2.1 安装
```
git clone https://github.com/pingcap/go-ycsb.git
cd go-sybc
make
```
因为测试机之前装过go，而且忘了装哪了，遇到点Go环境问题
> https://www.cnblogs.com/dairuiquan/p/12490871.html

make的时候还报如下错误：
```
build command-line-arguments: cannot load bufio: cannot find module providing package bufio
Makefile:21: recipe for target 'build' failed
```
经过多次调试，还是有问题，后面看了项目里的go.mod才发现需要go1.13版本，所以又把原来的go1.12版本卸载了，重新装了个go1.15版本的才make成功。

#### 2.2.2 导入数据
因为测试机性能不太行，所以很写入10000条数据也很慢
```
root@cfq-PC:~/tidb/src/github.com/pingcap/go-tpc#  ./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16
***************** properties *****************
"updateproportion"="0.5"
"recordcount"="10000"
"workload"="core"
"mysql.port"="4000"
"readproportion"="0.5"
"readallfields"="true"
"scanproportion"="0"
"mysql.host"="10.234.11.200"
"insertproportion"="0"
"requestdistribution"="uniform"
"operationcount"="1000"
"dotransactions"="false"
"threadcount"="16"
**********************************************
INSERT - Takes(s): 8.5, Count: 320, OPS: 37.7, Avg(us): 492824, Min(us): 175452, Max(us): 1778535, 99th(us): 1777000, 99.9th(us): 1779000, 99.99th(us): 1779000
INSERT - Takes(s): 18.5, Count: 797, OPS: 43.1, Avg(us): 400157, Min(us): 167019, Max(us): 1778535, 99th(us): 1777000, 99.9th(us): 1779000, 99.99th(us): 1779000
。。。
INSERT - Takes(s): 328.5, Count: 9932, OPS: 30.2, Avg(us): 530885, Min(us): 167019, Max(us): 1778535, 99th(us): 1161000, 99.9th(us): 1774000, 99.99th(us): 1779000
INSERT - Takes(s): 338.5, Count: 10000, OPS: 29.5, Avg(us): 532151, Min(us): 167019, Max(us): 1778535, 99th(us): 1161000, 99.9th(us): 1772000, 99.99th(us): 1778000
INSERT - Takes(s): 348.5, Count: 10000, OPS: 28.7, Avg(us): 532151, Min(us): 167019, Max(us): 1778535, 99th(us): 1161000, 99.9th(us): 1772000, 99.99th(us): 1778000
```
#### 2.2.3 测试
```
root@cfq-PC:~/tidb/src/github.com/pingcap/go-tpc# ./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16
***************** properties *****************
"scanproportion"="0"
"insertproportion"="0"
"readproportion"="0.5"
"readallfields"="true"
"updateproportion"="0.5"
"recordcount"="1000"
"threadcount"="16"
"operationcount"="10000"
"dotransactions"="true"
"requestdistribution"="uniform"
"mysql.host"="10.234.11.200"
"mysql.port"="4000"
"workload"="core"
**********************************************
READ   - Takes(s): 10.0, Count: 356, OPS: 35.6, Avg(us): 1191, Min(us): 286, Max(us): 225804, 99th(us): 3000, 99.9th(us): 226000, 99.99th(us): 226000
UPDATE - Takes(s): 9.4, Count: 285, OPS: 30.2, Avg(us): 556966, Min(us): 344625, Max(us): 1241742, 99th(us): 851000, 99.9th(us): 1242000, 99.99th(us): 1242000
READ   - Takes(s): 20.0, Count: 650, OPS: 32.5, Avg(us): 1277, Min(us): 286, Max(us): 225804, 99th(us): 3000, 99.9th(us): 226000, 99.99th(us): 226000
UPDATE - Takes(s): 19.4, Count: 568, OPS: 29.2, Avg(us): 554675, Min(us): 323431, Max(us): 1611308, 99th(us): 851000, 99.9th(us): 1612000, 99.99th(us): 1612000
。。。
READ   - Takes(s): 220.0, Count: 5040, OPS: 22.9, Avg(us): 1801, Min(us): 274, Max(us): 467820, 99th(us): 3000, 99.9th(us): 376000, 99.99th(us): 468000
UPDATE - Takes(s): 219.4, Count: 4930, OPS: 22.5, Avg(us): 673051, Min(us): 291718, Max(us): 2849054, 99th(us): 1294000, 99.9th(us): 2698000, 99.99th(us): 2850000
Run finished, takes 3m44.783754219s
READ   - Takes(s): 224.8, Count: 5054, OPS: 22.5, Avg(us): 1797, Min(us): 274, Max(us): 467820, 99th(us): 3000, 99.9th(us): 376000, 99.99th(us): 468000
UPDATE - Takes(s): 224.2, Count: 4944, OPS: 22.1, Avg(us): 672300, Min(us): 283466, Max(us): 2849054, 99th(us): 1294000, 99.9th(us): 2698000, 99.99th(us): 2850000
```
### 2.3 go-tpc
##### 2.3.1 安装
```
git clone https://github.com/pingcap/go-tpc.git
make build
```
#### 2.3.2 tpc-C
导入数据
```
root@cfq-PC:~/tidb/src/github.com/pingcap/go-tpc# ./bin/go-tpc tpcc -H 10.234.11.200 -P 4000 -D tpcc --warehouses 4 prepare
creating table warehouse
creating table district
creating table customer
creating table history
creating table new_order
creating table orders
creating table order_line
creating table stock
creating table item
load to item
load to warehouse in warehouse 1
load to stock in warehouse 1
load to district in warehouse 1
load to warehouse in warehouse 2
load to stock in warehouse 2
load to district in warehouse 2
load to warehouse in warehouse 3
load to stock in warehouse 3
load to district in warehouse 3
load to warehouse in warehouse 4
load to stock in warehouse 4
load to district in warehouse 4
load to customer in warehouse 1 district 1
load to history in warehouse 1 district 1
load to orders in warehouse 1 district 1
load to new_order in warehouse 1 district 1
。。。
begin to check warehouse 4 at condition 3.3.2.9
begin to check warehouse 4 at condition 3.3.2.10
Finished
```
运行测试
```
root@cfq-PC:~/tidb/src/github.com/pingcap/go-tpc# ./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 1m  --threads 2
[Current] NEW_ORDER - Takes(s): 4.6, Count: 1, TPM: 12.9, Sum(ms): 3749, Avg(ms): 3749, 90th(ms): 4000, 99th(ms): 4000, 99.9th(ms): 4000
。。。
[Current] NEW_ORDER - Takes(s): 8.6, Count: 5, TPM: 35.1, Sum(ms): 15323, Avg(ms): 3064, 90th(ms): 4000, 99th(ms): 4000, 99.9th(ms): 4000
[Current] PAYMENT - Takes(s): 9.6, Count: 2, TPM: 12.5, Sum(ms): 6045, Avg(ms): 3022, 90th(ms): 4000, 99th(ms): 4000, 99.9th(ms): 4000
Finished
[Summary] NEW_ORDER - Takes(s): 55.2, Count: 16, TPM: 17.4, Sum(ms): 56860, Avg(ms): 3553, 90th(ms): 8000, 99th(ms): 8000, 99.9th(ms): 8000
[Summary] ORDER_STATUS - Takes(s): 54.8, Count: 1, TPM: 1.1, Sum(ms): 1294, Avg(ms): 1294, 90th(ms): 1500, 99th(ms): 1500, 99.9th(ms): 1500
[Summary] PAYMENT - Takes(s): 58.9, Count: 22, TPM: 22.4, Sum(ms): 60763, Avg(ms): 2761, 90th(ms): 4000, 99th(ms): 8000, 99.9th(ms): 8000
[Summary] PAYMENT_ERR - Takes(s): 58.9, Count: 1, TPM: 1.0, Sum(ms): 174, Avg(ms): 174, 90th(ms): 192, 99th(ms): 192, 99.9th(ms): 192
[Summary] STOCK_LEVEL - Takes(s): 21.2, Count: 1, TPM: 2.8, Sum(ms): 1404, Avg(ms): 1404, 90th(ms): 1500, 99th(ms): 1500, 99.9th(ms): 1500
tpmC: 17.4
```
看到最后```tpmC: 17.4```，说明一分钟就只能成交 17.4笔订单，性能真是令人捉急。


注：tpcc的warehouse数量越少，意味着事务冲突越严重，因为warehouse的数量很少的话，很多客户需要去同一个warehouse那里购买物品，就会造成比较严重的事务冲突。

**warehouse少的话，应减少测试的线程数**


#### 2.3.3 tpc-H

- sf=2 表示大概会有2GB的数据量
- tpc-h 测试会占用大量的内存，因此不建议把sf设置的过大；机器内存是60GB的话，sf最好不要超过30

导入数据
```
root@cfq-PC:~/tidb/src/github.com/pingcap/go-tpc# ./bin/go-tpc tpch prepare -H 10.234.11.200 -P 4000 -D tpch --sf 2 --analyze
creating nation
creating region
creating part
creating supplier
creating partsupp
creating customer
creating orders
creating lineitem
generating nation table
generate nation table done
generating region table
generate region table done
generating customers table
generate customers table done
generating suppliers table
generate suppliers table done
generating part/partsupplier tables
```
运行测试
```
 ./bin/go-tpc tpch run  -H 10.234.11.200 -P 4000 -D tpch --sf 2 
[Current] Q1: 2.12s
[Current] Q2: 14.45s
[Current] Q3: 3.29s
[Current] Q4: 0.08s
[Current] Q5: 0.43s
[Current] Q6: 0.30s
[Current] Q7: 0.29s
[Current] Q8: 0.27s
[Current] Q10: 0.33s
[Current] Q11: 0.91s
[Current] Q12: 0.34s
[Current] Q13: 0.24s
[Current] Q14: 0.21s
[Current] Q9: 11.21s
。。。
^C
Got signal [interrupt] to exit.
Finished
[Summary] Q1: 1.31s
[Summary] Q10: 0.33s
[Summary] Q11: 0.90s
[Summary] Q12: 0.30s
[Summary] Q13: 0.25s
[Summary] Q14: 0.21s
[Summary] Q15: 7.41s
[Summary] Q16: 0.79s
[Summary] Q17: 1.11s
[Summary] Q18: 0.37s
[Summary] Q19: 0.40s
[Summary] Q2: 7.71s
[Summary] Q20: 0.24s
[Summary] Q21: 0.33s
[Summary] Q22: 0.51s
[Summary] Q3: 1.77s
[Summary] Q4: 0.08s
[Summary] Q5: 0.44s
[Summary] Q6: 0.25s
[Summary] Q7: 0.29s
[Summary] Q8: 0.25s
[Summary] Q9: 6.23s
```

#### 2.3.4 补充
- TPC-C（大约1992年）模拟了一个'老派'OLTP应用程序，它看起来像一个批发分销商，有少量仓库，库存服务于大量零售点。在这种情况下，它衡量“每分钟交易次数”（tpmC）。它假设DRAM非常稀缺的老式IT架构，因此它在很大程度上依赖于磁盘IO。
- TPC-E是一个现代的OLTP应用程序，它模拟股票经纪并使用由股票价格波动驱动的更复杂的模拟世界，并模拟客户下市场订单，限价订单和限价订单的混乱“外部世界” 。 TPC-E采用现代IT架构，其中DRAM和计算资源更加丰富，因此它不依赖于存储性能。
- TPC-H是一个OLAP工作负载，用于衡量“数据仓库”环境中的查询分析。

简而言之，TPC-E适用于OLTP，TPC-H适用于OLAP，TPC-C基本上是过时的。


## 三、测试并分析性能监控
因为之前在安装和测试压测工具的时候已经导入了一些数据到我们的数据库中，所以现在不需要导入数据，直接进行测试就好了。

### 3.1 sysbench
运行如下命令
```
sysbench --test=oltp --mysql-host=10.234.11.200 --mysql-port=4000 --mysql-db=test --oltp-table-size=500000 --mysql-user=root  run
```
得到的各项监控如下
![TiDB Query Summary 中的 qps 与 duration](https://img-blog.csdnimg.cn/20200823174802291.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![tikv 各 server 的 CPU 以及 QPS 指标](https://img-blog.csdnimg.cn/20200823174856168.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![grpc 的 qps 以及 duration](https://img-blog.csdnimg.cn/2020082317493471.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

### 3.2 go-ycsb
依次运行如下命令
```
./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16

./bin/go-ycsb run mysql -P workloads/workloada -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 32

./bin/go-ycsb run mysql -P workloads/workloadb -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16

./bin/go-ycsb run mysql -P workloads/workloadc -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16
Run finished, takes 424.77755ms
READ   - Takes(s): 0.4, Count: 9999, OPS: 23634.1, Avg(us): 668, Min(us): 309, Max(us): 4164, 99th(us): 3000, 99.9th(us): 4000, 99.99th(us): 5000

./bin/go-ycsb run mysql -P workloads/workloadd -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16
Run finished, takes 440.341529ms
INSERT - Takes(s): 0.4, Count: 489, OPS: 1145.7, Avg(us): 502, Min(us): 246, Max(us): 3113, 99th(us): 3000, 99.9th(us): 4000, 99.99th(us): 4000
READ   - Takes(s): 0.4, Count: 9504, OPS: 22155.5, Avg(us): 692, Min(us): 306, Max(us): 11580, 99th(us): 3000, 99.9th(us): 5000, 99.99th(us): 12000

./bin/go-ycsb run mysql -P workloads/workloade -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16
Run finished, takes 893.273143ms
INSERT - Takes(s): 0.9, Count: 478, OPS: 541.4, Avg(us): 788, Min(us): 313, Max(us): 5089, 99th(us): 3000, 99.9th(us): 6000, 99.99th(us): 6000
SCAN   - Takes(s): 0.9, Count: 9522, OPS: 10688.7, Avg(us): 1436, Min(us): 528, Max(us): 10522, 99th(us): 5000, 99.9th(us): 7000, 99.99th(us): 11000

./bin/go-ycsb run mysql -P workloads/workloadf -p operationcount=10000 -p mysql.host=10.234.11.200 -p mysql.port=4000 --threads 16
```
（其中workloadc workloadd workloade 耗时都比较长，所以输出直接记下来了）

得到的各项监控如下
![TiDB Query Summary 中的 qps 与 duration](https://img-blog.csdnimg.cn/20200823224106839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![tikv 各 server 的 CPU 以及 QPS 指标](https://img-blog.csdnimg.cn/20200823224139781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![grpc 的 qps 以及 duration](https://img-blog.csdnimg.cn/20200823224209763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

其中QPS打的最高的应该是 workloadd

### 3.3 go-TPC
#### 3.3.1 go-tpcc
依次运行如下命令
```
 (时间点21:32)
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 1m  --threads 2
。。。
[Summary] NEW_ORDER - Takes(s): 56.9, Count: 15, TPM: 15.8, Sum(ms): 65943, Avg(ms): 4396, 90th(ms): 8000, 99th(ms): 8000, 99.9th(ms): 8000
[Summary] NEW_ORDER_ERR - Takes(s): 56.9, Count: 1, TPM: 1.1, Sum(ms): 423, Avg(ms): 423, 90th(ms): 512, 99th(ms): 512, 99.9th(ms): 512
[Summary] PAYMENT - Takes(s): 58.7, Count: 19, TPM: 19.4, Sum(ms): 46568, Avg(ms): 2450, 90th(ms): 4000, 99th(ms): 4000, 99.9th(ms): 4000
[Summary] PAYMENT_ERR - Takes(s): 58.7, Count: 1, TPM: 1.0, Sum(ms): 1517, Avg(ms): 1517, 90th(ms): 2000, 99th(ms): 2000, 99.9th(ms): 2000
[Summary] STOCK_LEVEL - Takes(s): 52.1, Count: 2, TPM: 2.3, Sum(ms): 5489, Avg(ms): 2744, 90th(ms): 8000, 99th(ms): 8000, 99.9th(ms): 8000
tpmC: 15.8

 (时间点21:39)
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 3m  --threads 4
。。。
[Summary] DELIVERY - Takes(s): 121.2, Count: 3, TPM: 1.5, Sum(ms): 50361, Avg(ms): 16787, 90th(ms): 16000, 99th(ms): 16000, 99.9th(ms): 16000
[Summary] DELIVERY_ERR - Takes(s): 121.2, Count: 1, TPM: 0.5, Sum(ms): 9875, Avg(ms): 9875, 90th(ms): 16000, 99th(ms): 16000, 99.9th(ms): 16000
[Summary] NEW_ORDER - Takes(s): 171.5, Count: 72, TPM: 25.2, Sum(ms): 339300, Avg(ms): 4712, 90th(ms): 8000, 99th(ms): 16000, 99.9th(ms): 16000
[Summary] NEW_ORDER_ERR - Takes(s): 171.5, Count: 1, TPM: 0.3, Sum(ms): 28, Avg(ms): 28, 90th(ms): 32, 99th(ms): 32, 99.9th(ms): 32
[Summary] ORDER_STATUS - Takes(s): 149.5, Count: 6, TPM: 2.4, Sum(ms): 2304, Avg(ms): 384, 90th(ms): 1000, 99th(ms): 1000, 99.9th(ms): 1000
[Summary] PAYMENT - Takes(s): 176.6, Count: 72, TPM: 24.5, Sum(ms): 303249, Avg(ms): 4211, 90th(ms): 8000, 99th(ms): 16000, 99.9th(ms): 16000
[Summary] PAYMENT_ERR - Takes(s): 176.6, Count: 1, TPM: 0.3, Sum(ms): 270, Avg(ms): 270, 90th(ms): 512, 99th(ms): 512, 99.9th(ms): 512
[Summary] STOCK_LEVEL - Takes(s): 177.0, Count: 5, TPM: 1.7, Sum(ms): 15053, Avg(ms): 3010, 90th(ms): 8000, 99th(ms): 8000, 99.9th(ms): 8000
tpmC: 25.2

 (时间点21:44)
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 1m  --threads 8
tpmC: 37.9


 (时间点21:46)
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 1m  --threads 16
tpmC: 62.2


 (时间点21:48)
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 1m  --threads 32
tpmC: 84.9

 (时间点21:50)
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 1m  --threads 128
tpmC: 87.4

 (时间点21:56)
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 1m  --threads 64
tpmC: 79.2
```

得到的各项监控如下
![TiDB Query Summary 中的 qps 与 duration](https://img-blog.csdnimg.cn/20200824220713227.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![tikv 各 server 的 CPU 以及 QPS 指标](https://img-blog.csdnimg.cn/20200824220748553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![grpc 的 qps 以及 duration](https://img-blog.csdnimg.cn/20200824220814352.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

分析如下
- CPU使用率跟threads成正比
- 耗时跟跟threads没有太大关系
- QPS在 ```--threads 128```的时候突增了，但是测下来之后看到最后tpmC(每分钟的成交量)并没有上涨很多，感觉到了瓶颈
- 又调到 ```--threads 64```的时候，QPS和流量都降了很多，但是tpmC甚至比 ```--threads 32```的时候还低
- 感觉这台测试机最理想的threads数是32，此时cpu和qps都不算高，但是tpmC比较理想


#### 3.3.2 go-tpch
依次运行如下命令
```
 (时间点22:10~22:13)
./bin/go-tpc tpch run  -H 10.234.11.200 -P 4000 -D tpch --sf 2
Finished
[Summary] Q1: 0.57s
[Summary] Q10: 0.37s
[Summary] Q11: 0.97s
[Summary] Q12: 0.25s
[Summary] Q13: 0.26s
[Summary] Q14: 0.21s
[Summary] Q15: 2.99s
[Summary] Q16: 0.56s
[Summary] Q17: 0.63s
[Summary] Q18: 0.33s
[Summary] Q19: 0.34s
[Summary] Q2: 1.08s
[Summary] Q20: 0.22s
[Summary] Q21: 0.32s
[Summary] Q22: 0.33s
[Summary] Q3: 0.26s
[Summary] Q4: 0.08s
[Summary] Q5: 2.22s
[Summary] Q6: 0.20s
[Summary] Q7: 0.34s
[Summary] Q8: 0.22s
[Summary] Q9: 0.56s


(时间点22:16~22:19)
./bin/go-tpc tpch run  -H 10.234.11.200 -P 4000 -D tpch --sf 4
Finished
[Summary] Q1: 0.57s
[Summary] Q10: 0.35s
[Summary] Q11: 0.92s
[Summary] Q12: 0.25s
[Summary] Q13: 0.23s
[Summary] Q14: 0.27s
[Summary] Q15: 3.22s
[Summary] Q16: 0.63s
[Summary] Q17: 0.62s
[Summary] Q18: 0.34s
[Summary] Q19: 0.35s
[Summary] Q2: 1.00s
[Summary] Q20: 0.29s
[Summary] Q21: 0.32s
[Summary] Q22: 0.33s
[Summary] Q3: 0.26s
[Summary] Q4: 0.08s
[Summary] Q5: 2.24s
[Summary] Q6: 0.20s
[Summary] Q7: 0.35s
[Summary] Q8: 0.22s
[Summary] Q9: 0.62s


(时间点22:23~22:26)
./bin/go-tpc tpch run  -H 10.234.11.200 -P 4000 -D tpch --sf 8
Finished
[Summary] Q1: 0.57s
[Summary] Q10: 0.33s
[Summary] Q11: 0.93s
[Summary] Q12: 0.25s
[Summary] Q13: 0.23s
[Summary] Q14: 0.21s
[Summary] Q15: 3.12s
[Summary] Q16: 0.57s
[Summary] Q17: 0.63s
[Summary] Q18: 0.33s
[Summary] Q19: 0.34s
[Summary] Q2: 0.95s
[Summary] Q20: 0.23s
[Summary] Q21: 0.31s
[Summary] Q22: 0.33s
[Summary] Q3: 0.26s
[Summary] Q4: 0.08s
[Summary] Q5: 2.28s
[Summary] Q6: 0.20s
[Summary] Q7: 0.28s
[Summary] Q8: 0.22s
[Summary] Q9: 0.66s
```

得到的各项监控如下
![TiDB Query Summary 中的 qps 与 duration](https://img-blog.csdnimg.cn/20200824223241226.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![tikv 各 server 的 CPU 以及 QPS 指标](https://img-blog.csdnimg.cn/20200824223306551.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![grpc 的 qps 以及 duration](https://img-blog.csdnimg.cn/2020082422332384.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

分析如下
- sf=2和sf=4、sf=8在时延、QPS、CPU使用率和Tikv内存使用上都没有多大差别
