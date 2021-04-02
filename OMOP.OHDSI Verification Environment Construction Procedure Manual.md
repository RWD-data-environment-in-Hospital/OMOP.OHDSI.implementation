# OMOP環境構築手順書

------

## １．本手順書について

OMOP　CDMの検証環境を構築するための手順をご説明いたします。

※画面イメージについてはダミーデータを用いているため、実際のデータとはデータ数・画面に表示される値が異なりますのでご留意ください。

※各パッケージのインストールには、インターネット接続が必要です。

------
## ２．システム構成

![image-20210402093512874](../images/img/image-20210402093512874.png)

| |||
| :------- | ------------------------------------------------------------ |-------------------------------------------------|
| Docker   | https://www.docker.com/                                      |本手順書では、バージョン20.10.3を使用します。     |
| Broadsea | https://github.com/hapifhir/hapi-fhir-jpaserver-starter      ||
| R studio | https://rstudio.com/                                         ||
| HADES    | https://ohdsi.github.io/Hades/rSetup.html                    ||

------
## ３．事前準備

Broadseaの動作環境構築のための事前準備をします。

### ３．１．Dockerのインストール
対象サーバに root 特権のあるユーザでログインします。
既存の yum パッケージを更新します。

```
# yum update
# yum upgrade
```
公式の安定版 Yum リポジトリを設定します。
```
# yum install -y yum-utils device-mapper-persistent-data lvm2
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
Docker をインストールします（Broadseaの操作に必要となります）。
```
# yum makecache fast
# yum install -y docker-ce
```
以下のコマンドを実行し、Dockerのバージョン情報が表示されればインストール完了です。
```
# docker version
```
Dockerを起動します。
```
# systemctl start docker
```
### ３．２．docker-compose のインストール

対象サーバに root 特権のあるユーザでログインします。

docker-compose をインストールします（Broadsea 起動に必要となります）。
```
# sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-\`uname -s\`-\`uname -m\` -o /usr/local/bin/docker-compose
# sudo chmod +x /usr/local/bin/docker-compose
```
### ３．３．R Studio のインストール

対象サーバに root 特権のあるユーザでログインします。

R studio を動作させるためのユーザを追加します。
```
# useradd rstudio
# usermod -aG wheel rstudio
# echo '[新しいパスワード]' passwd --stdin rstudio
```
EPELをインストールし、サードパーティ製のパッケージをインストール可能にします。
```
# yum install epel-release
```
R をインストールします。
```
# yum install R
```
R studio をインストールします。
```
# sudo yum install -y --nogpgcheck https://download2.rstudio.org/rstudio-server-rhel-1.0.44-x86_64.rpm
```
ブラウザから http://**\[サーバIPアドレス\]**:8787 へアクセスし、R Studio を起動します。ログイン画面が表示されたら R Studio のユーザ名・パスワードを入力し、ログインの確認を行います。

 ![image-20210401162014465](../images/img/image-20210401162014465.png)

### ３．４．Libxml2 のインストール

Libxml2 をインストールします（Achilles をインストールする際に必要となります）。
```
# sudo yum install libxml2-devel
```
### ３．５．GitHUB からの資産ダウンロード

GitHUBから以下の資産をダウンロードします。 

root特権ユーザでログインし、Broadseaの設定ファイルをGitHUBより取得します。

```
# git clone https://github.com/OHDSI/Broadsea.git
```
取得したファイルは、作業ディレクトリへ移動します。

※下記の例では、「/root/dockerwk」へ取得したファイルを移動しています。
```
# cp -r Broadsea /root/dockerwk
```
データベース管理ユーザでログインし、OMOP CDM（V5） のテーブル定義をGitHUBより取得します。
```
# sudo curl -L https://github.com/OHDSI/CommonDataModel/archive/refs/tags/v5.3.1.tar.gz -o /home/postgres/
```
取得したファイルは、作業ディレクトリへ移動します。

※下記の例では、作業ディレクトリ「/home/postgres」へ取得したファイルを移動しています。
```
# cp v5.3.1.tar.gz /home/postgres
```
コピーしたファイルを展開します。
```
# cd /home/postgres
# tar -zxvf v5.3.1.tar.gz
```
PostgreSQL用のテーブル定義は取得した資産の下記ディレクトリにあります。
​	CommonDataModel-5.3.1/PostgreSQL/
​		OMOP CDM postgresql ddl.txt
​		OMOP CDM postgresql pk indexes.txt
​		OMOP CDM postgresql constraints.txt
上記の様にファイル名にスペースが含まれている状態のため、スペースを除去した形へリネームを行います。

```
# cd CommonDataModel-5.3.1/PostgreSQL/
# mv "OMOP CDM postgresql ddl.txt" OMOPCDMpostgresqlddl.txt
# mv "OMOP CDM postgresql pk indexes.txt" OMOPCDMpostgresqlpkindexes.txt
# mv "OMOP CDM postgresql constraints.txt" OMOPCDMpostgresqlconstraints.txt
```

------
## ４．データベース構築
OMOP Common Data Model のデータベース構築を行います。
### ４．１．スキーマの作成
対象サーバに データベース管理ユーザでログインし、psqlを起動します。

```
# psql
```
OMOP CDMのテーブルを作成するスキーマ「cdmv5」と、WebAPI用スキーマ「ohdsi」を作成します。
```
postgres=# create schema cdmv5 authorization postgres;
postgres=# create schema ohdsi authorization postgres;
```
作成が完了したら「\\q」を入力し、psqlを終了します。
```
postgres=# \q
```
### ４．２．OMOP Common Data Model テーブルの作成
対象サーバに データベース管理ユーザでログインし、psqlを起動します。
```
# psql
```
カレントスキーマを「cdmv5」へ変更します。
```
postgres=# set search_path to "cdmv5";
```
カレントスキーマが「cdmv5」に変更された事を確認します。
```
postgres=# select current_schema();
 current_schema
----------------
 cdmv5
(1 行)
```

３．４．で取得したテーブル定義を実行し、OMOP CDMのテーブルを作成します。
```
postgres=# \i /home/postgres/CommonDataModel-5.3.1/PostgreSQL/OMOPCDMpostgresqlddl.txt
postgres=# \i /home/postgres/CommonDataModel-5.3.1/PostgreSQL/OMOPCDMpostgresqlddl.txt
postgres=# \i /home/postgres/CommonDataModel-5.3.1/PostgreSQL/OMOPCDMResultspostgresqlddl.txt

```
テーブルが作成された事を確認します。
```
postgres=# \dt
                   リレーションの一覧
 スキーマ |         名前          |    型    |  所有者
----------+-----------------------+----------+----------
 cdmv5    | attribute_definition  | テーブル | postgres
 cdmv5    | care_site             | テーブル | postgres
　　：
```
作成が完了したら「\\q」を入力し、psqlを終了します。
```
postgres=# \q
```
### ４．3．レコード投入方法のご紹介

CDMへのレコードの挿入は、タブ区切りファイル等をpsqlで取り込む事で行います。

OHDSI/ATHENA などで配布されている Vocaburaryファイルもタブ区切りの形式です。

FTP等によるサーバへのファイルの配置、またはGitHUBからの取得などで区切りファイルを任意の場所に配置します。

ファイルを配置後、対象サーバに データベース管理ユーザでログインし、psqlを起動します。

```
# psql
```
カレントスキーマを「cdmv5」へ変更します。
```
postgres=# set search_path to "cdmv5";
```
カレントスキーマが「cdmv5」に変更された事を確認します。
```
postgres=# select current_schema();
 current_schema
----------------
 cdmv5
(1 行)
```
ファイルを指定して、レコードの取込を行います。
```
postgres=#  COPY [テーブル名] FROM
 '[パス情報含むファイル名]'
 WITH DELIMITAR E'\t' NULL AS '' ;
```
具体的な例は下記の様になります。
```
postgres=#  COPY concept FROM
 '/home/postgres/CSVData/CONCEPT.csv'
 WITH DELIMITAR E'\t' NULL AS '' ;
```

CSVファイルの取り込みは頻繁に利用される方法ですので、操作を覚えておいてください。

------
## ５．Broadseaの構築

Broadseaコンテナの動作設定を行います。

### ５．１．Broadseaイメージの取得

root特権ユーザでログインし、BroadseaのDockerイメージをDockerHubより取得します。

Dockerが起動している状態で実行します。
```
# docker pull ohdsi/broadsea-webtools:latest
# docker pull ohdsi/broadsea-methodslibrary:latest
```

### ５．２．設定ファイルの編集

３．３．で取得した設定ファイルを編集します。

Broadsea/postgresqlディレクトリにあるファイル、docker-compose.ymlをBroadseaディレクトリにコピーします。

※３．３の例と同様に、取得したファイルを「/root/dockerwk」へ移動した前提として記載しています。
```
# cd /root/dockerwk/Broadsea/postgresql
# cp docker-compose.yml ../
```
コピーした docker-compose.yml を開き、内容を編集します。
```
# cd /root/dockerwk/Broadsea
# vi docker-compose.yml
```
編集内容は下記の通りです。
```
version: '2'

services:

  broadsea-methods-library:
    image: ohdsi/broadsea-methodslibrary
    ports:
      - "8787:8787"
      - "6311:6311"
    environment:
      - PASSWORD=mypass

  broadsea-webtools:
    image: ohdsi/broadsea-webtools
    ports:
      - "8080:8080"
    volumes:
     - .:/tmp/drivers/:ro
     - ./config-local.js:/usr/local/tomcat/webapps/atlas/js/config-local.js:ro
    environment:
      - WEBAPI_URL=http://[ホストのIPアドレス]:8080											→①
      - env=webapi-postgresql
      - datasource_driverClassName=org.postgresql.Driver
      - datasource_url=jdbc:postgresql://[ホストのIPアドレス]:5432/[データベース名]			→②
      - datasource.cdm.schema=[CDM用のスキーマ名]												→③
      - datasource.ohdsi.schema=[WebAPI用のスキーマ名]→④
      - datasource_username=[データベース接続ユーザ名]											→⑤
      - datasource_password=[データベース接続パスワード]											→⑥
      - spring.jpa.properties.hibernate.default_schema=[WebAPI用のスキーマ名]
      - spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
      - spring.batch.repository.tableprefix=ohdsi.BATCH_
      - flyway_datasource_driverClassName=org.postgresql.Driver
      - flyway_datasource_url=jdbc:postgresql://[ホストのIPアドレス]:5432/[データベース名]		→②
      - flyway_schemas=[WebAPI用のスキーマ名]													→④
      - flyway.placeholders.ohdsiSchema=[WebAPI用のスキーマ名]									→④
      - flyway_datasource_username=[データベース接続ユーザ名]									→⑤
      - flyway_datasource_password=[データベース接続パスワード]									→⑥
      - flyway.locations=classpath:db/migration/postgresql

```
- ①\[ホストのIPアドレス\]：Dockerが動作しているホストのIPアドレスを指定します。

- ②\[データベース名\]：CDMを作成したデータベース名を指定します。

- ③\[CDM用のスキーマ名\]：４．１．で作成したCDM用のスキーマ名を指定します。

- ④\[WebAPI用のスキーマ名\]：４．１．で作成したWebAPI用のスキーマ名を指定します。

- ⑤\[データベース接続ユーザ名\]：データベースへ接続するためのユーザ名を指定します。

- ⑥\[データベース接続パスワード\]：データベースへ接続するユーザのパスワードを指定します。


続いて、Broadseaディレクトリにあるファイル、config-local.jsを編集します。
```
# cd /root/dockerwk/Broadsea
# vi config-local.js
```

```
define([], function () {
        var configLocal = {};

        // clearing local storage otherwise source cache will obscure the override settings
        localStorage.clear();

        var getUrl = window.location;
        var baseUrl = getUrl.protocol + "//" + getUrl.host;

        // WebAPI
        configLocal.api = {
                name: 'OHDSI',
        //      url: baseUrl + '/WebAPI/'
                url: 'http://[ホストのIPアドレス]:8080/WebAPI/'			→①
        };

        configLocal.cohortComparisonResultsEnabled = false;
        configLocal.userAuthenticationEnabled = false;
        configLocal.plpResultsEnabled = false;

        return configLocal;
});

```
- ①\[ホストのIPアドレス\]：Dockerが動作しているホストのIPアドレスを指定します。


### ５．３．Achilles / CohortMethod のインストール

ブラウザから http://**\[サーバIPアドレス\]**:8787 へアクセスし、R Studio を起動します。ログイン画面が表示されたら R Studio のユーザ名・パスワードを入力し、ログインします。

![image-20210401162228118](./img\image-20210401162228118.png)

ログイン後、画面左下のコンソールより下記コマンドを入力し、 Achillesをインストールします。

![image-20210401162240604](../images/img/image-20210401162240604.png)
```
> install.packages("remotes")
> remotes::install_github("ohdsi/Achilles")
```
同様に、R Studio から下記コマンドを入力し、 CohortMethod パッケージをインストールします。
```
> remotes::install_github("OHDSI/CohortMethod")
```
※尚、CohortMethodインストール時に以下PKGも同時にインストールされます。

- DatabaseConnector (\= 4.0.0)
- Cyclops (\= 3.0.0) 
- FeatureExtraction (>= 3.0.0
- Andromeda (\= 0.3.0) 

続いて、DB接続ドライバ（JDBC)をインストールします。

インストール先を環境変数「DATABASECONNECTOR_JAR_FOLDER」に設定して実行します。

※ディレクトリは任意となります。この例では「/home/rstudio/jdbcDrivers」としています。
```
> Sys.setenv("DATABASECONNECTOR_JAR_FOLDER" = "/home/rstudio/jdbcDrivers")
> downloadJdbcDrivers("postgresql")
```
※DBがPostgreSQL以外の場合は対応するドライバをインストールしてください。下記でCohortMethodパッケージが正しくインストールされているか確認します。
```
> connectionDetails <- createConnectionDetails(dbms="postgresql",
	server="localhost/postgres",												→①
	user = "postgres",															→②
	password = "｛PostgreSQLインストール時に設定したパスワード｝")
> checkCmInstallation(connectionDetails)
```
①serverには\"｛ホスト名（またはホストのIPアドレス）｝/｛DB名｝\"を記載します。

　※host名はHADES環境とDBを同一端末で構築した場合は\"localhost\"と記載します。

　※DB名にはpostgresインストール時にデフォルトで作成される\"postgres\"を記載します。

②Userには上記デフォルトで作成されるDBのユーザ名\"postgres\"を記載します。上記を実行し、下記が表示されていればインストール成功です。

![image-20210401162747844](./img\image-20210401162747844.png)

※createConnectionDetails関数を呼び出せない場合は、下記コマンドでdatabaseConncection関数を呼び出してから再度、実行してください。

```
> library(databaseConncection)
```
### ５．４．初回起動と起動後の追加設定

設定ファイルの編集、及びインストールが完了したら、下記コマンドを実行し、コンテナを起動します。

コマンドは、編集したdocker-compose.yml、config-local.jsがあるディレクトリで実行します。
```
# cd /root/dockerwk/Broadsea
# docker-compose up -d
Starting broadseamaster_broadsea-methods-library_1 ... 
Starting broadseamaster_broadsea-methods-library_1
Starting broadseamaster_broadsea-webtools_1 ... 
Starting broadseamaster_broadsea-webtools_1 ... Done
```
コンテナ起動後、ブラウザから **http://\[ホストのIPアドレス\]:8080/atlas/** へアクセスします。

![image-20210401162822595](./img\image-20210401162822595.png)

ATLASの画面が表示されたら、一度画面を閉じます。

画面を閉じたらroot特権ユーザでログインし、コンテナを停止します。

「docker ps -a」でコンテナの状態を表示、コンテナIDを確認し、対象のコンテナを「docker stop コンテナID」で停止します。
```
# docker ps -a
CONTAINER_ID IMAGE COMMAND ・・・
84ed9b76a32a ohdsi/broadsea-webtools "/usr/bin/supervisord" ・・・
# docker stop 84ed9b76a32a
```
データベース管理ユーザでログインし、psqlを起動します。
```
# psql
```
カレントスキーマを WebAPI用スキーマ（この例では ohdsi） へ変更します。
```
postgres=# set search_path to \"ohdsi\";
```
変更後のカレントスキーマ確認します。
```
postgres=# select current_schema();
 current_schema
----------------
 ohdsi
(1 行)
```
カレントスキーマ確認後にテーブル一覧を表示すると、初回起動時にいくつかのテーブルが作成されている事が分かります。

作成されたテーブルの中に sourceとsource_daimon がある事を確認します。
```
postgres=# \dt
                       リレーションの一覧
 スキーマ |             名前              |    型    |  所有者
----------+-------------------------------+----------+----------
 ohdsi    | analysis_generation_info      | テーブル | postgres
 ohdsi    | batch_job_execution           | テーブル | postgres
 ohdsi    | batch_job_execution_context   | テーブル | postgres
　　　　　　　　：
 ohdsi    | source                        | テーブル | postgres
 ohdsi    | source_daimon                 | テーブル | postgres
 ohdsi    | user_import_job               | テーブル | postgres
 ohdsi    | user_import_job_weekdays      | テーブル | postgres

```
次のSQLを実行し、WebAPIの設定情報を登録します。
```
postgres=# INSERT INTO ohdsi.source 
 (source_id, source_name, source_key, source_connection, source_dialect)
 VALUES (1, 'OHDSI CDM V5 Database', 'OHDSI-CDMV5', 
 'jdbc:postgresql://[ﾎｽﾄのIPｱﾄﾞﾚｽ]:5432/[ﾃﾞｰﾀﾍﾞｰｽ名]?user=[ﾕｰｻﾞ名]&password=[ﾊﾟｽﾜｰﾄﾞ]',
 'postgresql');
```
※CDM ドメインの設定
```
postgres=# INSERT INTO ohdsi.source_daimon
 (source_daimon_id, source_id, daimon_type, table_qualifier, priority)
 VALUES (1, 1, 0, '[CDMスキーマ名]', 2);
```
※VOCABULARY ドメインの設定
```
postgres=# INSERT INTO ohdsi.source_daimon
 (source_daimon_id, source_id, daimon_type, table_qualifier, priority) 
 VALUES (2, 1, 1, '[CDMスキーマ名]', 2);
```
※RESULTS ドメインの設定
```
postgres=# INSERT INTO ohdsi.source_daimon
 (source_daimon_id, source_id, daimon_type, table_qualifier, priority) 
 VALUES (3, 1, 2, '[WebAPIスキーマ名]', 2);
```
※EVIDENCE ドメインの設定
```
postgres=# INSERT INTO ohdsi.source_daimon
 ( source_daimon_id, source_id, daimon_type, table_qualifier, priority)
 VALUES (4, 1, 3, '[WebAPIスキーマ名]', 2);
```
SQL実行後、root特権ユーザにて、コンテナの再起動を行います。
```
# cd /root/dockerwk/Broadsea
# docker-compose up -d
Starting broadseamaster_broadsea-methods-library_1 ... 
Starting broadseamaster_broadsea-methods-library_1
Starting broadseamaster_broadsea-webtools_1 ... 
Starting broadseamaster_broadsea-webtools_1 ... Done
```

### ５．５．Achillesの実行

ブラウザから http://**\[サーバIPアドレス\]**:8787 へアクセスし、R Studio を起動し、ログインします。

![image-20210401162910324](../images/img/image-20210401162910324.png)

ログイン後、画面左下のConsole部分に下記のコマンドを入力し、Achilles を起動します。
```
> library(Achilles)
> Sys.setenv("DATABASECONNECTOR_JAR_FOLDER"="/home/rstudio/jdbcDrivers")
> connectionDetails <- createConnectionDetails( 
	dbms="postgresql",
	server="[サーバIPアドレス]/[データベース名]",				→①
	user="[データベース接続ユーザ名]",							→②
	password="[データベース接続パスワード]",						→③
	port="5432")                                             
> achilles(connectionDetails,                              
	cdmDatabaseSchema = "[CDMのスキーマ]",					→④
	resultsDatabaseSchema="[WebAPIのスキーマ]",				→⑤
	vocabDatabaseSchema = "[CDMのスキーマ]", numThreads = 1,	→⑥
	sourceName = "[テーブル設定内容]",							→⑦
	cdmVersion = "5.3.1", 
	runHeel = FALSE, 
	runCostAnalysis = FALSE)
```

①\[ホストのIPアドレス\]：Dockerが動作しているホストのIPアドレスを指定します。

　\[データベース名\]：CDMを作成したデータベース名を指定します。

②\[データベース接続ユーザ名\]：データベースへ接続するためのユーザ名を指定します。

③\[データベース接続パスワード\]：データベースへ接続するユーザのパスワードを指定します。

④\[CDM用のスキーマ名\]：５．４．でCDMドメインに指定した内容を設定します。

⑤\[WebAPI用のスキーマ名\]：５．４．でRESULTドメインに指定した内容を設定します。

⑥\[CDM用のスキーマ名\]：５．４．でVOCABULARYドメインに指定した内容を設定します。

⑦\[テーブル設定内容\]：５．４．でsouceテーブルに登録した「source_name」の内容を設定します。



以上で環境構築の手順は完了となります。

------
## ６．ATLAS画面表示

「docker ps -a」コマンドを実行し、コンテナが起動されている事を確認します。
```
# docker ps -a
 CONTAINER_ID  IMAGE                   COMMAND                ・・・   STATUS
 84ed9b76a32a  ohdsi/broadsea-webtools "/usr/bin/supervisord" ・・・   UP 9hours
```

コンテナが起動されている事を確認し、ブラウザから **http://\[ホストのIPアドレス\]:8080/atlas/** へアクセスして下さい。

![image-20210401163325567](../images/img/image-20210401163325567.png)
