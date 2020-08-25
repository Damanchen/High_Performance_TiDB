原本以为编译安装应该还好，可能会遇到一些环境问题，Google下应该就能解决，但还是太年轻了，，，

看完视频课之后，准备安装前，看到这样一篇博客，[TiDB在Centos7上通过源码编译安装](https://blog.csdn.net/feinifi/article/details/79657502)，大概看了一下，觉得还行，就开始参考着上手，先准备在centos测试机上搭一下。

### 1.安装环境依赖
```
yum install gcc-c++ git golang -y
```
cmake3比较难装，遇到了一些困难
参考 [centos安装cmake3](https://blog.csdn.net/Imagine_Dragon/article/details/78398600) 解决了

### 2.开始```git clone```编译代码
#### 2.1 尝试gitee
```git clone```代码的时候很慢，几度都想放弃了，直到看到了这篇文章 [Github源代码下载速度慢的解决办法](https://zhuanlan.zhihu.com/p/102409790) 把github上的代码fork再同步到自己的gitee账户上；

这样```git clone```是很快，但是编译的时候又会出问题，因为在 ```makefile``` 文件里设置了很多```github.com/pingcap```的配置项；

本来想看看修改下```Makefile```文件，再进行编译的，但是发现```makefile```文件太复杂了，估计改了只会错的更多，所以就换思路了。

#### 2.2 科学上网
尝试了gitee之后，觉得还是应该直接```git clone https://github.com/pingcap/tidb.git```才是最正确，最省事的方式，所以就想到FQ加速，看看能不能快速下载；

下了几个FQ软件，试了好几次，还是一如既往的慢，速度提不起来，一直```git clone```不了，，，

#### 2.3 dockerfile
后面发现有dockerfile文件，想直接用dockerfile文件编译镜像，然后再修改源码，尝试编译；

但用dockerfile生成镜像的时候，还是因为网络问题，一直不能成功，，，

#### 2.4 最后尝试
放弃了测试机，最后还是硬着头皮用```git clone```把```tidb```项目下载到了本地**mac环境**，然后到tidb的目录下，然后执行```make```命令，等了一两分钟，没有报错，提示编译好了，感激涕零啊。
```
$ make
CGO_ENABLED=1 GO111MODULE=on go build  -tags codes  -ldflags '-X "github.com/pingcap/parser/mysql.TiDBReleaseVersion=v4.0.0-beta.2-960-g5184a0d70" -X "github.com/pingcap/tidb/util/versioninfo.TiDBBuildTS=2020-08-16 02:27:00" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitHash=5184a0d7060906e2022d18f11532f119f5df3f39" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitBranch=master" -X "github.com/pingcap/tidb/util/versioninfo.TiDBEdition=Community" ' -o bin/tidb-server tidb-server/main.go
Build TiDB Server successfully!
```

**建议反馈**
感觉直接```git clone```GitHub上的开源项目对国内的用户体验不是很好。应该是有其他方式绕过这个问题，直接进行编译和部署的，但是自己目前对go项目的编译部署不是很熟悉，所以暂时放弃了，，，

### 3.代码学习与修改
关于作业题目“改写tidb源码，使得TiDB启动事务时，能打印出一个‘hello transaction’”，看了下源码，先在如下地方做了修改：
```
func (s *session) NewTxn(ctx context.Context) error {
	if s.txn.Valid() {
		txnID := s.txn.StartTS()
		err := s.CommitTxn(ctx)
		if err != nil {
			return err
		}
		vars := s.GetSessionVars()
		logutil.Logger(ctx).Info("NewTxn() inside a transaction auto commit",
			zap.Int64("schemaVersion", vars.TxnCtx.SchemaVersion),
			zap.Uint64("txnStartTS", txnID))
	}

	txn, err := s.store.Begin()
	logutil.BgLogger().Info("hello transaction --- by cfq")
	if err != nil {
		return err
	}
```
发现会输出大量的“hello transaction --- by cfq”，应该不是我们想要的效果，又在下面的地方做了修改：
```
func (e *SimpleExec) executeBegin(ctx context.Context, s *ast.BeginStmt) error {
	// If BEGIN is the first statement in TxnCtx, we can reuse the existing transaction, without the
	// need to call NewTxn, which commits the existing transaction and begins a new one.
	txnCtx := e.ctx.GetSessionVars().TxnCtx
	if txnCtx.History != nil {
		err := e.ctx.NewTxn(ctx)
		if err != nil {
			return err
		}
	}
	if e.ctx.GetSessionVars().ConnectionID != 0 {
		logutil.BgLogger().Info("hello transaction --- by cfq")
	}
	// With START TRANSACTION, autocommit remains disabled until you end
	// the transaction with COMMIT or ROLLBACK. The autocommit mode then
	// reverts to its previous state.
	e.ctx.GetSessionVars().SetStatusFlag(mysql.ServerStatusInTrans, true)
```

### 4.结果
编译输出如下：
```
$ make
CGO_ENABLED=1 GO111MODULE=on go build  -tags codes  -ldflags '-X "github.com/pingcap/parser/mysql.TiDBReleaseVersion=v4.0.0-beta.2-960-g5184a0d70-dirty" -X "github.com/pingcap/tidb/util/versioninfo.TiDBBuildTS=2020-08-17 03:15:38" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitHash=5184a0d7060906e2022d18f11532f119f5df3f39" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitBranch=master" -X "github.com/pingcap/tidb/util/versioninfo.TiDBEdition=Community" ' -o bin/tidb-server tidb-server/main.go
Build TiDB Server successfully!
```

mysql客户端
```
 mysql --host 127.0.0.1 --port 4000 -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.7.25-TiDB-v4.0.0-beta.2-960-g5184a0d70-dirty TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

#启动一个事务
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
```

日志输出如下：
```
[2020/08/16 21:16:11.396 +08:00] [INFO] [server.go:235] ["server is running MySQL protocol"] [addr=0.0.0.0:4000]
[2020/08/16 21:16:11.396 +08:00] [INFO] [http_status.go:82] ["for status and metrics report"] ["listening on addr"=0.0.0.0:10080]
[2020/08/16 21:16:11.398 +08:00] [INFO] [domain.go:1094] ["init stats info time"] ["take time"=2.044684ms]
[2020/08/16 21:16:27.094 +08:00] [INFO] [server.go:388] ["new connection"] [conn=1] [remoteAddr=127.0.0.1:49808]
[2020/08/16 21:16:30.998 +08:00] [INFO] [simple.go:563] ["hello transaction --- by cfq"]
```
应该是达到我们想要的效果了。


参考：

[通过 TiKV 入门 Rust](
https://maiyang.me/post/2018-08-02-rust-guide-by-tikv/)

[go的编译命令执行过程](
https://halfrost.com/go_command/)

-------------后续--------------
## 用deepin linux环境测试
### clone&make
```
git clone https://github.com/pingcap/tidb.git
git clone https://github.com/tikv/pd.git
git clone https://github.com/tikv/tikv.git
```

tidb和pd直接到对应目录下面make就能编译成功，tikv因为是rust语言写的，需要先安装rust然后再编译，执行如下命令安装rust：
```
 curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
之后也是执行make就能编译成功，只是时间比较久。

### deploy
参考TiUP的启动方式
#### PD
```
cd pd
mkdir pd-1
./bin/pd-server --name=pd-1 --data-dir=./pd-1/data --peer-urls=http://127.0.0.1:2380 --advertise-peer-urls=http://127.0.0.1:2380 --client-urls=http://127.0.0.1:2379 --advertise-client-urls=http://127.0.0.1:2379 --log-file=./pd-1/pd.log --initial-cluster=pd-1=http://127.0.0.1:2380 &
```

#### TiKV
```
cd tikv
mkdir tikv-1
./target/release/tikv-server --addr=127.0.0.1:20160 --advertise-addr=127.0.0.1:20160 --status-addr=127.0.0.1:20180 --pd=http://127.0.0.1:2379 --config=./tikv-1/tikv.toml --data-dir=./tikv-1/data --log-file=./tikv-1/tikv.log &
```
需要配置下 tikv.toml：
```
[rocksdb]
max-open-files = 256
[raftdb]
max-open-files = 256
[storage]
reserve-space = 0
```

#### TiDB
```
cd tidb
mkdir tidb-1
./bin/tidb-server -P 4000 --store=tikv --host=127.0.0.1 --status=10080 --path=127.0.0.1:2379 --log-file=./tidb-1/tidb.log &
```

### test
使用mysql连接进行测试：
```
mysql -h127.0.0.1 -P4000 -uroot
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.25-TiDB-v4.0.0-beta.2-1016-gf5fa3e7aa TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| INFORMATION_SCHEMA |
| METRICS_SCHEMA     |
| PERFORMANCE_SCHEMA |
| mysql              |
| test               |
+--------------------+
5 rows in set (0.00 sec)
```
说明是可以使用的，其余步骤就和之前是一样的了。
