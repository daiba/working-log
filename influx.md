## influx data 構築資料

### influxdb [構築](https://docs.influxdata.com/influxdb/v1.5/introduction/installation/)
```
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y install influxdb
$ sudo systemctl start influxdb
```
設定表示は
```
$ influxd config
```

### telegraf [構築](https://docs.influxdata.com/telegraf/v1.5/introduction/installation/)
```
$  curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
$ source /etc/lsb-release
$ echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y install telegraf
```

### kapacitor [構築](https://docs.influxdata.com/kapacitor/v1.4/)
```
$ wget https://dl.influxdata.com/kapacitor/releases/kapacitor_1.4.1_amd64.deb
$ sudo dpkg -i kapacitor_1.4.1_amd64.deb
```

### influxdb [使い方](https://docs.influxdata.com/influxdb/v1.5/introduction/getting-started/)
exitで終了．
```
$ sudo service influxd start
$ influx -precision rfc3339
```

#### CREATE
```
> CREATE DATABASE mydb
> SHOW DATABASES
name: databases
name
----
telegraf
_internal
mydb
> USE mydb
Using databse mydb
```

#### INSERT
```
> INSERT cpu,host=serverA,region=us_west value=0.64
```

#### SELECT
```
> SELECT "host", "region", "value" FROM "cpu"
name: cpu
time                           host    region  value
----                           ----    ------  -----
2018-04-11T08:19:42.143650687Z serverA us_west 0.64
```

####  SELECT *
```
> INSERT temperature,machine=unit42,type=assembly external=25,internal=37
> SELECT * FROM "temperature"
name: temperature
time                          external internal machine type
----                          -------- -------- ------- ----
2018-04-11T08:22:41.06977588Z 25       37       unit42  assembly
```

#### LIMIT
```
> SELECT * FROM /.*/ LIMIT 1
name: cpu
time                           external host    internal machine region  type value
----                           -------- ----    -------- ------- ------  ---- -----
2018-04-11T08:19:42.143650687Z          serverA                  us_west      0.64
name: temperature
time                          external host internal machine region type     value
----                          -------- ---- -------- ------- ------ ----     -----
2018-04-11T08:22:41.06977588Z 25            37       unit42         assembly 
```

#### DROP
```
> SHOW DATABASES
name: databases
name
----
telegraf
_internal
mydb
> DROP DATABASE mydb
> SHOW DATABASES
name: databases
name
----
telegraf
_internal
```

### telegraf [使い方](https://docs.influxdata.com/telegraf/v1.5/introduction/getting-started/)
```
$ telegraf -sample-config -input-filter cpu:mem -output-filter influxdb > telegraf.conf
$ ps -ef | grep telegraf
telegraf  1129     1  0 00:40 ?        00:00:03 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d
$ grep inputs. /etc/telegraf/telegraf.conf
# declared inputs, and sent to the declared outputs.
  ## Precision will NOT be used for service inputs. It is up to each individual
[[inputs.cpu]]
[[inputs.disk]]
[[inputs.diskio]]
...
```

構造
```
$ influx -precision rfc3339
Connected to http://localhost:8086 version 1.5.1
InfluxDB shell version: 1.5.1
> SHOW DATABASES
name: databases
name
----
telegraf
_internal
> USE telegraf
Using database telegraf
> SHOW MEASUREMENTS
name: measurements
name
----
cpu
disk
diskio
kernel
mem
processes
swap
system
> SHOW FIELD KEYS
name: cpu
fieldKey         fieldType
--------         ---------
usage_guest      float
usage_guest_nice float
usage_idle       float
usage_iowait     float
usage_irq        float
usage_nice       float
usage_softirq    float
usage_steal      float
usage_system     float
usage_user       float
...

> SELECT usage_idle FROM cpu WHERE cpu = 'cpu-total' LIMIT 5
name: cpu
time                 usage_idle
----                 ----------
2018-04-11T07:36:30Z 86.98698698698698
2018-04-11T07:36:40Z 99.30000000000005
2018-04-11T07:36:50Z 66.56626506024094
2018-04-11T07:37:00Z 99.7995991983968
2018-04-11T07:37:10Z 99.60000000000001
```
CPUについての[解説](https://github.com/influxdata/telegraf/blob/master/plugins/inputs/system/CPU_README.md)

### kapacitor [使い方](https://docs.influxdata.com/kapacitor/v1.4/introduction/getting-started/)
```
$ curl -G 'http://localhost:8086/query?db=telegraf' --data-urlencode 'q=SELECT mean(usage_id
le) FROM cpu'
{"results":[{"statement_id":0,"series":[{"name":"cpu","columns":["time","mean"],"values":[["1970-01-01T00:00:00Z",9
9.47764781572646]]}]}]}

$ cat cpu_alert.tick 
dbrp "telegraf" . "autogen"
stream
  |from()
    .measurement('cpu')
  |alert()
    .crit(lambda: int("usage_idle") < 70)
    .log('/tmp/alerts.log')
```

tickスクリプト登録
```
$ kapacitor define cpu_alert -tick cpu_alert.tick

$ kapacitor list tasks
ID        Type      Status    Executing Databases and Retention Policies
cpu_alert stream    disabled  false     ["telegraf"."autogen"]

$ kapacitor show cpu_alert
ID: cpu_alert
Error: 
Template: 
Type: stream
Status: disabled
Executing: false
Created: 12 Apr 18 02:49 UTC
Modified: 12 Apr 18 02:49 UTC
LastEnabled: 01 Jan 01 00:00 UTC
Databases Retention Policies: ["telegraf"."autogen"]
TICKscript:
dbrp "telegraf"."autogen"
stream
    |from()
        .measurement('cpu')
    |alert()
        .crit(lambda: int("usage_idle") < 70)
        .log('/tmp/alerts.log')
DOT:
digraph cpu_alert {
stream0 -> from1;
from1 -> alert2;
}
```
実行
```
$ kapacitor record stream -task cpu_alert -duration 60s
deb156bc-06c6-4ea2-b6a2-e8840a5435f9
$ kapacitor list tasks
ID        Type      Status    Executing Databases and Retention Policies
cpu_alert stream    disabled  false     ["telegraf"."autogen"]

$ kapacitor list recordings deb156bc-06c6-4ea2-b6a2-e8840a5435f9
ID                                   Type    Status    Size      Date                   
deb156bc-06c6-4ea2-b6a2-e8840a5435f9 stream  finished  384 B     12 Apr 18 07:34 UTC
```
削除
```
$ kapacitor delete tasks cpu_alert
$ kapacitor list tasks
ID Type      Status    Executing Databases and Retention Policies

$ kapacitor record stream -task cpu_alert -duration 60s
0a355208-ff0b-40a3-a5ce-1caa1b841579
```
最終的に動いたtickスクリプト
```
$ cat cpu_alert.tick 
dbrp "telegraf"."autogen"
stream
  |from()
    .measurement('cpu')
  |alert()
    .crit(lambda: "usage_idle" < 100.0)
    .log('/tmp/alerts.log')
```
### telegraf plugin[作成](https://www.influxdata.com/blog/telegraf-go-collection-agent/)
#### telegraf 停止
| 操作 | コマンド |
|:--|:--|
| サービス起動 | systemctl start ${unit} |
| サービス停止 | systemctl stop ${unit} |
| サービス再起動 | systemctl restart ${unit} |
| サービスリロード | systemctl reload ${unit} |
| サービスステータス表示 | systemctl status ${unit} |
| サービス自動起動有効 | systemctl enable ${unit} |
| サービス自動起動無効 | systemctl disable ${unit} |
| サービス自動起動設定確認 | sytemctl is-enabled ${unit} |
| サービス一覧 | systemctl list-unit-files --type=service |
| 設定ファイル再読み込み | systemctl daemon-reload |

起動状態
```
$ systemctl list-unit-files --type=service | grep tel
telegraf.service                           enabled 
$ systemctl is-enabled telegraf.service
enabled
```

PATHに $HOME/go/bin を追加しておく
```
$ go get github.com/influxdata/telegraf
$ cd /go/src/github.com/influxdata/telegraf
$ git checkout -b MyPlugin
Switched to a new branch 'MyPlugin'
$ make
```

### telegrafを動かす

### 使い方
```
$ cd go/src/github.com/influxdata/telegraf
$ vi plugins/inputs/icmp/icmp.go
$ make
$ sudo ./telegraf -config /home/kei/scratch/telegraf.conf
```

```
$ grep -v \# scratch/telegraf.conf | grep -v ^$
[global_tags]
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = ""
  debug = false
  quiet = false
  logfile = ""
  hostname = ""
  omit_hostname = false
[[outputs.influxdb]]
[[inputs.icmp]]
  device = "ens4"
  ```
  
