# drbd-perf-on-aws
# 1.はじめに
オンプレで共有ディスクを用いたHA構成をクラウド上で実現しようとした場合、単純には共有ディスク構成を作ることができません(左図)。一応、AWSではマルチアタッチを使用して複数インスタンスにEBSを割当てたり、Azureでも同様な機能は提供されてはいますが、制約もあるためそのまま共有ディスクの変わりというわけにはいかないようです。
※上記AWS/Azureいずれの機能も(アベイラビリティ)ゾーンを跨いだ構成は組めません。

- [Amazon EBSのマルチアタッチ機能とAzure Shared Disksを比較してみる](https://qiita.com/hayao_k/items/b978cd26622536000a6a)
- [EBS Multi-Attachの使いどころを考えてみた](https://dev.classmethod.jp/articles/ebs-multi-attach-usecase-thought/)

<img width="600" alt="img1.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/181305/90dce159-535d-7a8f-659d-ff6605d1a661.png">

他のやり方として、レプリケーションソフトを使って代替する手があるのですが(右図)、その場合はレプリケーションソフトによる同期処理による(write)性能が気になります。本記事では、AWS上にDRBDというボリュームレプリケーション(OSS)を使って擬似HA構成を組んだ場合、write性能がどうなるか調べてみたいと思います。

# 2.システム構成
## 2-1.概略イメージ
<img width="800" alt="img7.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/181305/5f90a1ee-7f2a-b2ef-8cf1-e83093447495.png">

## 2-2.AWS構成図
<img width="800" alt="img3.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/181305/b0ff917e-9057-a3df-9397-aae450de22c8.png">

## 2-3.性能測定イメージ
<img width="600" alt="img2.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/181305/ac086dcd-d694-41c4-30e8-f8cb5f56f1d4.png">

## 2-4.ホスト情報
- host1-3共通:
    - t2/medium
    - CentOS 8.2.2004
    - Linux version 4.18.0-193.6.3.el8_2.x86_64

- host1:
    - IP アドレス: 172.16.1.50
    - ディスク:
        - /dev/xvda - OS
        - /dev/xvdb – DRBD (/dev/drbd0)
        - /dev/xvdc – DRBD (/dev/drbd1) 
        - /dev/xvdd – ローカル
- host2:
    - IP アドレス: 172.16.1.60
    - ディスク:
        - /dev/xvda - OS
        - /dev/xvdb – DRBD (/dev/drbd0) 
- host3:
    - IP アドレス: 172.16.2.50
    - ディスク:
        - /dev/xvda - OS
        - /dev/xvdb – DRBD (/dev/drbd1) 

# 3.環境構築
## 3-1.OS準備
```
# dnf update
# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# dnf install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
# dnf install -y sysstat
# dnf install -y dstat
# dnf install -y fio
```

## 3-2.DRBDのインストール&セットアップ
### DRBDパッケージのインストール
```
# dnf install -y kmod-drbd90
# dnf install -y drbd90
```

### DRBDの設定
host1-host2間の同期設定。host1で作成し、host2へコピーします。

```:/etc/drbd.d/r0.res
resource r0{
  net {
   protocol C;
  }
  options {
  }
  on ip-172-16-1-50.ap-northeast-1.compute.internal {
    address 172.16.1.50:7789;
    device /dev/drbd0;
    disk /dev/xvdb;
    meta-disk internal;
  }
  on ip-172-16-1-60.ap-northeast-1.compute.internal {
    address 172.16.1.60:7789;
    device /dev/drbd0;
    disk /dev/xvdb;
    meta-disk internal;
  }
}
```

host1-host3間の同期設定。host1で作成し、host3へコピーします。

```:/etc/drbd.d/r1.res
resource r1{
  net {
   protocol C;
  }
  options {
  }
  on ip-172-16-1-50.ap-northeast-1.compute.internal {
    address 172.16.1.50:7790;
    device /dev/drbd1;
    disk /dev/xvdc;
    meta-disk internal;
  }
  on ip-172-16-2-50.ap-northeast-1.compute.internal {
    address 172.16.2.50:7790;
    device /dev/drbd1;
    disk /dev/xvdb;
    meta-disk internal;
  }
}
```
### リソース作成と有効化
host1-host2間のリソース作成と有効化。host1, 2で実行します。

```
# drbdadm create-md r0
# drbdadm up r0
```
host1-host3間のリソース作成と有効化。host1, 3で実行します。

```
# drbdadm create-md r1
# drbdadm up r1
```

### 同期実行
下記はhost1のみで実施します。

```
// host1-host2間同期開始
# drbdadm primary --force r0
// host1-host3間同期開始
# drbdadm primary --force r1

// 同期中
# drbdadm status
r0 role:Primary
  disk:UpToDate
  ip-172-16-1-60.ap-northeast-1.compute.internal role:Secondary
    replication:SyncSource peer-disk:Inconsistent done:83.70

r1 role:Primary
  disk:UpToDate
  ip-172-16-2-50.ap-northeast-1.compute.internal role:Secondary
    replication:SyncSource peer-disk:Inconsistent done:73.31

// 同期完了
# drbdadm status
r0 role:Primary
  disk:UpToDate
  ip-172-16-1-60.ap-northeast-1.compute.internal role:Secondary
    peer-disk:UpToDate

r1 role:Primary
  disk:UpToDate
  ip-172-16-2-50.ap-northeast-1.compute.internal role:Secondary
    peer-disk:UpToDate
```

最後に、ファイルシステムを作成し、適当なディレクトリにマウントして置きます。

```
// ファイルスシステム作成
mkfs -t xfs /dev/drbd0
mkfs -t xfs /dev/drbd1
mkfs -t xfs /dev/xvdd

// マウント
mount /dev/drbd0 /mnt/drbd0-fs/
mount /dev/drbd1 /mnt/drbd1-fs/
mount /dev/xvdd /mnt/local-fs/

// 確認
# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        1.9G     0  1.9G   0% /dev
tmpfs           1.9G  8.0K  1.9G   1% /dev/shm
tmpfs           1.9G   17M  1.9G   1% /run
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/xvda2      8.0G  1.3G  6.7G  17% /
tmpfs           378M     0  378M   0% /run/user/1000
/dev/xvdd       100G  858M  100G   1% /mnt/local-fs
/dev/drbd0      100G  858M  100G   1% /mnt/drbd0-fs
/dev/drbd1      100G  826M  100G   1% /mnt/drbd1-fs
```

## 3-3.PostgreSQLのインストール&セットアップ
### PostgreSQLのインストール
PostgreSQLは、CentOS標準リポジトリから、下記でインストールしました。今回は性能測定にpgbenchというツールを使うため、合せてそちらもインストールしておきます。

```
# dnf install -y postgresql
# dnf install -y postgresql-server
# dnf install -y postgresql-contrib  //pgbench用
```

### セットアップ
次に、pgbenchを使うためのデータベースの設定を行います。今回は、1.AZ内レプリケーション(DRBD0)、2.AZ跨ぎレプリケーション(DRBD1)、3.ローカルディスクの3パターンで測定を実施するため、データベースは下記マウントしたファイルシステムをそれぞれ利用します。DRBDのセットアップ時に作成した、/mnt以下のディレクトリを確認します。

```
// mountポイント確認
# cd /mnt
//postgreSQLは、デフォルトpostgresユーザを利用するため、所有者とパーミッションを下記の通り変更します。
# chown -R postgres:postgres drbd0-fs drbd1-fs local-fs
# chmod -R 700 drbd0-fs drbd1-fs local-fs
// 結果はこんな感じになります
# ll
total 0
drwxr-xr-x 2 postgres postgres 6 Jul 26 04:06 drbd0-fs
drwxr-xr-x 2 postgres postgres 6 Jul 26 04:06 drbd1-fs
drwxr-xr-x 2 postgres postgres 6 Jul 26 04:08 local-fs
```

3つのデータベースは、今回はSystemdの設定を使って切替える事にしました(もっとうまい方法があるかと思いますが。。)。下記のようにUnit定義の中の「PGDATA」として3つ準備します。

```:/usr/lib/systemd/system/postgresql.service 
(中略)
#Environment=PGDATA=/var/lib/pgsql/data
Environment=PGDATA=/mnt/drbd0-fs 
#Environment=PGDATA=/mnt/drbd1-fs
#Environment=PGDATA=/mnt/local-fs
(中略)
```

その後、上記各設定ファイルのコメントアウトを(1つずつ)削除し、下記コマンドを実行しデータベースの初期化を行います(3回実施することになります)。

```
# systemctl daemon-reload   #Unit定義更新
# postgresql-setup initdb   #データベース初期化
```

データベースのセットアップは以上になります。

# 4.ディスク性能測定(fio)
## 4-1.測定方法
### fio設定ファイル

```:fio.conf
[global]
ioengine=libaio
iodepth=1
size=1g
direct=1
runtime=30
stonewall

[Seq-Write-1M]
bs=1m
rw=write

[Rand-Write-1M]
bs=1m
rw=randwrite

[Rand-Write-512K]
bs=512k
rw=randwrite

[Rand-Write-4K]
bs=4k
rw=randwrite
```
### fioによる測定
Terminalを3つ立上げ、それぞれのTerminalにて下記を順次実行。fioの実行が終わったら、iostatとdstatをCtl+Cで止める。

```
//Terminal1で実行
iostat -mx 1 | grep --line-buffered drbd0 |  awk '{print strftime("%y/%m/%d %H:%M:%S"), $0}' | tee iostat_drbd0.txt
//Terminal2で実行
dstat -t | tee dstat_drbd0.txt
//Terminal3で実行
fio --filename=/dev/drbd0 fio.conf
```

## 4-2.測定結果(1)(スループット)
<img width="1200" alt="img4.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/181305/c6e7a740-1494-24c6-c882-483e93b4ab2e.png">

## 4-3.測定結果(2)(write待ち時間)
iostatの「w_await」結果。[iostat manual](https://man7.org/linux/man-pages/man1/iostat.1.html)によると、

>
The average time (in milliseconds) for write requests issued to the device to be served. This includes the time spent by the requests in queue and the time spent servicing them.

とのこと。

<img width="600" alt="img5.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/181305/6560bdcc-feb5-4a3f-0fcc-927927d3ad7c.png">

# 5.アプリ性能測定(pgbench)
## 5-1.測定方法
こちらのサイトを参考に、pgbenchによる測定を行います。

- pgbenchマニュアル (https://www.postgresql.jp/document/9.2/html/pgbench.html)
- pgbenchの使いこなし (https://lets.postgresql.jp/documents/technical/contrib/pgbench)

```
# systemctl daemon-reload   #使うデータベース格納場所(PGDATA)をコメントアウトし、Unit定義を更新
# systemctl start postgresql.service   # データベース起動
# sudo su - postgres
$ createdb test   #testという名のデータベース作成
$ pgbench -i test   #pgbench用のテストデータをtestデータベースへロード
$ pgbench -c 10 -t 1000 -l test   #測定実行
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
number of transactions per client: 1000
number of transactions actually processed: 10000/10000
latency average = 9.111 ms
tps = 1097.529558 (including connections establishing)
tps = 1097.770942 (excluding connections establishing)
$ exit   #postgresユーザから抜ける
# systemctl start postgresql.service   #データベース停止
```
上記より性能(TPS)は1097程度出ている事がわかります(PostgreSQLへの接続オーバーヘッド有り/無しで2パターンTPSは表示さる)。また、オプションとして「-l」を指定している事により実行後ローカルに「pgbench_log.1163」というトランザクション詳細結果ファイルが作成されます(詳細につきましては、上記サイトを確認ください)。これをデータベースを切替えてそれぞれ実行します。

## 5-2.測定結果(TPS&応答時間)
<img width="1684" alt="img6.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/181305/9893c059-4a60-179e-9ca9-da3a895364bd.png">

# 6.まとめ
TBD

# 7.参考
- [詳解 システム・パフォーマンス](https://www.amazon.co.jp/dp/4873117909)
- [ファイルシステムのベンチマーク集](http://www.nminoru.jp/~nminoru/unix/fs_benchmarks.html)
