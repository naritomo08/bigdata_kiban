# Hadoop基盤 を Ubuntu 24.04 限定で構築する Ansible

前に提示したHadoop/Hive/Zeppelin構築についてまとまったため、
ansibleで構築自動化させました。

https://qiita.com/naritomo08/items/cf1d89b753a217617c09

https://qiita.com/naritomo08/items/f68ccb8e9c0a59b9ba02

https://qiita.com/naritomo08/items/d03e78202c759f2a2c06

## 前提

site.yml と各 role の先頭で `assert` を行い、Ubuntu 24.04 以外では必ず失敗します。
メモリは2GBあれば動きます。

本playbookではシングルHadoop&Hive&Zeppelinまで構築できます。

## 事前準備

```bash
git clone git@github.com:naritomo08/bigdata_kiban.git
cd github_kiban
ansible-galaxy collection install community.postgresql
vi inventory
→ホスト名、IP部分を変更する。
```

## 実行

```bash
ansible-playbook -i inventory.ini single_site.yml
```

## 変数

`group_vars/all.yml` を環境に合わせて調整してください（IP/ホスト名、BigTop repo URL、メモリ等）。

## 動作確認(HDFS)

```bash
sudo su - hadoop
hdfs dfs -mkdir /input
echo "hello hadoop hadoop yarn" | hdfs dfs -put - /input/test.txt

hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples-3.3.6.jar \
  wordcount \
  -D mapreduce.map.memory.mb=256 \
  -D mapreduce.reduce.memory.mb=256 \
  -D yarn.app.mapreduce.am.resource.mb=512 \
  -D mapreduce.map.java.opts="-Xmx200m" \
  -D mapreduce.reduce.java.opts="-Xmx200m" \
  -D yarn.app.mapreduce.am.command-opts="-Xmx400m" \
  /input /output
```

結果確認：

```bash
hdfs dfs -cat /output/part-r-00000
→それぞれの単語と数が出てくること。

やり直す際は以下のコマンドを入れてまた実施する。
hdfs dfs -rm -r -skipTrash /output
```

以下の管理画面が参照できること。

| 管理画面 | URL |
|-----|-----|
|NameNode|http://ホストIPアドレス:9870|
|DataNode|http://ホストIPアドレス:9864|
|ResourceManager|http://ホストIPアドレス:8088|
|NodeManager|http://ホストIPアドレス:8042|
|TimelineService(API応答)|http://ホストIPアドレス:8188/ws/v2/timeline|
|YARN UI2|http://ホストIPアドレス:8088/ui2|

## 動作確認(Hive)

### Beeline 接続（メモリ制限付き）

```bash
sudo su - hadoop
/usr/lib/hive/bin/beeline \
  --hiveconf mapreduce.map.memory.mb=512 \
  --hiveconf mapreduce.reduce.memory.mb=512 \
  --hiveconf yarn.app.mapreduce.am.resource.mb=512 \
  --hiveconf mapreduce.map.java.opts="-Xmx384m" \
  --hiveconf mapreduce.reduce.java.opts="-Xmx384m" \
  --hiveconf yarn.app.mapreduce.am.command-opts="-Xmx384m" \
  -u 'jdbc:hive2://localhost:10000/default' \
  -n hadoop
```

### 実行エンジン確認

```bash
set hive.execution.engine;
```

結果：

```bash
hive.execution.engine=mr
```

### テーブル作成・INSERT・SELECT

```bash
CREATE TABLE t1 (
  col1 INT,
  col2 STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE;

INSERT INTO t1 VALUES (1,'a'),(2,'b');

SELECT * FROM t1;
```

結果：

```bash
1   a
2   b
```

### 4. YARN UI での確認（成功の裏取り）

```bash
http://ホストIPアドレス:8088


Application Type: MAPREDUCE

State: FINISHED

Logs に Exception がないこと
```

## Hive設定

### Interpreter 設定画面

右上ユーザー → Interpreter → 右上のcreate

```bash
Interpreter Name
hive

Interpreter group
jdbc
```

properties:

以下のパラメータを編集する。

```bash
default.url
jdbc:hive2://master1:10000/default
```

以下のパラメータを追記する。

```bash
hive.driver
org.apache.hive.jdbc.HiveDriver

hive.user
hive
```

## 動作確認（HiveQL）

### 新規 Notebook 作成

Notebook → Create new note

Note Name:hive-test
Default Interpreter:hive

### Hive クエリ実行

以下の処理を順々に行う。
エラーが出ず動かせること。

```bash
%hive
SHOW DATABASES;
```

```bash
%hive
USE default;
SHOW TABLES;
```

```bash
%hive
SELECT * FROM t1 LIMIT 10;
```

→前のコマンドで何もテーブルが出てない場合、実施しないこと。

```bash
%hive
drop table t1;
```

→前のコマンドを実施していない場合、実施しないこと。

```bash
%hive
CREATE TABLE t1 (
  col1 INT,
  col2 STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE;
```

```bash
%hive
set mapreduce.map.memory.mb=256;
set mapreduce.reduce.memory.mb=256;
set yarn.app.mapreduce.am.resource.mb=256;

INSERT INTO t1 VALUES (1,'a'),(2,'b');
```

```bash
%hive
SELECT * FROM t1 LIMIT 10;
```
