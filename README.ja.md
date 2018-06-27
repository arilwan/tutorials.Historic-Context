[![FIWARE Banner](https://fiware.github.io/tutorials.Historic-Context/img/fiware.png)](https://www.fiware.org/developers)

このチュートリアルは、コンテキスト・データをサードパーティのデータベースに保存してコンテキストの履歴ビューを作成するために使用する汎用イネーブラである、[FIWARE Cygnus](http://fiware-cygnus.readthedocs.io/en/latest/) の概要です。このチュートリアルでは、[前のチュートリアル](https://github.com/Fiware/tutorials.IoT-Agent)で接続した IoT センサをアクティブにし、これらのセンサからの測定値をデータベースに保存してさらに分析します。

このチュートリアルでは、全体で [cUrl](https://ec.haxx.se/) コマンドを使用していますが、[Postman documentation](http://fiware.github.io/tutorials.Historic-Context/) も利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/4824d3171f823935dcab)


# 内容

- [データの永続性](#data-persistence)
- [アーキテクチャ](#architecture)
- [前提条件](#prerequisites)
  * [Docker と Docker Compose](#docker-and-docker-compose)
  * [Cygwin for Windows](#cygwin-for-windows)
- [起動](#start-up)
- [Mongo DB - コンテキスト・データをデータベースに永続化](#mongo-db---persisting-context-data-into-a-database)
  * [Mongo DB - データベース・サーバの設定](#mongo-db---database-server-configuration)
  * [Mongo DB - Cygnus の設定](#mongo-db---cygnus-configuration)
  * [Mongo DB - 起動](#mongo-db---start-up)
    + [Cygnus サービスの健全性をチェック](#checking-the-cygnus-service-health)
    + [コンテキスト・データの生成](#generating-context-data)
    + [コンテキスト変更のサブスクライブ](#subscribing-to-context-changes)
  * [Mongo DB - データベースからデータを読み込む](#mongo-db----reading-data-from-a-database)
    + [Mongo DB サーバ上で利用可能なデータベースを表示](#show-available-databases-on-the-mongo-db-server)
    + [サーバから履歴コンテキストを読み込む](#read-historical-context-from-the-server)
- [PostgreSQL - コンテキスト・データをデータベースに永続化](#postgresql---persisting-context-data-into-a-database)
  * [PostgreSQL - データベース・サーバの設定](#postgresql---database-server-configuration)
  * [PostgreSQL - Cygnusの設定](#postgresql---cygnus-configuration)
  * [PostgreSQL - 起動](#postgresql---start-up)
    + [Cygnusサービスの健全性をチェック](#checking-the-cygnus-service-health-1)
    + [コンテキスト・データの生成](#generating-context-data-1)
    + [コンテキスト変更のサブスクライブ](#subscribing-to-context-changes-1)
  * [PostgreSQL - データベースからデータを読み込む](#postgresql---reading-data-from-a-database)
    + [PostgreSQL サーバ上で利用可能なデータベースを表示](#show-available-databases-on-the-postgresql-server)
    + [PostgreSQL サーバから履歴コンテキストを読み込む](#read-historical-context-from-the-postgresql-server)
- [MySQL - コンテキスト・データをデータベースに永続化](#mysql---persisting-context-data-into-a-database)
  * [MySQL - データベース・サーバの設定](#mysql---database-server-configuration)
  * [MySQL - Cygnus の設定](#mysql---cygnus-configuration)
  * [MySQL - 起動](#mysql---start-up)
    + [Cygnus サービスの健全性をチェック](#checking-the-cygnus-service-health-2)
    + [コンテキスト・データの生成](#generating-context-data-2)
    + [コンテキスト変更のサブスクライブ](#subscribing-to-context-changes-2)
  * [MySQL - データベースからデータを読み込む](#mysql---reading-data-from-a-database)
    + [MySQL サーバ上で利用可能なデータベースを表示](#show-available-databases-on-the-mysql-server)
    + [MySQL サーバから履歴コンテキストを読み込む](#read-historical-context-from-the-mysql-server)
- [マルチ・エージェント - 複数のデータベースへのコンテキスト・データの永続化](#multi-agent---persisting-context-data-into-a-multiple-databases)
  * [マルチ・エージェント - 複数のデータベースのための Cygnus 設定](#multi-agent---cygnus-configuration-for-multiple-databases)
  * [マルチ・エージェント - 起動](#multi-agent---start-up)
    + [Cygnus サービスの健全性をチェック](#checking-the-cygnus-service-health-3)
    + [コンテキスト・データの生成](#generating-context-data-3)
    + [コンテキスト変更のサブスクライブ](#subscribing-to-context-changes-3)
  * [マルチ・エージェント - 永続化データの読み込み](#multi-agent---reading-persisted-data)
- [次のステップ](#next-steps)


<a name="data-persistence"></a>
# データの永続性

> "History will be kind to me for I intend to write it."
>
> — Winston Churchill


以前のチュートリアルでは、実世界の状態の測定値を提供する IoT センサと、**Orion Context Broker** と **IoT Agent** の 2つの FIWARE コンポーネントを紹介しました。このチュートリアルでは、新しいデータ永続化コンポーネント FIWARE **Cygnus** を紹介します。

これまでのシステムは、現在のコンテキストを扱うように構築されています。つまり、与えられた瞬間に現実のオブジェクトの状態を定義するデータ・エンティティを保持しています。

この定義から、コンテキストはシステムの**現在**の状態のみに関心があります。システムの履歴状態を報告するのは既存コンポーネントの責任ではありません。コンテキストは、各センサが Context Broker に送信した最後の測定値に基づいています。

これを行うためには、コンテキストが更新されるたびに、状態の変化をデータベースに永続化するために、既存のアーキテクチャを拡張する必要があります。

過去のコンテキスト・データを永続化することは、大規模なデータ分析に役立ちます。傾向を発見するために使用することができます。また、データをサンプリングして集約して、外部データ測定の影響を取り除くこともできます。ただし、各スマート・ソリューションでは、各エンティティ型の重要性が異なり、エンティティと属性を異なるレートでサンプリングする必要があります。

コンテキスト・データを使用するためのビジネス要件はアプリケーションごとに異なりますので、履歴データの永続化のための標準的な使用例は1つもありません。それぞれの状況は固有のものであり、1つのサイズがすべてに適合するケースではありません。したがって、Context Broker に履歴的なコンテキスト・データの永続性を与えることではなく、この役割は、構成可能な別のコンポーネント **Cygnus** に分離されています。

ご想像のとおり、オープンソース・プラットフォームの一部としての **Cygnus** は、データの永続化に使用されるデータベースに関する技術に依存しません。使用するデータベースは、ビジネス・ニーズに応じて異なります。

ただし、この柔軟性を提供するにはコストがかかります。システムの各部分を個別に設定し、必要な最小限のデータだけを通知するように通知を設定する必要があります。


#### デバイス・モニタ

このチュートリアルの目的のために、一連のダミー IoT デバイスが作成され、Context Broker に接続されます。使用しているアーキテクチャとプロトコルの詳細は、[IoT Sensors チュートリアル](https://github.com/Fiware/tutorials.IoT-Sensors)にあります。各デバイスの状態は、次の UltraLight デバイス・モニタの Web ページで確認できます : `http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Historic-Context/img/device-monitor.png)



<a name="architecture"></a>
# アーキテクチャ

このアプリケーションは、[前のチュートリアル](https://github.com/Fiware/tutorials.IoT-Agent/)で作成したコンポーネントと ダミー IoT デバイスをベースにしています。3つの FIWARE コンポーネントを使用します。[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/), [IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/), コンテキスト・データをデータベースに永続化するための [Cygnus Generic Enabler](http://fiware-cygnus.readthedocs.io/en/latest/) を導入しました。Orion Context Broker と IoT Agent の両方が [MongoDB](https://www.mongodb.com/) テクノロジを利用して保持している情報の永続性を維持しています。**MySQL**, **PostgreSQL**, **MongoDB** データベースのいずれかで、履歴コンテキスト・データを永続化します。

したがって、全体のアーキテクチャーは以下の要素で構成されます :

* 3つの **FIWARE 汎用イネーブラー** :
  * FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/)は、[NGSI](http://fiware.github.io/specifications/ngsiv2/latest/) を使用してリクエストを受信します 
  * FIWARE [IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/) は、[Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) フォーマットのダミー IoT デバイスからノース・バウンドの測定値を受信し、Context Broker がコンテキスト・エンティティの状態を変更するための [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) リクエストに変換します
 * FIWARE Cygnus はコンテキストの変更をサブスクライブし、データベース (**MySQL** , **PostgreSQL** , **Mongo-DB**) に保持します。
* 以下の**データベース**の 1つ、2つまたは3つ :
  * 基礎となる [MongoDB](https://www.mongodb.com/) データベース :
    + **Orion Context Broker** が、データ・エンティティなどのコンテキスト・データ情報を保持し、サブスクリプション、レジストレーションするために使用します
    + **IoT Agent** がデバイスの URL やキーなどのデバイス情報を保持するために使用します
    + 履歴コンテキスト・データを保持するためのデータ・シンクとして潜在的に使用します
  * 追加の [PostgreSQL](https://www.postgresql.org/) データベース :
    + 履歴データを保持するためのデータ・シンクとして潜在的に使用します
  * 追加の [MySQL](https://www.mysql.com/) データベース :
    + 履歴データを保持するためのデータ・シンクとして潜在的に使用します
* 3つの**コンテキストプロバイダ** :
  * **在庫管理フロントエンド**は、このチュートリアルで使用していません。これは以下を行います :
    + 店舗情報を表示し、ユーザーがダミー IoT デバイスと対話できるようにします
    + 各店舗で購入できる商品を表示します
    + ユーザが製品を購入して在庫数を減らすことを許可します
  * HTTP 上で動作する [Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) プロトコルを使用して、[ダミー IoT デバイス](https://github.com/Fiware/tutorials.IoT-Sensors)のセットとして機能する Web サーバ
  * このチュートリアルでは、**コンテキスト・プロバイダのNGSI proxy** は使用しません。これは以下を行います :
    + [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) を使用してリクエストを受信します
    + 独自の API を独自のフォーマットで使用して、公開されているデータ・ソースへのリクエストを行います
    + [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) 形式でコンテキスト・データOrion Context Broker に返します

要素間のすべての対話は HTTP リクエストによって開始されるため、エンティティはコンテナ化され、公開されたポートから実行されます。

チュートリアルの各セクションの具体的なアーキテクチャについては、以下で説明します。


<a name="prerequisites"></a>
# 前提条件

<a name="docker-and-docker-compose"></a>
## Dokcer と Docker Compose

物事を単純にするために、両方のコンポーネントが [Docker](https://www.docker.com) を使用して実行されます。**Docker** は、さまざまコンポーネントをそれぞれの環境に分離することを可能にするコンテナ・テクノロジです。

* Docker Windows にインストールするには、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってください
* Docker Mac にインストールするには、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
* Docker Linux にインストールするには、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行するためのツールです。[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Getting-Started/master/docker-compose.yml) ファイルは、アプリケーションのために必要なサービスを設定する使用されています。つまり、すべてのコンテナ・サービスは1つのコマンドで呼び出すことができます。Docker Compose は、デフォルトで Docker for Windows とDocker for Mac の一部としてインストールされますが、Linux ユーザは[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う必要があります。

<a name="cygwin-for-windows"></a>
## Cygwin for Windows

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは [cygwin](http://www.cygwin.com/) をダウンロードして、Windows 上の Linux ディストリビューションと同様のコマンドライン機能を提供する必要があります。


<a name="start-up"></a>
# 起動

開始する前に、必要なDockerイメージをローカルで取得または構築しておく必要があります。リポジトリを複製し、以下のコマンドを実行して必要なイメージを作成してください :

```console
git clone git@github.com:Fiware/tutorials.Historic-Context.git
cd tutorials.Historic-Context

./services create
``` 

>**注** `context-provider` イメージは、まだ Docker Hub にプッシュされていません。続行する前に Docker ソースをビルドできないと、次のエラーが発生します :
>
>```
>Pulling context-provider (fiware/cp-web-app:latest)...
>ERROR: The image for the service you're trying to recreate has been removed.
>```


その後、リポジトリ内で提供される [services](https://github.com/Fiware/tutorials.Historic-Context/blob/master/services) の Bash スクリプトを実行することによって、コマンドラインからすべてのサービスを初期化できます :

```console
./services <command>
``` 

ここで、`<command>` は、有効にしたいデータベースによって異なります。このコマンドは、前のチュートリアルのシード・データをインポートし、起動時にダミー IoT センサをプロビジョニングします。

>:information_source: **注:** クリーンアップをやり直したい場合は、次のコマンドを使用して再起動することができます :
>
>```console
>./services stop
>``` 
>


<a name="mongo-db---persisting-context-data-into-a-database"></a>
# Mongo DB - コンテキスト・データをデータベースに永続化

MongoDB テクノロジーを使用して、履歴コンテキスト・データを永続化することは、Orion Context Broker と IoT Agent に関連するデータを保持するために既に MongoDB インスタンスを使用しているため、比較的簡単に構成できます。MongoDB インスタンスは標準 `27017` ポートをリッスンしており、全体のアーキテクチャは以下のようになります :

![](https://fiware.github.io/tutorials.Historic-Context/img/cygnus-mongo.png)

<a name="mongo-db---database-server-configuration"></a>
## Mongo DB - データベース・サーバの設定

```yaml
  mongo-db:
    image: mongo:3.6
    hostname: mongo-db
    container_name: db-mongo
    ports:
        - "27017:27017"
    networks:
        - default
    command: --bind_ip_all --smallfiles
```

<a name="mongo-db---cygnus-configuration"></a>
## Mongo DB - Cygnus の設定

```yaml
  cygnus:
    image: fiware/cygnus-ngsi:latest
    hostname: cygnus
    container_name: fiware-cygnus
    depends_on:
        - mongo-db
    networks:
        - default
    expose:
        - "5080"
    ports:
        - "5050:5050"
        - "5080:5080"
    environment:
        - "CYGNUS_MONGO_HOSTS=mongo-db:27017"
        - "CYGNUS_LOG_LEVEL=DEBUG"
        - "CYGNUS_SERVICE_PORT=5050"
        - "CYGNUS_API_PORT=5080"
```


`cygnus` コンテナは、2つのポートでリッスンしています :

* Cygnus のサブスクリプション・ポート : `5050` は、サービスが Orion context broker からの通知をリッスンするポートです
* Cygnus の管理ポート : `5080`は、純粋にチュートリアル・アクセスのために公開されているため、cUrl または Postman は同じネットワークの一部ではなくプロビジョニング・コマンドを作成できます


`cygnus` コンテナは、示されているように環境変数によって制御できます :

| キー	                        | 値           | 説明      |
|-------------------------------|--------------|-----------|
|CYGNUS_MONGO_HOSTS         |`mongo-db:27017` |  Cygnus が履歴コンテキスト・データを保持するために接続する Mongo-DB サーバのカンマ区切りリスト |
|CYGNUS_LOG_LEVEL               |`DEBUG`       | Cygnus のログレベル |
|CYGNUS_SERVICE_PORT            |`5050`        | コンテキスト・データの変更をサブスクライブするときに Cygnus がリッスンする通知ポート|
|CYGNUS_API_PORT                |`5080`        | Cygnus が操作上の理由でリッスンするポート |

<a name="mongo-db---start-up"></a>
## Mongo DB - 起動

**Mongo DB** データベースのみでシステムを起動するには、次のコマンドを実行します :

```console
./services mongodb
``` 

<a name="checking-the-cygnus-service-health"></a>
### Cygnus サービスの健全性をチェック
 
Cygnus が動作したら、公開されている `CYGNUS_API_PORT` ポートへの HTTP リクエストを行うことでステータスを確認できます。レスポンスがブランクの場合、これは通常、Cygnus が実行されていないか、別のポートでリッスンしているためです。

#### :one: リクエスト :

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### レスポンス :

レスポンスは次のようになります :

```json
{
    "success": "true",
    "version": "1.8.0_SNAPSHOT.ed50706880829e97fd4cf926df434f1ef4fac147"
}
```

>**トラブルシューティング** : レスポンスが空白の場合はどうなりますか？
>
> * Dokcer コンテナが動作していることを確認するには :
>
>```bash
>docker ps
>```
>
>いくつかのコンテナが走っているのを確認してください。`cygnus` が実行されていない場合は、必要に応じてコンテナを再起動できます。


<a name="generating-context-data"></a>
### コンテキスト・データの生成

このチュートリアルでは、コンテキストが定期的に更新されるシステムを監視する必要があります。ダミー IoT センサを使用してこれを行うことができます。`http://localhost:3000/device/monitor` でデバイス・モニタのページを開き、**スマート・ドア**のロックを解除し、**スマート・ランプ**をオンにします。これは、ドロップ・ダウン・リストから適切なコマンドを選択し、`send` ボタンを押すことによって行うことができます。デバイスからの測定値のストリームは、同じページに表示されます :

![](https://fiware.github.io/tutorials.Historic-Context/img/door-open.gif)



<a name="subscribing-to-context-changes"></a>
### コンテキスト変更のサブスクライブ

動的コンテキスト・システムが起動したら、**Cygnus** にコンテキストの変更を通知する必要があります。

これは、Orion Context Broker の `/v2/subscription` エンドポイントに POST リクエストを行うことによって行われます。

* `fiware-service` と `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続されている IoT センサからの測定値だけをリッスンするために使用します。センサがこれらの設定を使用してプロビジョニングされているためです
* リクエスト・ボディの `idPattern` は、すべてのコンテキスト・データの変更が Cygnus に通知されるようにします
* 通知 `url` は設定された `CYGNUS_API_PORT` と一致する必要があります
* Cygnus は現在、古い NGSI v1 形式の通知のみを受け付けているため、`attrsFormat=legacy` が必要です
* `throttling` 値は、変更がサンプリングされる割合を定義します

#### :two: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus of all context changes",
  "subject": {
    "entities": [
      {
        "idPattern": ".*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

ご覧のとおり、コンテキスト・データを保持するために使用されるデータベースは、サブスクリプションの詳細に影響を与えません。各データベースで同じです。レスポンスは **201 - Created** です。




<a name="mongo-db----reading-data-from-a-database"></a>
## Mongo DB - データベースからデータを読み込む

コマンドラインから `mongo-db` データを読み込むには、コマンドライン・プロンプトを表示するために、`mongo` ツールにアクセスして `mongo` イメージのインタラクティブなインスタンスを実行する必要があります :

```console
docker run -it --network fiware_default  --entrypoint /bin/bash mongo
```

次のようにコマンドラインを使用して、実行中の `mongo-db` データベースにログインできます :

```bash
mongo --host mongo-db
```

<a name="show-available-databases-on-the-mongo-db-server"></a>
### Mongo DB サーバ上で利用可能なデータベースを表示

使用可能なデータベースのリストを表示するには、次のように文を実行します :

#### クエリ :

```
show dbs
```

#### 結果 :

```
admin          0.000GB
iotagentul     0.000GB
local          0.000GB
orion          0.000GB
orion-openiot  0.000GB
sth_openiot    0.000GB
```

結果には、デフォルトで MongoDB によって設定されている、2つのデータベース `admin` と `local` があり、FIWARE プラットフォームによって作成された 4つのデータベースも含まれています。Orion Context Broker は、それぞれの `fiware-service` に 2つのデータベース・インスタンスを作成しました。

- IoT デバイスのエンティティは、`openiot` `fiware-service` ヘッダを使用して作成され、別々に保持されるのに対し、ストア・エンティティは、`fiware-service` を定義することなく作成され、したがって `orion` データベース内に保持されます。 IoT Agent は、IoT センサ・データ`iotagentul` という別の **MongoDB** データベースに保持するように初期化されました。

Orgn Context Broker に Cygnus をサブスクリプションした結果、`sth_openiot` という新しいデータベースが作成されました。履歴コンテキストを保持する **Mongo DB** データベースのデフォルト値は、`sth_` プレフィックスの後ろに `fiware-service` ヘッダが続くため、`sth_openiot` は IoT デバイスの履歴コンテキストを保持します。


<a name="read-historical-context-from-the-server"></a>
### サーバから履歴コンテキストを読み込む

#### クエリ :

```
use sth_openiot
show collections
```

#### 結果 :

```
switched to db sth_openiot

sth_/_Door:001_Door
sth_/_Door:001_Door.aggr
sth_/_Lamp:001_Lamp
sth_/_Lamp:001_Lamp.aggr
sth_/_Motion:001_Motion
sth_/_Motion:001_Motion.aggr
```

`sth_openiot` 内を見ると、一連のテーブルが作成されていることがわかります。各テーブルの名前は、`sth_` プレフィックスの後に `fiware-servicepath` ヘッダとそれに続くエンティティ id が続きます。各エンティティごとに2つのテーブルが作成されます。`.aggr` テーブルには、後でチュートリアルでアクセスするいくつかの集計データが格納されています。生データは、`.aggr` サフィックスなしのテーブルで見ることができます。

履歴データは各テーブル内のデータを確認するで見ることができます。デフォルトでは、各行には単一の属性のサンプリング値が含まれます。

#### クエリ :

```
db["sth_/_Door:001_Door"].find().limit(10)
```

#### 結果 :

```
{ "_id" : ObjectId("5b1fa48630c49e0012f7635d"), "recvTime" : ISODate("2018-06-12T10:46:30.897Z"), "attrName" : "TimeInstant", "attrType" : "ISO8601", "attrValue" : "2018-06-12T10:46:30.836Z" }
{ "_id" : ObjectId("5b1fa48630c49e0012f7635e"), "recvTime" : ISODate("2018-06-12T10:46:30.897Z"), "attrName" : "close_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
{ "_id" : ObjectId("5b1fa48630c49e0012f7635f"), "recvTime" : ISODate("2018-06-12T10:46:30.897Z"), "attrName" : "lock_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
{ "_id" : ObjectId("5b1fa48630c49e0012f76360"), "recvTime" : ISODate("2018-06-12T10:46:30.897Z"), "attrName" : "open_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
{ "_id" : ObjectId("5b1fa48630c49e0012f76361"), "recvTime" : ISODate("2018-06-12T10:46:30.836Z"), "attrName" : "refStore", "attrType" : "Relationship", "attrValue" : "Store:001" }
{ "_id" : ObjectId("5b1fa48630c49e0012f76362"), "recvTime" : ISODate("2018-06-12T10:46:30.836Z"), "attrName" : "state", "attrType" : "Text", "attrValue" : "CLOSED" }
{ "_id" : ObjectId("5b1fa48630c49e0012f76363"), "recvTime" : ISODate("2018-06-12T10:45:26.368Z"), "attrName" : "unlock_info", "attrType" : "commandResult", "attrValue" : " unlock OK" }
{ "_id" : ObjectId("5b1fa48630c49e0012f76364"), "recvTime" : ISODate("2018-06-12T10:45:26.368Z"), "attrName" : "unlock_status", "attrType" : "commandStatus", "attrValue" : "OK" }
{ "_id" : ObjectId("5b1fa4c030c49e0012f76385"), "recvTime" : ISODate("2018-06-12T10:47:28.081Z"), "attrName" : "TimeInstant", "attrType" : "ISO8601", "attrValue" : "2018-06-12T10:47:28.038Z" }
{ "_id" : ObjectId("5b1fa4c030c49e0012f76386"), "recvTime" : ISODate("2018-06-12T10:47:28.081Z"), "attrName" : "close_status", "attrType" : "commandStatus", "attrValue" : "UNKNOWN" }
```

通常の **Mongo-DB** クエリ構文は、適切なフィールドと値をフィルタリングするために使用できます。たとえば、`id=Motion:001_Motion` の**モーション・センサ**が蓄積しているレートを読み取るには、次のようにクエリを作成します :

#### クエリ :

```
db["sth_/_Motion:001_Motion"].find({attrName: "count"},{_id: 0, attrType: 0, attrName: 0 } ).limit(10)
```

#### 結果 :

```
{ "recvTime" : ISODate("2018-06-12T10:46:18.756Z"), "attrValue" : "8" }
{ "recvTime" : ISODate("2018-06-12T10:46:36.881Z"), "attrValue" : "10" }
{ "recvTime" : ISODate("2018-06-12T10:46:42.947Z"), "attrValue" : "11" }
{ "recvTime" : ISODate("2018-06-12T10:46:54.893Z"), "attrValue" : "13" }
{ "recvTime" : ISODate("2018-06-12T10:47:00.929Z"), "attrValue" : "15" }
{ "recvTime" : ISODate("2018-06-12T10:47:06.954Z"), "attrValue" : "17" }
{ "recvTime" : ISODate("2018-06-12T10:47:15.983Z"), "attrValue" : "19" }
{ "recvTime" : ISODate("2018-06-12T10:47:49.090Z"), "attrValue" : "23" }
{ "recvTime" : ISODate("2018-06-12T10:47:58.112Z"), "attrValue" : "25" }
{ "recvTime" : ISODate("2018-06-12T10:48:28.218Z"), "attrValue" : "29" }
```

MongoDB クライアントを離れて、インタラクティブ・モードを終了するには、次のコマンドを実行します :

```console
exit
```

```console
exit
```



<a name="postgresql---persisting-context-data-into-a-database"></a>
# PostgreSQL - コンテキスト・データをデータベースに永続化

履歴データ**PostgreSQL** などの代替データベースに保存するには、PostgreSQL サーバをホストする追加のコンテナが必要です。このデータのデフォルトの Docker イメージを使用できます。PostgreSQL インスタンスは標準 `5432` ポートをリッスンしており、全体のアーキテクチャは以下のようになります :

![](https://fiware.github.io/tutorials.Historic-Context/img/cygnus-postgres.png)

MongoDB コンテナには、Orion Context Broker と IoT Agent に関連するデータを保持する必要があるため、2つのデータベースを持つシステムが用意されています。


<a name="postgresql---database-server-configuration"></a>
## PostgreSQL - データベース・サーバの設定

```yaml
  postgres-db:
      image: postgres:latest
      hostname: postgres-db
      container_name: db-postgres
      expose:
        - "5432"
      ports:
        - "5432:5432"
      networks:
        - default
      environment:
        - "POSTGRES_PASSWORD=password"
        - "POSTGRES_USER=postgres"
        - "POSTGRES_DB=postgres"

```

`postgres-db` は、コンテナは単一のポートでリッスンします :

* ポート `5432` は PostgreSQL サーバのデフォルト・ポートです。必要に応じてデータベースのデータを表示する `pgAdmin4` ツールを実行できるように公開されています

`postgres-db` は、次に示されているように環境変数によって制御されます :

| キー            | 値       | 説明                          |
|-----------------|----------|-------------------------------|
|POSTGRES_PASSWORD|`password`| PostgreSQL データベース・ユーザのパスワード |
|POSTGRES_USER    |`postgres`| PostgreSQL データベース・ユーザのユーザ名 |
|POSTGRES_DB      |`postgres`| PostgreSQL データベースの名前 | 


<a name="postgresql---cygnus-configuration"></a>
## PostgreSQL - Cygnus の設定

```yaml
  cygnus:
    image: fiware/cygnus-ngsi:latest
    hostname: cygnus
    container_name: fiware-cygnus
    networks:
        - default
    depends_on:
        - postgres-db
    expose:
        - "5080"
    ports:
        - "5050:5050"
        - "5080:5080"
    environment:
        - "CYGNUS_POSTGRESQL_HOST=postgres-db"
        - "CYGNUS_POSTGRESQL_PORT=5432"
        - "CYGNUS_POSTGRESQL_USER=postgres" 
        - "CYGNUS_POSTGRESQL_PASS=password" 
        - "CYGNUS_LOG_LEVEL=DEBUG"
        - "CYGNUS_SERVICE_PORT=5050"
        - "CYGNUS_API_PORT=5080"
        - "CYGNUS_POSTGRESQL_ENABLE_CACHE=true"
```

`cygnus` コンテナは、2つのポートでリッスンしています :

* Cygnus のサブスクリプション・ポート : `5050` は、サービスが Orion Context Broker からの通知をリッスンするポートです
* Cygnus の管理ポート : `5080` は純粋にチュートリアルのアクセスのために公開されているため、cUrl または Postman は同じネットワークの一部ではなくプロビジョニング・コマンドを作成できます

`cygnus` コンテナは、次のように環境変数によって制御されます :

| キー                          | 値           | 説明      |
|-------------------------------|--------------|-----------|
|CYGNUS_POSTGRESQL_HOST         |`postgres-db` | 履歴コンテキスト・データの永続化に使用される PostgreSQL サーバのホスト名 |
|CYGNUS_POSTGRESQL_PORT         |`5432`        | PostgreSQL サーバがコマンドをリッスンするために使うポート|
|CYGNUS_POSTGRESQL_USER         |`postgres`    | PostgreSQL データベース・ユーザのユーザ名 | 
|CYGNUS_POSTGRESQL_PASS         |`password`    | PostgreSQL データベース・ユーザのパスワード |
|CYGNUS_LOG_LEVEL               |`DEBUG`       | Cygnus のログレベル|
|CYGNUS_SERVICE_PORT            |`5050`        | コンテキスト・データの変更をサブスクライブするときに Cygnus がリッスンする通知ポート |
|CYGNUS_API_PORT                |`5080`        | Cygnus が操作上の理由でリッスンするポート |
|CYGNUS_POSTGRESQL_ENABLE_CACHE |`true`        | PostgreSQL 設定内でキャッシングを有効にするためのスイッチ |

<a name="postgresql---start-up"></a>
## PostgreSQL - 起動

**PostgreSQL** データベースを使用してシステムを起動するには、次のコマンドを実行します :

```console
./services postgres
``` 

<a name="checking-the-cygnus-service-health-1"></a>
### Cygnus サービスの健全性をチェック
 
Cygnus が動作したら、公開されている `CYGNUS_API_PORT` ポートへの HTTP リクエストを行うことでステータスを確認できます。レスポンスがブランクの場合、これは通常、Cygnus が実行されていないか、別のポートでリッスンしているためです。

#### :three: リクエスト :

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### レスポンス :

レスポンスは次のようになります :

```json
{
    "success": "true",
    "version": "1.8.0_SNAPSHOT.ed50706880829e97fd4cf926df434f1ef4fac147"
}
```

>**トラブルシューティング** : レスポンスが空白の場合はどうなりますか？
>
> * Dokcer コンテナが動作していることを確認するには
>
>```bash
>docker ps
>```
>
>いくつかのコンテナが走っているのを確認してください。`cygnus` が実行されていない場合は、必要に応じてコンテナを再起動できます。


<a name="generating-context-data-1"></a>
### コンテキスト・データの生成

このチュートリアルでは、コンテキストが定期的に更新されるシステムを監視する必要があります。ダミー IoT センサを使用してこれを行うことができます。`http://localhost:3000/device/monitor` でデバイス・モニタのページを開き、**スマート・ドア**のロックを解除し、**スマート・ランプ**をオンにします。これは、ドロップ・ダウン・リストから適切なコマンドを選択し、`send` ボタンを押すことによって行うことができます。デバイスからの測定値のストリームは、同じページに表示されます :

![](https://fiware.github.io/tutorials.Historic-Context/img/door-open.gif)



<a name="subscribing-to-context-changes-1"></a>
### コンテキスト変更のサブスクライブ

動的コンテキスト・システムが起動したら、**Cygnus** にコンテキストの変更を通知する必要があります。

これは、Orion Context Broker の `/v2/subscription` エンドポイントに POST リクエストを行うことによって行われます。

* `fiware-service` と `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続されている IoT センサからの測定値だけをリッスンするために使用します。センサがこれらの設定を使用してプロビジョニングされているためです
* リクエスト・ボディの `idPattern` は、すべてのコンテキスト・データの変更が Cygnus に通知されるようにします
* 通知 `url` は設定された `CYGNUS_API_PORT` と一致する必要があります
* Cygnus は現在、古い NGSI v1 形式の通知のみを受け付けているため、`attrsFormat=legacy` が必要です
* `throttling` 値は、変更がサンプリングされる割合を定義します

#### :four: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus of all context changes",
  "subject": {
    "entities": [
      {
        "idPattern": ".*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

ご覧のとおり、コンテキスト・データを保持するために使用されるデータベースは、サブスクリプションの詳細に影響を与えません。各データベースで同じです。レスポンスは **201 - Created** です。


<a name="postgresql---reading-data-from-a-database"></a>
## PostgreSQL - データベースからデータを読み込む

コマンドラインから PostgreSQL データを読み取るには、`postgres` クライアントにアクセスする必要があります。これを行うには、コマンドライン・プロンプトを表示するために接続文字列を指定した `postgresql-client` イメージのインタラクティブなインスタンスを実行します :

```console
docker run -it --rm  --network fiware_default jbergknoff/postgresql-client \
   postgresql://postgres:password@postgres-db:5432/postgres
```

<a name="show-available-databases-on-the-postgresql-server"></a>
### PostgreSQL サーバ上で利用可能なデータベースを表示

使用可能なデータベースのリストを表示するには、次のように文を実行します :

#### クエリ :

```
\list
```

#### 結果 :

```
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)
```

結果には、2つのテンプレート・データベース `template0` と `template1` と、Dokcer コンテナが起動したときの `postgres` データベースの設定が含まれています。


使用可能なスキーマのリストを表示するには、次のように文を実行します :

#### クエリ :

```
\dn
```

#### 結果 :

```
  List of schemas
  Name   |  Owner   
---------+----------
 openiot | postgres
 public  | postgres
(2 rows)
```


Orion Context Broker に Cygnus をサブスクリプションした結果、`openiot` という新しいスキーマが作成されました。スキーマの名前は `fiware-service` ヘッダに一致します。したがって、`openiot` は、IoT デバイスの履歴コンテキストを保持します。


<a name="read-historical-context-from-the-postgresql-server"></a>
### PostgreSQL サーバから履歴コンテキストを読み込む

Docker コンテナをネットワーク内で実行すると、実行中のデータベースに関する情報を取得することができます。


#### クエリ :

```sql
SELECT table_schema,table_name
FROM information_schema.tables
WHERE table_schema ='openiot'
ORDER BY table_schema,table_name;
```

#### 結果 :

```
 table_schema |    table_name     
--------------+-------------------
 openiot      | door_001_door
 openiot      | lamp_001_lamp
 openiot      | motion_001_motion
(3 rows)
```

`table_schema` は、コンテキスト・データとともに提供される `fiware-service` ヘッダと一致します。

テーブル内のデータを読み込むには、次のように select 文を実行します :

#### クエリ :

```sql
SELECT * FROM openiot.motion_001_motion limit 10;
```

#### 結果 :

```
  recvtimets   |         recvtime         | fiwareservicepath |  entityid  | entitytype |  attrname   |   attrtype   |        attrvalue         |                                    attrmd                                    
---------------+--------------------------+-------------------+------------+------------+-------------+--------------+--------------------------+------------------------------------------------------------------------------
 1528803005491 | 2018-06-12T11:30:05.491Z | /                 | Motion:001 | Motion     | TimeInstant | ISO8601      | 2018-06-12T11:30:05.423Z | []
 1528803005491 | 2018-06-12T11:30:05.491Z | /                 | Motion:001 | Motion     | count       | Integer      | 7                        | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:30:05.423Z"}]
 1528803005491 | 2018-06-12T11:30:05.491Z | /                 | Motion:001 | Motion     | refStore    | Relationship | Store:001                | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:30:05.423Z"}]
 1528803035501 | 2018-06-12T11:30:35.501Z | /                 | Motion:001 | Motion     | TimeInstant | ISO8601      | 2018-06-12T11:30:35.480Z | []
 1528803035501 | 2018-06-12T11:30:35.501Z | /                 | Motion:001 | Motion     | count       | Integer      | 10                       | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:30:35.480Z"}]
 1528803035501 | 2018-06-12T11:30:35.501Z | /                 | Motion:001 | Motion     | refStore    | Relationship | Store:001                | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:30:35.480Z"}]
 1528803041563 | 2018-06-12T11:30:41.563Z | /                 | Motion:001 | Motion     | TimeInstant | ISO8601      | 2018-06-12T11:30:41.520Z | []
 1528803041563 | 2018-06-12T11:30:41.563Z | /                 | Motion:001 | Motion     | count       | Integer      | 12                       | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:30:41.520Z"}]
 1528803041563 | 2018-06-12T11:30:41.563Z | /                 | Motion:001 | Motion     | refStore    | Relationship | Store:001                | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:30:41.520Z"}]
 1528803047545 | 2018-06-12T11:30:47.545Z | /     
```

通常の **PostgreSQL** クエリ構文を使用して、適切なフィールドと値をフィルタリングすることができます。たとえば、`id=Motion:001_Motion` の**モーション・センサ**が蓄積しているレートを読み取るには、次のようにクエリを作成します :

#### クエリ :

```sql
SELECT recvtime, attrvalue FROM openiot.motion_001_motion WHERE attrname ='count'  limit 10;
```

#### 結果 :

```
         recvtime         | attrvalue 
--------------------------+-----------
 2018-06-12T11:30:05.491Z | 7
 2018-06-12T11:30:35.501Z | 10
 2018-06-12T11:30:41.563Z | 12
 2018-06-12T11:30:47.545Z | 13
 2018-06-12T11:31:02.617Z | 15
 2018-06-12T11:31:32.718Z | 20
 2018-06-12T11:31:38.733Z | 22
 2018-06-12T11:31:50.780Z | 24
 2018-06-12T11:31:56.825Z | 25
 2018-06-12T11:31:59.790Z | 26
(10 rows)
```

Postgres クライアントを終了してインタラクティブ・モードを終了するには、次のコマンドを実行します :

```console
\q
```
 You will then return to the commmand line.




<a name="mysql---persisting-context-data-into-a-database"></a>
# MySQL - コンテキスト・データをデータベースに永続化

同様に、履歴コンテキスト・データ**MySQL** に永続化するには、MySQL サーバをホストする追加のコンテナが必要になります。このデータのデフォルトの Docker イメージも使用できます。MySQL インスタンスは標準 `3306` ポートでリッスンしており、全体のアーキテクチャは以下のようになります :

![](https://fiware.github.io/tutorials.Historic-Context/img/cygnus-mysql.png)

MongoDB コンテナは、Orion Context Broker と IoT Agent に関連するデータを保持する必要があるため、2つのデータベースを持つシステムがあります。


<a name="mysql---database-server-configuration"></a>
## MySQL - データベース・サーバの設定

```yaml
  mysql-db:
      restart: always
      image: mysql:5.7
      hostname: mysql-db
      container_name: db-mysql
      expose:
        - "3306"
      ports:
        - "3306:3306"
      networks:
        - default
      environment:
        - "MYSQL_ROOT_PASSWORD=123"
        - "MYSQL_ROOT_HOST=%"
```

`mysql-db` コンテナは、単一ポートで待機しています :

* ポート `3306` は MySQL サーバのデフォルト・ポートです。これは公開されているので、必要に応じて他のデータベース・ツールを実行してデータを表示することもできます

`mysql-db` コンテナは、次のように環境変数によって制御されます :

| キー              | 値       | 説明                                     |
|-------------------|----------|------------------------------------------|
|MYSQL_ROOT_PASSWORD|`123`     | MySQL `root` アカウントに設定されているパスワードを指定します |
|MYSQL_ROOT_HOST    |`postgres`| デフォルトでは、MySQL によって `root'@'localhost` アカウントが作成されます。このアカウントはコンテナ内からのみ接続できます。この環境変数を設定すると、他のホストからのルート接続が可能になります | 


<a name="mysql---cygnus-configuration"></a>
## MySQL - Cygnusの設定

```yaml
  cygnus:
    image: fiware/cygnus-ngsi:latest
    hostname: cygnus
    container_name: fiware-cygnus
    networks:
        - default
    depends_on:
        - mysql-db
    expose:
        - "5080"
    ports:
        - "5050:5050"
        - "5080:5080"
    environment:
        - "CYGNUS_MYSQL_HOST=mysql-db"
        - "CYGNUS_MYSQL_PORT=3306"
        - "CYGNUS_MYSQL_USER=root" 
        - "CYGNUS_MYSQL_PASS=123" 
        - "CYGNUS_LOG_LEVEL=DEBUG"
        - "CYGNUS_SERVICE_PORT=5050"
        - "CYGNUS_API_PORT=5080"
```



`cygnus` コンテナは、2つのポートでリッスンしています :

* Cygnus のサブスクリプション・ポート : `5050` は、サービスが Orion context broker からの通知をリッスンするポートです
* Cygnus の管理ポート : `5080`は、純粋にチュートリアル・アクセスのために公開されているため、cUrl または Postman は同じネットワークの一部ではなくプロビジョニング・コマンドを作成できます


`cygnus` コンテナは、図のように環境変数によって制御されます :

| キー                          | 値           | 説明      |
|-------------------------------|--------------|-----------|
|CYGNUS_MYSQL_HOST              |`mysql-db`    | Hostname of the MySQL server used to persist historical context data |
|CYGNUS_MYSQL_PORT              |`3306`        | Port that the MySQL server uses to listen to commands |
|CYGNUS_MYSQL_USER              |`root`        | Username for the MySQL database user | 
|CYGNUS_MYSQL_PASS              |`123`         | Password for the MySQL database user |
|CYGNUS_LOG_LEVEL               |`DEBUG`       | The logging level for Cygnus |
|CYGNUS_SERVICE_PORT            |`5050`        | Notification Port that Cygnus listens when subcribing to context data changes|
|CYGNUS_API_PORT                |`5080`        | Port that Cygnus listens on for operational reasons |


<a name="mysql---start-up"></a>
## MySQL - 起動

**MySQL** データベースを使用してシステムを起動するには、次のコマンドを実行します :

```console
./services mysql
``` 

<a name="checking-the-cygnus-service-health-2"></a>
### Cygnus サービスの健全性をチェック
 
Cygnus が動作したら、公開されている `CYGNUS_API_PORT` ポートへの HTTP リクエストを行うことでステータスを確認できます。レスポンスがブランクの場合、これは通常、Cygnus が実行されていないか、別のポートでリッスンしているためです。

#### :five: リクエスト :

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### レスポンス :

レスポンスは次のようになります :

```json
{
    "success": "true",
    "version": "1.8.0_SNAPSHOT.ed50706880829e97fd4cf926df434f1ef4fac147"
}
```

>**トラブルシューティング** : レスポンスが空白の場合はどうなりますか？
>
> * Dokcer コンテナが動作していることを確認するには :
>
>```bash
>docker ps
>```
>
>いくつかのコンテナが走っているのを確認してください。`cygnus` が実行されていない場合は、必要に応じてコンテナを再起動できます。


<a name="generating-context-data-2"></a>
### コンテキスト・データの生成

このチュートリアルでは、コンテキストが定期的に更新されるシステムを監視する必要があります。ダミー IoT センサを使用してこれを行うことができます。`http://localhost:3000/device/monitor` でデバイス・モニタのページを開き、**スマート・ドア**のロックを解除し、**スマート・ランプ**をオンにします。これは、ドロップ・ダウン・リストから適切なコマンドを選択し、`send` ボタンを押すことによって行うことができます。デバイスからの測定値のストリームは、同じページに表示されます :

![](https://fiware.github.io/tutorials.Historic-Context/img/door-open.gif)


<a name="subscribing-to-context-changes-2"></a>
### コンテキスト変更のサブスクライブ

動的コンテキスト・システムが起動したら、Cygnus にコンテキストの変更を通知する必要があります。

これは、Orion Context Broker の `/v2/subscription` エンドポイントに POST リクエストを行うことによって行われます。

* `fiware-service` と `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続されている IoT センサからの測定値だけをリッスンするために使用します。センサがこれらの設定を使用してプロビジョニングされているためです
* リクエスト・ボディの `idPattern` は、すべてのコンテキスト・データの変更が Cygnus に通知されるようにします
* 通知 `url` は設定された `CYGNUS_API_PORT` と一致する必要があります
* Cygnus は現在、古い NGSI v1 形式の通知のみを受け付けているため、`attrsFormat=legacy` が必要です
* `throttling` 値は、変更がサンプリングされる割合を定義します

#### :six: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus of all context changes",
  "subject": {
    "entities": [
      {
        "idPattern": ".*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

ご覧のとおり、コンテキスト・データを保持するために使用されるデータベースは、サブスクリプションの詳細に影響を与えません。各データベースで同じです。レスポンスは **201 - Created** です。


<a name="mysql---reading-data-from-a-database"></a>
## MySQL - データベースからデータを読み込む

コマンドラインから MySQL データを読み込むには、`mysql` クライアントにアクセスする必要があります。これを行うには、コマンドライン・プロンプトを表示するために接続文字列を指定した `mysql` イメージのインタラクティブなインスタンスを実行します :

```console
docker run -it --rm  --network fiware_default mysql mysql -h mysql-db -P 3306  -u root -p123
```

<a name="show-available-databases-on-the-mysql-server"></a>
### MySQL サーバ上で利用可能なデータベースを表示

使用可能なデータベースのリストを表示するには、次のように文を実行します :

#### クエリ :

```sql
SHOW DATABASES;
```

#### 結果 :

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| openiot            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```


使用可能なスキーマのリストを表示するには、次のように文を実行します :

#### クエリ :

```sql
SHOW SCHEMAS;
```

#### 結果 :

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| openiot            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```


Orgn Context Broker に Cygnus をサブスクリプションした結果、`openiot` という新しいスキーマが作成されました。スキーマの名前は `fiware-service` ヘッダに一致します。したがって、`openiot` は、IoT デバイスの履歴コンテキストを保持します。


<a name="read-historical-context-from-the-mysql-server"></a>
### MySQL サーバから履歴コンテキストを読み込む

Docker コンテナをネットワーク内で実行すると、実行中のデータベースに関する情報を取得することができます。


#### クエリ :

```sql
SHOW tables FROM openiot;
```

#### 結果 :

```
 table_schema |    table_name     
--------------+-------------------
 openiot      | door_001_door
 openiot      | lamp_001_lamp
 openiot      | motion_001_motion
(3 rows)
```

`table_schema` は、コンテキスト・データとともに提供される `fiware-service` ヘッダと一致します。

テーブル内のデータを読み込むには、次のように select 文を実行します :

#### クエリ :

```sql
SELECT * FROM openiot.Motion_001_Motion limit 10;
```

#### 結果 :

```
+---------------+-------------------------+-------------------+------------+------------+-------------+--------------+--------------------------+------------------------------------------------------------------------------+
| recvTimeTs    | recvTime                | fiwareServicePath | entityId   | entityType | attrName    | attrType     | attrValue                | attrMd                                                                       |
+---------------+-------------------------+-------------------+------------+------------+-------------+--------------+--------------------------+------------------------------------------------------------------------------+
| 1528804397955 | 2018-06-12T11:53:17.955 | /                 | Motion:001 | Motion     | TimeInstant | ISO8601      | 2018-06-12T11:53:17.923Z | []                                                                           |
| 1528804397955 | 2018-06-12T11:53:17.955 | /                 | Motion:001 | Motion     | count       | Integer      | 3                        | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:53:17.923Z"}] |
| 1528804397955 | 2018-06-12T11:53:17.955 | /                 | Motion:001 | Motion     | refStore    | Relationship | Store:001                | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:53:17.923Z"}] |
| 1528804403954 | 2018-06-12T11:53:23.954 | /                 | Motion:001 | Motion     | TimeInstant | ISO8601      | 2018-06-12T11:53:23.928Z | []                                                                           |
| 1528804403954 | 2018-06-12T11:53:23.954 | /                 | Motion:001 | Motion     | count       | Integer      | 5                        | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:53:23.928Z"}] |
| 1528804403954 | 2018-06-12T11:53:23.954 | /                 | Motion:001 | Motion     | refStore    | Relationship | Store:001                | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:53:23.928Z"}] |
| 1528804409970 | 2018-06-12T11:53:29.970 | /                 | Motion:001 | Motion     | TimeInstant | ISO8601      | 2018-06-12T11:53:29.948Z | []                                                                           |
| 1528804409970 | 2018-06-12T11:53:29.970 | /                 | Motion:001 | Motion     | count       | Integer      | 7                        | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:53:29.948Z"}] |
| 1528804409970 | 2018-06-12T11:53:29.970 | /                 | Motion:001 | Motion     | refStore    | Relationship | Store:001                | [{"name":"TimeInstant","type":"ISO8601","value":"2018-06-12T11:53:29.948Z"}] |
| 1528804446083 | 2018-06-12T11:54:06.83  | /                 | Motion:001 | Motion     | TimeInstant | ISO8601      | 2018-06-12T11:54:06.062Z | []                                                                           |
+---------------+-------------------------+-------------------+------------+------------+-------------+--------------+--------------------------+------------------------------------------------------------------------------+    
```

通常の **MySQL** クエリ構文を使用して、適切なフィールドと値をフィルタリングすることができます。たとえば、`id=Motion:001_Motion` の**モーション・センサ**が蓄積しているレートを読み取るには、次のようにクエリを作成します :

#### クエリ :

```sql
SELECT recvtime, attrvalue FROM openiot.Motion_001_Motion WHERE attrname ='count' LIMIT 10;
```

#### 結果 :

```
+-------------------------+-----------+
| recvtime                | attrvalue |
+-------------------------+-----------+
| 2018-06-12T11:53:17.955 | 3         |
| 2018-06-12T11:53:23.954 | 5         |
| 2018-06-12T11:53:29.970 | 7         |
| 2018-06-12T11:54:06.83  | 12        |
| 2018-06-12T11:54:12.132 | 13        |
| 2018-06-12T11:54:24.177 | 14        |
| 2018-06-12T11:54:36.196 | 16        |
| 2018-06-12T11:54:42.195 | 18        |
| 2018-06-12T11:55:24.300 | 23        |
| 2018-06-12T11:55:30.350 | 25        |
+-------------------------+-----------+
10 rows in set (0.00 sec)
```

MySQL クライアントを終了し、インタラクティブ・モードを終了するには、次のコマンドを実行します :

```console
\q
```
その後、コマンドラインに戻ります。


<a name="multi-agent---persisting-context-data-into-a-multiple-databases"></a>
# マルチ・エージェント - 複数のデータベースへのコンテキスト・データの永続化

また、複数のデータベースを同時に設定するように Cygnus を設定することもできます。以前の3つの例のアーキテクチャを組み合わせて、複数のポートでリッスンするように cygnus を構成することができます


![](https://fiware.github.io/tutorials.Historic-Context/img/cygnus-all-three.png)

現在、データ永続化のための PostgreSQL と MySQLと、データ永続化と Orion Context Broker と IoT Agent に関連するデータ永続化の両方のための MongoDB という 3つのデータベースのシステムを持っています。

<a name="multi-agent---cygnus-configuration-for-multiple-databases"></a>
## マルチ・エージェント - 複数のデータベースのための Cygnus 設定

```yaml
  cygnus:
    image: fiware/cygnus-ngsi:latest
    hostname: cygnus
    container_name: fiware-cygnus
    depends_on:
      - mongo-db
      - mysql-db
      - postgres-db
    networks:
      - default
    expose:
      - "5080"
      - "5081"
      - "5084"
    ports:
      - "5050:5050"
      - "5051:5051"
      - "5054:5054"
      - "5080:5080"
      - "5081:5081"
      - "5084:5084"
    environment:
      - "CYGNUS_MULTIAGENT=true"
      - "CYGNUS_POSTGRESQL_HOST=postgres-sb"
      - "CYGNUS_POSTGRESQL_PORT=5432"
      - "CYGNUS_POSTGRESQL_USER=postgres" 
      - "CYGNUS_POSTGRESQL_PASS=password" 
      - "CYGNUS_POSTGRESQL_ENABLE_CACHE=true"
      - "CYGNUS_MYSQL_HOST=mysql-db"
      - "CYGNUS_MYSQL_PORT=3306"
      - "CYGNUS_MYSQL_USER=root" 
      - "CYGNUS_MYSQL_PASS=123" 
      - "CYGNUS_LOG_LEVEL=DEBUG"
```



マルチ・エージェントのモードでは、`cygnus` コンテナは複数のポートで待機しています :

* このサービスは、Orion Context Broker からの通知用ポート `5050-5055` でリッスンします
* 管理ポート `5080-5085` はチュートリアル・アクセスのためだけに公開されているため、cUrl または Postman は同じネットワークの一部ではなくプロビジョニング・コマンドを作成できます

以下に、デフォルトのポート・マッピングを示します :

| sink       | port | admin_port |
|-----------:|-----:|-----------:|
| mysql      | 5050 | 5080       |
| mongo      | 5051 | 5081       |
| ckan       | 5052 | 5082       |
| hdfs       | 5053 | 5083       |
| postgresql | 5054 | 5084       |
| cartodb    | 5055 | 5085       |

CKAN、HDFS、または CartoDB データを保持していないため、これらのポートを開く必要はありません。




`cygnus` コンテナは、次のように環境変数によって制御されます :

| キー                          | 値           | 説明      |
|-------------------------------|--------------|-----------|
|CYGNUS_MULTIAGENT              |`true`        | データを複数のデータベースに保持するかどうか |
|CYGNUS_MONGO_HOSTS             |`mongo-db:27017` | Cygnus が履歴コンテキスト・データを保持するために接続する Mongo-DB サーバのカンマ区切りリスト|
|CYGNUS_POSTGRESQL_HOST         |`postgres-db` | 履歴コンテキスト・データの永続化に使用される PostgreSQL サーバのホスト名 |
|CYGNUS_POSTGRESQL_PORT         |`5432`        | PostgreSQL サーバがコマンドをリッスンするために使うポート |
|CYGNUS_POSTGRESQL_USER         |`postgres`    | PostgreSQL データベース・ユーザのユーザ名| 
|CYGNUS_POSTGRESQL_PASS         |`password`    | PostgreSQL データベース・ユーザのパスワード |
|CYGNUS_MYSQL_HOST              |`mysql-db`    | 履歴コンテキスト・データの永続化に使用される MySQL サーバのホスト名 |
|CYGNUS_MYSQL_PORT              |`3306`        | MySQL サーバがコマンドをリッスンするために使用するポート |
|CYGNUS_MYSQL_USER              |`root`        | MySQL データベース・ユーザのユーザ名 | 
|CYGNUS_MYSQL_PASS              |`123`         | MySQL データベース・ユーザのパスワード |
|CYGNUS_LOG_LEVEL               |`DEBUG`       | Cygnus のログレベル |


<a name="multi-agent---start-up"></a>
## マルチ・エージェント - 起動

**複数**のデータベースを使用してシステムを起動するには、次のコマンドを実行します :

```console
./services multiple
``` 

<a name="checking-the-cygnus-service-health-3"></a>
### Cygnus サービスの健全性をチェック
 
Cygnus が動作したら、公開されている `CYGNUS_API_PORT` ポートへの HTTP リクエストを行うことでステータスを確認できます。レスポンスがブランクの場合、これは通常、Cygnus が実行されていないか、別のポートでリッスンしているためです。

#### :seven: リクエスト :

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### レスポンス :

レスポンスは次のようになります 

```json
{
    "success": "true",
    "version": "1.8.0_SNAPSHOT.ed50706880829e97fd4cf926df434f1ef4fac147"
}
```

>**トラブルシューティング** : レスポンスが空白の場合はどうなりますか？
>
> * Dokcer コンテナが動作していることを確認するには :
>
>```bash
>docker ps
>```
>
>いくつかのコンテナが走っているのを確認してください。`cygnus` が実行されていない場合は、必要に応じてコンテナを再起動できます。


<a name="generating-context-data-3"></a>
### コンテキスト・データの生成

このチュートリアルでは、コンテキストが定期的に更新されるシステムを監視する必要があります。ダミー IoT センサを使用してこれを行うことができます。`http://localhost:3000/device/monitor` でデバイス・モニタのページを開き、**スマート・ドア**のロックを解除し、**スマート・ランプ**をオンにします。これは、ドロップ・ダウン・リストから適切なコマンドを選択し、`send` ボタンを押すことによって行うことができます。デバイスからの測定値のストリームは、同じページに表示されます :

![](https://fiware.github.io/tutorials.Historic-Context/img/door-open.gif)


<a name="subscribing-to-context-changes-3"></a>
### コンテキスト変更のサブスクライブ

動的コンテキスト・システムが起動したら、**Cygnus** にコンテキストの変更を通知する必要があります。

これは、Orion Context Broker の `/v2/subscription` エンドポイントに POST リクエストを行うことによって行われます。

* `fiware-service` と `fiware-servicepath` ヘッダは、サブスクリプションをフィルタリングして、接続されている IoT センサからの測定値だけをリッスンするために使用します。センサがこれらの設定を使用してプロビジョニングされているためです
* リクエスト・ボディの `idPattern` は、すべてのコンテキスト・データの変更が Cygnus に通知されるようにします
* 通知 `url` は設定された `CYGNUS_API_PORT` と一致する必要があります
* Cygnus は現在、古い NGSI v1 形式の通知のみを受け付けているため、`attrsFormat=legacy` が必要です
* `throttling` 値は、変更がサンプリングされる割合を定義します

#### :eight: リクエスト :

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus of all context changes",
  "subject": {
    "entities": [
      {
        "idPattern": ".*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

ご覧のとおり、コンテキスト・データを保持するために使用されるデータベースは、サブスクリプションの詳細に影響を与えません。各データベースで同じです。レスポンスは **201 - Created** です。

<a name="multi-agent---reading-persisted-data"></a>
## マルチ・エージェント - 永続化データの読み込み

添付されたデータベースから永続化されたデータを読み込むには、このチュートリアルの前のセクションを参照してください。

<a name="next-steps"></a>
# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか？このシリーズの他のチュートリアルを読むことで見つけることができます :

&nbsp; 101. [Getting Started](https://github.com/Fiware/tutorials.Getting-Started)<br/>
&nbsp; 102. [Entity Relationships](https://github.com/Fiware/tutorials.Entity-Relationships)<br/>
&nbsp; 103. [CRUD Operations](https://github.com/Fiware/tutorials.CRUD-Operations)<br/>
&nbsp; 104. [Context Providers](https://github.com/Fiware/tutorials.Context-Providers)<br/>
&nbsp; 105. [Altering the Context Programmatically](https://github.com/Fiware/tutorials.Accessing-Context)<br/> 
&nbsp; 106. [Subscribing to Changes in Context](https://github.com/Fiware/tutorials.Subscriptions)<br/>

&nbsp; 201. [Introduction to IoT Sensors](https://github.com/Fiware/tutorials.IoT-Sensors)<br/>
&nbsp; 202. [Provisioning an IoT Agent](https://github.com/Fiware/tutorials.IoT-Agent)<br/>
&nbsp; 250. [Introduction to Fast-RTPS and Micro-RTPS ](https://github.com/Fiware/tutorials.Fast-RTPS-Micro-RTPS)<br/>

&nbsp; 301. [Persisting Context Data](https://github.com/Fiware/tutorials.Historic-Context)<br/>
&nbsp; 302. [Querying Time Series Data](https://github.com/Fiware/tutorials.Short-Term-History)<br/>