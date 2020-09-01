## 一、课前准备
### 启动Tidb集群
因为Lesson-2已经通过tiup工具部署好了一个tidb集群，所以现在只需要执行如下命令就可以直接启动tidb集群了
```
tiup cluster start cfq-tidb-cluster

（测试集群是否正常启动）
mysql -h10.234.11.200 -P4000 -uroot
```
第一次启动的时候tidb没有起来，又执行了一遍stop和start，然后就好了。

## 二、课程梳理
### 2.1 CPU profile
#### 2.1.1 通过tidb dashboard进行分析
> https://docs.pingcap.com/zh/tidb/stable/dashboard-profiling

在浏览器中访问``` http://10.234.11.200:2379/dashboard/#/overview ```

#### 2.1.2 通过debug.zip分析tidb性能
通过如下命令分析60s，并把分析 tidb 性能问题所需的多个 profile 文件打包生成一个 zip 文件
```
curl http://{TiDBIP}:10080/debug/zip?seconds=60 --output debug.zip
```
debug.zip 解压后可以看到以下几个文件:

- version tidb-server 版本
- profile
CPU profile 文件
- heap
heap profile 文件
- goroutine
所有 goroutine 的栈信息
- mutex
mutex profile 文件
- config
tidb server 的配置文件

通过如下命令在 web 界面上查看
```
go tool pprof -http=:8080 debug/profile
```

> 火焰图是从线程或者说是函数调用栈的角度去展示哪些地方消耗了CPU，也可以很好的指导我们去阅读代码，因为其展示了函数的调用关系。

#### 2.1.3 通过perf命令分析Tikv CPU 性能
```
git clone https://github.com/pingcap/tidb-inspect-tools
cd tracing_tools/perf
sudo sh ./cpu_tikv.sh $tikv-pid
sudo sh ./dot_tikv.sh $tikv-pid
```
在命令之前，需要执行```apt install linux-perf```安装perf；但是执行反馈找不到命令，参考如下文章解决
> https://blog.csdn.net/sinat_35866463/article/details/103476997

之后修改cpu_tikv.sh脚本，但还是因为deepin和centos系统的一些文件系统结构不同，不能执行，所以就不考虑用这个分析了。

### 2.2 IO
#### 2.2.1  Grafana: Disk-Performance

#### 2.2.2  iostat
iostat -x 1 -m
```
○ Device: rrqm/s wrqm/s r/s w/s rMB/s wMB/s avgrq-sz avgqu-sz await r_await w_await svctm %util
○ nvme0n1 0.00 0.00 0.00 5.00 0.00 0.02 8.00 0.00 0.00 0.00 0.00 0.00 0.00
○ sda 0.00 15.00 0.00 30.00 0.00 0.20 13.60 0.00 0.07 0.00 0.07 0.07 0.20
○ sdb 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00
```
- await:平均每次设备 I/O 操作的等待时间 (毫秒)
- r_await:读操作等待 时间
    - 如果该值比较高(比如超过 5ms 级别)说明磁盘读压力比较大
- w_await:写操作等待 时间
    - 如果该值比较高(比如超过 5ms 级别)说明磁盘写压力比较大 
- 可以看 io size、io 连续性、读写瞬时流量、读写分别的 iops
 

#### 2.2.3  iotop
- iostat 是从盘的角度出发来看 IO
- iotop 是从进程角度看 IO
    - iotop 用于看各个线程的 io 累积量，有没有超出预期，顺便作为 fsync 不足的佐证(jbd2流量超 大)
    - sudo iotop -o
        - 只显示有IO输出的进程。
```
Total DISK READ :       0.00 B/s | Total DISK WRITE :     141.13 K/s
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:     144.94 K/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND                                                           
  368 be/3 root        0.00 B/s   95.36 K/s  0.00 % 15.02 % [jbd2/sda3-8]
 5016 be/4 tidb        0.00 B/s   15.26 K/s  0.00 %  2.19 % bin/pd-server --name=pd-10.234.11.~le=/tidb-deploy/pd-2379/log/pd.log
11435 be/4 tidb        0.00 B/s    3.81 K/s  0.00 %  1.37 % bin/pd-server --name=pd-10.234.11.~le=/tidb-deploy/pd-2379/log/pd.log
15073 be/4 tidb        0.00 B/s   26.70 K/s  0.00 %  0.00 % bin/prometheus/prometheus --config~-9090 --storage.tsdb.retention=30d
12305 be/4 tidb        0.00 B/s    0.00 B/s  0.00 %  0.00 % bin/tikv-server --addr 0.0.0.0:201~-20161/log/tikv.log [rocksdb:low4]
12311 be/4 tidb        0.00 B/s    0.00 B/s  0.00 %  0.00 % bin/tikv-server --addr 0.0.0.0:201~20160/log/tikv.log [rocksdb:high1]
   24 be/0 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [kworker/2:0H]
   25 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [cpuhp/3]
   26 rt/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [watchdog/3]
```

 #### 2.2.4 iosnoop
 >  http://www.brendangregg.com/blog/2014-07-16/iosnoop-for-linux.html
 
 把如下代码copy到本地，然后```chmod +x```，即可运行
 ```
  https://github.com/brendangregg/perf-tools/blob/master/iosnoop
 ```
- 磁盘延迟毛刺
    - iosnoop -ts [-d device] [-p PID] [-i iotype]
```
 ./iosnoop 
Tracing block I/O. Ctrl-C to end.
COMM         PID    TYPE DEV      BLOCK        BYTES     LATms
jbd2/sda3-36 368    WS   8,0      975467552    106496     0.54
jbd2/sda3-36 368    FF   8,0      18446744073709551615 0          7.99
<idle>       0      WS   8,0      975467760    4096       0.31
<idle>       0      FF   8,0      18446744073709551615 0          7.95
pd-server    11563  WS   8,0      1816451512   4096       0.30
pd-server    11563  FF   8,0      18446744073709551615 0         12.24
pd-server    11440  WS   8,0      1667165688   4096       0.28
pd-server    11440  WS   8,0      1667165712   4096       0.51
pd-server    11440  WS   8,0      1667165728   4096       0.60
pd-server    11440  FF   8,0      18446744073709551615 0         10.86
pd-server    11440  WS   8,0      1667175136   4096       0.35
pd-server    11440  FF   8,0      18446744073709551615 0          4.30
jbd2/sda3-36 368    WS   8,0      975467768    61440      0.45
jbd2/sda3-36 368    FF   8,0      18446744073709551615 0         12.99
<idle>       0      WS   8,0      975467888    4096       0.32
<idle>       0      FF   8,0      18446744073709551615 0          7.95
pd-server    11415  WS   8,0      1816451512   4096       0.26
pd-server    11415  FF   8,0      18446744073709551615 0         14.46
pd-server    11415  WS   8,0      1667165688   4096       0.32
pd-server    11415  WS   8,0      1667165712   4096       0.54
pd-server    11415  WS   8,0      1667165728   4096       0.60
pd-server    11415  FF   8,0      18446744073709551615 0         25.81
pd-server    11415  WS   8,0      1667175136   4096       0.27
pd-server    11415  FF   8,0      18446744073709551615 0          4.34
kworker/u24: 23574  W    8,0      1806835880   688128     1.95
kworker/u24: 23574  W    8,0      1806837224   155648     3.65
<idle>       0      WM   8,0      1113488688   4096       3.57
pd-server    5016   WS   8,0      1816451512   4096       0.31
pd-server    5016   FF   8,0      18446744073709551615 0         15.55
<idle>       0      WS   8,0      1667165752   12288      0.40
pd-server    11415  FF   8,0      18446744073709551615 0         15.41
pd-server    11415  WS   8,0      1667175144   4096       0.38
pd-server    11415  FF   8,0      18446744073709551615 0          4.06
jbd2/sda3-36 368    WS   8,0      975467896    69632      0.51
jbd2/sda3-36 368    FF   8,0      18446744073709551615 0         18.27
```

#### 2.2.5 其他IO工具
- fio 可以用于测我们比较关注的三个磁盘的重要指标:读写带宽、IOPS、fsync / 每秒，另外还可以获得 io 延迟分布
    - fsync = n，n 次 write 之后调用一次 fsync 
- pg_test_fsync 用于测 fsync 性能，简单易用


### 2.3 Memory
#### 2.3.1 TiDB Memory
- 查看 in-use 内存
    - go tool pprof -http=:8080 debug/heap
    - 从 heap profile 图里，我们可以看 in-use 的内存，都是由哪些函数分配出来的
- 查看历史上 alloc 的 space
    - go tool pprof -alloc_space -http=:8080 debug/heap
    - 在 profile heap 的时候指定 -alloc_space，可以看到总共分配过的内存，即使已经被释放了
- 查看 mutex 竞争情况
    - go tool pprof -contentions -http=:8080 debug/mutex
 

#### 2.3.2 TiKV Memory
- 方式1:perf 命令
    - https://github.com/pingcap/tidb-inspect-tools/tree/master/tracing_tools/perf
    - cd tracing_tools/perf
    - sudo sh mem.sh $tikv-pid
    - 针对 TiKV 需要做一些事情 sudo perf probe -x {tikv-binary} -a malloc，然后用 perf record -e
probe_tikv:malloc -F 99 -p $1 -g -- sleep 10 替代 mem.sh 的第一个命令
- 方式2:bcc 工具
    - Linux 4.9 及以上版本
    - sudo /usr/share/bcc/tools/stackcount -p $tikv-pid -U $tikv-binary-path:malloc > out.stacks
    - ./stackcollapse.pl < out.stacks | ./flamegraph.pl --color=mem \
    - --title="malloc() Flame Graph" --countname="calls" > out.svg
    - http://www.brendangregg.com/FlameGraphs/memoryflamegraphs.html
- 方式3:jemalloc 统计信息
    - tiup ctl tikv --host=$tikv-ip:$tikv-status-port metrics --tag=jemalloc
    - tiup ctl:nightly tikv --host=$tikv-ip:$tikv-status-port metrics --tag=jemalloc

```
tiup ctl tikv --host=10.234.11.200:20180 metrics --tag=jemalloc
```
报错如下（用第二条命令下载之后还是报同样的错误）：
```
DebugClient::metrics: RpcFailure: 1-CANCELLED Received http2 header with status: 404
Error: exit status 255
Error: run `/root/.tiup/components/ctl/v4.0.4/ctl` (wd:/root/.tiup/data/jemalloc) failed: exit status 1
```

### 2.4 Vtune
> https://www.jianshu.com/p/9e7f0b0ef358

```
Complete
--------------------------------------------------------------------------------
Thank you for installing and for using the Intel VTune Amplifier 2019 Update 3


To get started using Intel VTune Amplifier 2019 Update 3:
  - To set your environment variables:
    - csh/tcsh users: source
/opt/intel/vtune_amplifier_2019.3.0.590814/amplxe-vars.csh
    - bash users: source
/opt/intel/vtune_amplifier_2019.3.0.590814/amplxe-vars.sh
  - To start the graphical user interface: amplxe-gui
  - To use the command-line interface: amplxe-cl
  - For additional product configuration instructions, see the 
    post-installation steps in the Intel VTune Amplifier 
    Installation Guide:
/opt/intel/vtune_amplifier_2019.3.0.590814/documentation/en/
  - For more getting started resources:
/opt/intel/vtune_amplifier_2019.3.0.590814/documentation/en/welcomepage/get_star
ted.htm

To view movies and additional training, visit
http://www.intel.com/software/products
```
打开VTune应用
```
/opt/intel/vtune_amplifier/bin64/amplxe-gui
```

远程使用Vtune需要先安装vnc
> https://blog.csdn.net/qq_26733603/article/details/104346282


## 三、实操
### 3.1 准备

先在自己电脑上执行 ```curl http://10.234.11.200:10080/debug/zip?seconds=60 --output debug.zip```，看下没有进行压测的CPU状态；

执行``` go tool pprof -http=:8080 debug/profile``` 命令时，浏览器报如下错误
```
Could not execute dot; may need to install graphviz.
```
说明没有安装```graphviz```，执行```brew install graphviz```进行安装

之后再次执行``` go tool pprof -http=:8080 debug/profile```就可以在本地浏览器上看到分析图
![](https://img-blog.csdnimg.cn/20200830091605517.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

热力图
![](https://img-blog.csdnimg.cn/20200901193153674.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

### 3.2 开始压测
开始对集群进行压测，因为之前用这个测试比较顺利并且效果明显
```
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 10m  --threads 32/64/128
```

第一次压测时间 09:22~09:32
```
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 10m  --threads 32
```
iotop输出如下
```
Total DISK READ :     422.62 K/s | Total DISK WRITE :     491.16 K/s
Actual DISK READ:     422.62 K/s | Actual DISK WRITE:     967.08 K/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND                                                                                                
  368 be/3 root        0.00 B/s    0.00 B/s  0.00 % 97.15 % [jbd2/sda3-8]
11438 be/4 tidb        0.00 B/s   19.04 K/s  0.00 % 12.77 % bin/pd-server --name=pd-10.234.11.200-2379 --client-~nf/pd.toml --log-file=/tidb-deploy/pd-2379/log/pd.log
12031 be/4 tidb        0.00 B/s   15.23 K/s  0.00 % 11.79 % bin/pd-server --name=pd-10.234.11.200-2379 --client-~nf/pd.toml --log-file=/tidb-deploy/pd-2379/log/pd.log
11445 be/4 tidb        0.00 B/s    7.61 K/s  0.00 %  9.74 % bin/pd-server --name=pd-10.234.11.200-2379 --client-~nf/pd.toml --log-file=/tidb-deploy/pd-2379/log/pd.log
12673 be/4 tidb      118.03 K/s    0.00 B/s  0.00 %  8.82 % bin/tikv-server --addr 0.0.0.0:20160 --advertise-add~tidb-deploy/tikv-20160/log/tikv.log [sched-worker-po]
12500 be/4 tidb       57.11 K/s    0.00 B/s  0.00 %  7.97 % bin/tikv-server --addr 0.0.0.0:20161 --advertise-add~tidb-deploy/tikv-20161/log/tikv.log [unified-read-po]
12670 be/4 tidb        0.00 B/s    0.00 B/s  0.00 %  6.98 % bin/tikv-server --addr 0.0.0.0:20160 --advertise-add~tidb-deploy/tikv-20160/log/tikv.log [sched-worker-po]
12612 be/4 tidb       64.73 K/s    0.00 B/s  0.00 %  6.78 % bin/tikv-server --addr 0.0.0.0:20160 --advertise-add~tidb-deploy/tikv-20160/log/tikv.log [unified-read-po]
12590 be/4 tidb       64.73 K/s    0.00 B/s  0.00 %  5.51 % bin/tikv-server --addr 0.0.0.0:20162 --advertise-add~tidb-deploy/tikv-20162/log/tikv.log [unified-read-po]
12591 be/4 tidb       64.73 K/s    0.00 B/s  0.00 %  4.72 % bin/tikv-server --addr 0.0.0.0:20160 --advertise-add~tidb-deploy/tikv-20160/log/tikv.log [unified-read-po]
12518 be/4 tidb       53.30 K/s    0.00 B/s  0.00 %  4.02 % bin/tikv-server --addr 0.0.0.0:20161 --advertise-add~tidb-deploy/tikv-20161/log/tikv.log [store-read-norm]
12933 be/4 tidb        0.00 B/s    7.61 K/s  0.00 %  2.93 % bin/tidb-server -P 4000 --status=10080 --host=0.0.0.~b.toml --log-file=/tidb-deploy/tidb-4000/log/tidb.log
12683 be/4 tidb        0.00 B/s   49.50 K/s  0.00 %  0.42 % bin/tikv-server --addr 0.0.0.0:20160 --advertise-add~ /tidb-deploy/tikv-20160/log/tikv.log [raftstore-5-1]
12538 be/4 tidb        0.00 B/s   41.88 K/s  0.00 %  0.39 % bin/tikv-server --addr 0.0.0.0:20161 --advertise-add~ /tidb-deploy/tikv-20161/log/tikv.log [raftstore-1-0]
```
看这io也还好

第二次压测时间09:50~10:00
```
./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 10m  --threads 64
```
iostat输出如下：
```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.50    0.00    0.25   25.46    0.00   73.79

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.68    0.00    0.59   15.42    0.00   82.31

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           2.34    0.00    0.67   12.21    0.00   84.78

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.33    0.00    0.42   12.72    0.00   85.54
```
iowait较大，可能是磁盘负载有点过大了

### 3.3 监控分析
- 第二次开始测试的时候，```StmtPrepare OK```的QPS打的很高，是在准备测试数据吗？
![](https://img-blog.csdnimg.cn/20200830101301474.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


- 第一次测试的时候，中间有一段时间 Read sda 带宽打的很高，但是第二次没有，，这暂时没找到是啥原因；
![](https://img-blog.csdnimg.cn/20200830101437377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


- 只看```node_exporter```图，两次压测的时候，区别不是很大，不过在没有压测的时候，``` io util```也会被打满一段时间，这是因为有一些异步磁盘同步回写的操作吗？这些操作是不是太占IO了？
![](https://img-blog.csdnimg.cn/20200830101601849.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


- 对比前后两次的``` tikv memory```，发现第二次用了更多的内存，应该是缓存了更多的数据到内存中，但是最后的每分钟成交笔数```tpmC: 68.8```还没有第一次的```tpmC: 74.5```高，这是因为啥资源占用导致的呢？这个内存的控制可以放到代码中来优化吗，而不是调整机器的系统参数？
![](https://img-blog.csdnimg.cn/20200830101924287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


- 查看两次压测时CPU的调用情况(go tool pprof -http=:8080 debug/profile)，发现都是不一样的
![](https://img-blog.csdnimg.cn/20200830115031314.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20200830115108700.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


- 查看内存使用情况 (```go tool pprof -http=:8082 debug/heap```）,调用情况有比较大的差别，而且第二次调用了更多的函数而且还使用了更多的内存
![](https://img-blog.csdnimg.cn/20200830115349526.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20200830115814933.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


- 使用```go tool pprof -contentions -http=:8080 debug/mutex```查看mutex竞争情况，发现两次的数量和调用情况差异也还蛮大的
![](https://img-blog.csdnimg.cn/2020083012114183.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20200830121255250.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhbWFuY2hlbg==,size_16,color_FFFFFF,t_70#pic_center)


## 四、issue

- 部署环境
   - CPU   12核 Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
   - 内存  16GB
   - 磁盘  ST1000DM010 - hard drive - 1 TB
   - 一台机器上面部署3个TIKV，1个PD，1个TIDB，1个Tiflash
- 压测命令
   - ```./bin/go-tpc tpcc -H 10.234.11.200  -P 4000 -D tpcc --warehouses 4 run --time 10m  --threads 32```
- 对应profile的情况
  - 在进行压测的时候，I/O util被打满，但是在没有压测时候，也会出现I/O Util被打满一段时间的情况
- 建议优化
  - 没在进行压测的时候，可能是有异步任务在跑，会对磁盘进行一些数据的读写，这个不可避免
  - 建议在做异步任务的时候，根据机器的配置，适当的降低一下并发度或者流量
  - 可以根据机器的配置，在编译的时候就把这个参数设置好，或者可以后期通过配置文件进行修改

> https://github.com/pingcap/tidb/issues/19668
