# Python CDK: Creating a HTTP API Source

2021年12月25日現在の[Python CDK: Creating a HTTP API Source](https://docs.airbyte.io/connector-development/tutorials/cdk-tutorial-python-http)をDeepLで翻訳したものです。


**目次**

- [Python CDK: Creating a HTTP API Source](#python-cdk-creating-a-http-api-source)
  - [概要](#概要)
  - [要件](#要件)
  - [チェックリスト](#チェックリスト)
- [Step 1: テンプレートを使ってソースを作成する](#step-1-テンプレートを使ってソースを作成する)
- [Step 2: 新しいソースの依存関係をインストールする](#step-2-新しいソースの依存関係をインストールする)
  - [イテレーションサイクルに関する留意点](#イテレーションサイクルに関する留意点)
    - [依存関係](#依存関係)
    - [開発環境](#開発環境)
    - [実装の反復](#実装の反復)
- [Step 3: コネクタに必要な入力を定義する](#step-3-コネクタに必要な入力を定義する)
- [Step 4: 接続チェックを実施する](#step-4-接続チェックを実施する)
- [Step 5: ストリームのスキーマを宣言する](#step-5-ストリームのスキーマを宣言する)
- [Step 6: ストリームを読み込むための機能を実装する](#step-6-ストリームを読み込むための機能を実装する)
  - [インクリメンタル同期を追加する](#インクリメンタル同期を追加する)
- [Step 7: Airbyteのコネクタを使用する](#step-7-airbyteのコネクタを使用する)
- [Step 8: 単体テストまたは結合テストの作成](#step-8-単体テストまたは結合テストの作成)
  - [単体テスト](#単体テスト)
  - [結合テスト](#結合テスト)
  - [標準テスト](#標準テスト)

## 概要

PythonでAirbyteソースを作成し、HTTP APIからデータを読み込む方法をステップバイステップで説明します。ここでは、シンプルでCDKの多くの機能を実証するため、例として為替レートAPIを使用します。

## 要件

* Python &gt;= 3.7
* Docker
* NodeJS \(コネクタの生成にのみ使用します。NodeJSの依存関係は近日中に削除します\)

これ以降のコマンドはすべて、`python` が python &gt;=3.7.0 のバージョンであるという前提で表記します。システムによっては、`python` が Python2 を、`python3` が Python3 を指している場合があります。もしあなたのマシンがそうであれば、このガイドのすべての `python` コマンドを `python3` に置き換えてください。

## チェックリスト

* Step 1: テンプレートを使ってソースを作成する
* Step 2: 新しいソースの依存関係をインストールする
* Step 3: コネクタに必要な入力を定義する
* Step 4: 接続チェックを実施する
* Step 5: ストリームのスキーマを宣言する
* Step 6: ストリームを読み込むための機能を実装する
* Step 7: Airbyteのコネクタを使用する
* Step 8: 単体テストまたは結合テストの作成

チェックリストの各ステップは、以下の手順で詳しく説明します。またチュートリアルの最後にAirbyteの一般的なリリースに含めるためのコネクタ提出方法についても触れます。  

# Step 1: テンプレートを使ってソースを作成する

Airbyteはコネクタの基礎となるコードジェネレータを提供しています。

```bash
$ cd airbyte-integrations/connector-templates/generator # カレントディレクトリがAirbyteプロジェクトのrootであると仮定しています。
# NPMをお持ちでない場合は、https://www.npmjs.com/get-npm からインストールしてください。
$ ./generate.sh
```

すると対話型のヘルパーアプリケーションが表示されます。カーソルキーを使って、リストからテンプレートを選びます。今回のケースでは`Python HTTP API Source` テンプレートを選択し、コネクタの名前を入力します。このシェルスクリプトは airbyte/airbyte-integrations/connectors/ に新しいコネクタの名前を付けて新しいディレクトリを作成します。

今回の手順では、このソースを `python-http-example` と呼びます。このチュートリアルの最終的なソースコードは、[ここ](https://github.com/airbytehq/airbyte/tree/master/airbyte-integrations/connectors/source-python-http-tutorial) にあります。

このチュートリアルで構築するソースは、[Rates API](https://exchangeratesapi.io/) からデータを取得します。これは、フィアット通過の過去の為替レートを記録した、フリーでオープンな API です。

# Step 2: 新しいソースの依存関係をインストールする

モジュールを生成したので、そのディレクトリに移動して依存関係をインストールしましょう。

```text
cd ../../connectors/source-<name>
python -m venv .venv # .venvディレクトリに仮想環境を作成します。
source .venv/bin/activate # enable the venv
pip install -r requirements.txt
```

このステップでは、最初の Python 環境を設定します。**以降のすべての** `python` または `pip` コマンドは仮想環境をアクティベートした状態を前提としています。

では、意図した通りに動作するか確認しましょう。以下を実行してください。

```text
python main.py spec
```

いくつかの出力が表示されるはずです。

```text
{"type": "SPEC", "spec": {"documentationUrl": "https://docsurl.com", "connectionSpecification": {"$schema": "http://json-schema.org/draft-07/schema#", "title": "Python Http Tutorial Spec", "type": "object", "required": ["TODO"], "additionalProperties": false, "properties": {"TODO: This schema defines the configuration required for the source. This usually involves metadata such as database and/or authentication information.": {"type": "string", "description": "describe me"}}}}}
```

Airbyte Protocolの`spec`コマンドを実行しました! これについては後で詳しく説明しますが、これはすべてが正しく繋がっていることを確認するための簡単なサニティチェックです。

`main.py` ファイルは、コネクタを簡単に実行できるようにするための単純なスクリプトであることに気を付けてください。呼び出す際のフォーマットは `python main.py <command> [args]` です。サポートしているコマンドについては、モジュールが生成する `README.md` を参照してください。

## イテレーションサイクルに関する留意点

### 依存関係

Python の依存関係は `airbyte-integrations/connectors/source-<source-name>/setup.py` の `install_requires` フィールドで宣言する必要があります。そこで、いくつかの Airbyte の依存関係がすでに宣言されていることに気がつくでしょう。これらは、ジェネレータが提供するヘルパーインターフェースにあなたのソースがアクセスするためのものです。

ソースのディレクトリに `requirements.txt` があることに気がつくかもしれません。これは編集しないでください。これは自動生成され、Airbyte の依存関係を提供するために使用されます。すべての依存関係は `setup.py` で宣言する必要があります。

### 開発環境

上で実行したコマンドは、あなたのソースのための[Python仮想環境](https://docs.python.org/3/tutorial/venv.html)を作成します。IDE で自動補完や依存関係の解決を適切に行いたい場合は、仮想環境 `airbyte-integrations/connectors/source-<source-name>/.venv` を指定してください。また、`setup.py` の依存関係を変更したら、いつでも `pip install -r requirements.txt` を再実行するようにしてください。

### 実装の反復

ソースの反復処理には、2つの方法をお勧めします。あなたのスタイルに合った方を選んでください。

**pythonを使ってソースを実行する**

ソースのディレクトリに `main.py` という python ファイルがあることに気がつくでしょう。このファイルは開発の便宜のために存在します。このファイルは、あなたのソースが動作するかどうかをテストするために実行されます。

```text
# airbyte-integrations/connectors/source-<name>上で実行する
python main.py spec
python main.py check --config secrets/config.json
python main.py discover --config secrets/config.json
python main.py read --config secrets/config.json --catalog sample_files/configured_catalog.json
```

この方法の良いところは、pythonの中で完全に反復処理できることです。欠点は、実際にAirbyteで実行されるようにソースを実行するわけではないことです。具体的には、ソースを格納するDockerコンテナの中でソースを実行していないのです。

**dockerを使用してソースを実行する**

Airbyte で実行するのと全く同じように（Docker コンテナ内で）ソースを実行したい場合は、コネクタモジュールのあるディレクトリ（`airbyte-integrations/connectors/source-python-http-example`）から以下のコマンドを使用してください。

```text
# まずコンテナを構築します
docker build . -t airbyte/source-<name>:dev

# 次に下記コマンドを実行してコンテナを起動します
docker run --rm airbyte/source-python-http-example:dev spec
docker run --rm -v $(pwd)/secrets:/secrets airbyte/source-python-http-example:dev check --config /secrets/config.json
docker run --rm -v $(pwd)/secrets:/secrets airbyte/source-python-http-example:dev discover --config /secrets/config.json
docker run --rm -v $(pwd)/secrets:/secrets -v $(pwd)/sample_files:/sample_files airbyte/source-python-http-example:dev read --config /secrets/config.json --catalog /sample_files/configured_catalog.json
```

注意：実装を変更するたびに、`docker build . -t airbyte/source-<name>:dev`でコネクタのイメージを再ビルドする必要があります。これにより、新しい Python コードが docker コンテナに追加されるようになります。

この方法の良いところは、Airbyte で実行されるのと全く同じようにソースを実行することです。トレードオフとして、各変更間でコネクタが再構築されるため、イテレーションが若干遅くなることです。

# Step 3: コネクタに必要な入力を定義する

各コネクタは、基礎となるデータソースからデータを読み取るために必要な入力を宣言します。これがAirbyte Protocolの `spec` オペレーションです。

これを実装する最も簡単な方法は、`source_<name>/spec.json` に `.json` ファイルを作成して、[ConnectorSpecification](https://github.com/airbytehq/airbyte/blob/master/airbyte-protocol/models/src/main/resources/airbyte_protocol/airbyte_protocol.yaml#L211) のスキーマに従って、あなたのコネクタの入力を記述することです。これはソースを開発する際の良いスタート地点になります。JsonSchema を使用して、入力が何であるかを定義します。以下はFreshdesk APIソースの `spec.json` がどのようなものかを示す[例](https://github.com/airbytehq/airbyte/blob/master/airbyte-integrations/connectors/source-freshdesk/source_freshdesk/spec.json)です。

specとは何かということについては、Airbyte Protocolについての[ドキュメント](https://docs.airbyte.io/understanding-airbyte/airbyte-specification)をご覧ください。

冒頭で自動生成したシェルスクリプトは、`spec`メソッドの実装を自動的に行います。これは`source.py`と同じディレクトリに `spec.json` というファイルが存在することを想定しています。あなたが`spec.json`で必要なJsonSchemaを宣言すれば、このステップは終了です。

今回のサンプルでは通貨データを取得してくるので、以下の `spec.json` を定義します。

```text
{
  "documentationUrl": "https://docs.airbyte.io/integrations/sources/exchangeratesapi",
  "connectionSpecification": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "Python Http Tutorial Spec",
    "type": "object",
    "required": ["start_date", "base"],
    "additionalProperties": false,
    "properties": {
      "start_date": {
        "type": "string",
        "description": "Start getting data from that date.",
        "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$",
        "examples": ["%Y-%m-%d"]
      },
      "base": {
        "type": "string",
        "examples": ["USD", "EUR"],
        "description": "ISO reference currency. See <a href=\"https://www.ecb.europa.eu/stats/policy_and_exchange_rates/euro_reference_exchange_rates/html/index.en.html\">here</a>."
      }
    }
  }
}
```

メタデータに加えて、2つの入力を定義します。

* `start_date`: 為替レートのトラッキング開始日
* `base`: トラッキングしたい通貨



# Step 4: 接続チェックを実施する

Airbyte Protocolの中で2番目に実装する操作は、`check`オペレーションです。

このオペレーションは、ユーザーからの入力データを用いてデータソースに接続できるかどうかを確認します。この入力データには`spec.json` に記述された値が記入されているべきであるという事に注意してください。つまり`spec.json` に「ソースには `username` と `password` が必要」と書かれていた場合、設定オブジェクトは `{ "username": "airbyte", "password": "password123" }` と記載するべきです。そして設定オブジェクトに含まれる認証情報をもとに、ソースに接続できたかどうかを知らせる jsonオブジェクトを返すような実装をする必要があります。

今回取り上げたAPIはクレデンシャルを必要としないので、これはかなり簡単なチェックです。その代わりに、ユーザーが入力した `base` 通貨が正規の通貨であるかどうか検証してみましょう。`source.py`の中に、以下の自動生成されたソースがあります。


```python
class SourcePythonHttpTutorial(AbstractSource):

    def check_connection(self, logger, config) -> Tuple[bool, any]:
        """
        TODO: 接続チェックを実装し、ユーザが入力したデータをAPI接続に使用できるかどうかを検証します。
        https://github.com/airbytehq/airbyte/blob/master/airbyte-integrations/connectors/source-stripe/source_stripe/source.py#L232 を参照してください。
        :param config: spec.jsonに準拠した、ユーザ入力データ
        :param logger: ロガー
        :return Tuple[bool, any]: (True, None) は入力データでAPI接続に成功した場合で、(False, error) はそうでない場合。
        """
        return True, None

...
```

下記の通り、入力された通貨が実際の通貨であることを検証するよう実装を変更します。

```python
    def check_connection(self, logger, config) -> Tuple[bool, any]:
        accepted_currencies = {"USD", "JPY", "BGN", "CZK", "DKK"}  # assume these are the only allowed currencies
        input_currency = config['base']
        if input_currency not in accepted_currencies:
            return False, f"Input currency {input_currency} is invalid. Please input one of the following currencies: {accepted_currencies}"
        else:
            return True, None
```

この実装をテストするために、有効なデータと無効なデータの2つのオブジェクトを作成し、コネクタへの入力データとして与えてみましょう。

```text
echo '{"start_date": "2021-04-01", "base": "USD"}'  > sample_files/config.json
echo '{"start_date": "2021-04-01", "base": "BTC"}'  > sample_files/invalid_config.json
python main.py check --config sample_files/config.json
python main.py check --config sample_files/invalid_config.json
```

以下のような出力が表示されるはずです。

```text
> python main.py check --config sample_files/config.json
{"type": "CONNECTION_STATUS", "connectionStatus": {"status": "SUCCEEDED"}}

> python main.py check --config sample_files/invalid_config.json
{"type": "CONNECTION_STATUS", "connectionStatus": {"status": "FAILED", "message": "Input currency BTC is invalid. Please input one of the following currencies: {'DKK', 'USD', 'CZK', 'BGN', 'JPY'}"}}
```

開発中はデフォルトで `secrets` ディレクトリが gitignore されるため、`secrets` を含むデータは `secrets/config.json` に保存することをお勧めします。

# Step 5: ストリームのスキーマを宣言する

Airbyte Protocol の `discover` メソッドは `AirbyteCatalog` を返します。これは、あるコネクタが出力するすべてのストリームとそのスキーマを宣言したオブジェクトです。また、ストリームがサポートする同期モード(full refresh または incremental)も宣言されます。詳細は、[catalog tutorial](https://docs.airbyte.io/understanding-airbyte/beginners-guide-to-catalog)を参照してください。

Airbyte CDKでは、これは簡単な作業です。コネクタの各ストリームについて、次のことが必要です。
1. `source.py` で `HttpStream` を継承した python `class` を作成します。
1. `<stream_name>.json` ファイルを `source_<name>/schemas/` ディレクトリに配置します。
ファイル名はスキーマを記述するストリームのスネークケース名で、内容はそのストリームからの出力を記述する JsonSchema である必要があります。

それでは`source.py` に `HttpStream` を継承したクラスを作成しましょう。様々なコネクタ機能を実装するために必要なことを記述した、膨大なコメントを持つクラスがあることに気づくでしょう。必要に応じて、これらのクラスを自由に読んでください。しかし、このチュートリアルでは、生成されたクラスを削除するか、以下の実装に合うように編集して、ゼロからクラスを追加すると仮定しましょう。

まず、為替レート API から取得するデータを表すストリームを作成することから始めます。

```python
class ExchangeRates(HttpStream):
    url_base = "https://api.exchangeratesapi.io/"

    # NOOPとして設定します
    primary_key = None

    def next_page_token(self, response: requests.Response) -> Optional[Mapping[str, Any]]:
        # APIはページネーションを提供しないので、レスポンスに含まれるページがもうないことを示すためにNoneを返します
        return None

    def path(
        self, 
        stream_state: Mapping[str, Any] = None, 
        stream_slice: Mapping[str, Any] = None, 
        next_page_token: Mapping[str, Any] = None
    ) -> str:
        return ""  # TODO

    def parse_response(
        self,
        response: requests.Response,
        stream_state: Mapping[str, Any],
        stream_slice: Mapping[str, Any] = None,
        next_page_token: Mapping[str, Any] = None,
    ) -> Iterable[Mapping]:
        return None  # TODO
```

この実装は完全に空っぽであり、実際には何もしていないことに注意してください。この点については次のステップで説明します。とにかく現時点ではこのストリームのスキーマを宣言したいのです。これはコネクタが `streams` メソッドから返すことで出力するストリームとして宣言します。

```python
from airbyte_cdk.sources.streams.http.auth import NoAuth

class SourcePythonHttpTutorial(AbstractSource):

    def check_connection(self, logger, config) -> Tuple[bool, any]:
        ...

    def streams(self, config: Mapping[str, Any]) -> List[Stream]:
        # NoAuthは、このAPIに認証が必要ないことを意味するだけであり、完全性を期すために含まれています。
        # 認証が必要ない場合はスキップします。
        # 他の認証方法としては、APIトークンやOauth2が利用可能です。 
        auth = NoAuth()  
        return [ExchangeRates(authenticator=auth)]
```

このストリームをコードで作成した後、`schemas/` フォルダに `exchange_rates.json` というファイルを置きます。出力スキーマを記述した JSON ファイルを [ここから](https://github.com/airbytehq/airbyte/blob/master/airbyte-cdk/python/docs/tutorials/http_api_source_assets/exchange_rates.json) ダウンロードして、`schemas/` に置いておくと便利です。

スキーマファイル `.json` を配置した状態で、コネクタがこのスキーマを見つけて、有効なカタログを生成できるかどうか見てみましょう。

```text
python main.py discover --config sample_files/config.json
```

このような出力が表示されるはずです。

```text
{"type": "CATALOG", "catalog": {"streams": [{"name": "exchange_rates", "json_schema": {"$schema": "http://json-schema.org/draft-04/schema#", "type": "object", "properties": {"base": {"type": "string"}, "rates": {"type": "object", "properties": {"GBP": {"type": "number"}, "HKD": {"type": "number"}, "IDR": {"type": "number"}, "PHP": {"type": "number"}, "LVL": {"type": "number"}, "INR": {"type": "number"}, "CHF": {"type": "number"}, "MXN": {"type": "number"}, "SGD": {"type": "number"}, "CZK": {"type": "number"}, "THB": {"type": "number"}, "BGN": {"type": "number"}, "EUR": {"type": "number"}, "MYR": {"type": "number"}, "NOK": {"type": "number"}, "CNY": {"type": "number"}, "HRK": {"type": "number"}, "PLN": {"type": "number"}, "LTL": {"type": "number"}, "TRY": {"type": "number"}, "ZAR": {"type": "number"}, "CAD": {"type": "number"}, "BRL": {"type": "number"}, "RON": {"type": "number"}, "DKK": {"type": "number"}, "NZD": {"type": "number"}, "EEK": {"type": "number"}, "JPY": {"type": "number"}, "RUB": {"type": "number"}, "KRW": {"type": "number"}, "USD": {"type": "number"}, "AUD": {"type": "number"}, "HUF": {"type": "number"}, "SEK": {"type": "number"}}}, "date": {"type": "string"}}}, "supported_sync_modes": ["full_refresh"]}]}}
```

簡単でしょう！これでコネクタは、ストリームのスキーマを宣言する方法を知ることができます。今回はソースが単純なのでストリームを1つだけ宣言しますが、多くのストリームがあったとしても原理は全く同じです。

スキーマを動的に定義することもできますが、それはこのチュートリアルの範囲外です。詳しくは[schema docs](../../cdk-python/full-refresh-stream.md#defining-the-streams-schema) を参照してください。


# Step 6: ストリームを読み込むための機能を実装する

スキーマを記述するのは良いことですが、どこかからデータを読み始めなければなりません。というわけで、さっそく取り掛かりましょう。その前に、これからやろうとしていることを説明します。

`HttpStream` スーパークラスは、[コンセプトのドキュメント](https://docs.airbyte.io/connector-development/cdk-python/http-streams) にあるように、HTTPエンドポイントからのデータの読み込みを容易にするものです。このクラスには、以下のような組み込み関数やヘルパーが含まれています。

* 認証
* ページネーション
* レート制限や一時的なエラーの処理
* その他便利な機能

このような機能を実現するためには、いくつかの入力を提供する必要があります。

* ヒットさせたいエンドポイントのURLとパス
* APIからのレスポンスのパース方法
* ページネーションの実行方法

オプションとして、リクエストをカスタマイズするための追加入力を提供することができます。

* リクエストパラメーターとヘッダー
* レート制限エラーをどのように認識し、どの程度待機させるか
* HTTPメソッドとリクエストボディ（必要に応じて）
* exponential backoff（指数関数的に処理のリトライ間隔を後退）ポリシーを設定する

バックオフポリシーのオプション

* `retry_factor` Exponential Backoff Policy の係数を指定します。
* `max_retries` バックオフポリシーの再試行回数の最大値を指定します(デフォルトは5回)
* `raise_on_http_errors`  False に設定すると、HTTP コード例外の発生を抑制します (デフォルトは True)。


他にも多くのカスタマイズ可能なオプションがあり、それらは [`airbyte_cdk.sources.streams.http.HttpStream`](https://github.com/airbytehq/airbyte/blob/master/airbyte-cdk/python/airbyte_cdk/sources/streams/http/http.py) クラスで見つけることができます。

為替レートAPIからデータを読み込むために、ストリームが作業を行うために必要な情報を入力します。まず、前日の為替レートを読み取るだけの基本的な読み取りを実装し、次にストリームスライスを使用してインクリメンタルな同期を実装します。

まず、`/latest`エンドポイントを使用して、直近の為替レートのデータを取得します。

```python
class ExchangeRates(HttpStream):
    url_base = "https://api.exchangeratesapi.io/"

    primary_key = None

    def __init__(self, base: str, **kwargs):
        super().__init__()
        self.base = base


    def path(
        self, 
        stream_state: Mapping[str, Any] = None, 
        stream_slice: Mapping[str, Any] = None, 
        next_page_token: Mapping[str, Any] = None
    ) -> str:
        # "/latest "パスは最新の為替レートを提供します
        return "latest"  

    def request_params(
            self,
            stream_state: Mapping[str, Any],
            stream_slice: Mapping[str, Any] = None,
            next_page_token: Mapping[str, Any] = None,
    ) -> MutableMapping[str, Any]:
        # API仕様上、クエリパラメータに基本通貨を含める必要があるため、このメソッドでそれを行います。
        return {'base': self.base}

    def parse_response(
            self,
            response: requests.Response,
            stream_state: Mapping[str, Any],
            stream_slice: Mapping[str, Any] = None,
            next_page_token: Mapping[str, Any] = None,
    ) -> Iterable[Mapping]:
        # 応答はシンプルなJSONで、そのスキーマはストリームのスキーマと正確に一致します。
        # なので、レスポンスを含むリストを返すだけです。
        return [response.json()]

    def next_page_token(self, response: requests.Response) -> Optional[Mapping[str, Any]]:
        # APIはページネーションを提供しません。
        # そこで、レスポンスにもうページがないことを示すために None を返します。
        return None
```

コード行数が多く感じるかもしれませんが、それはメソッドに多くの（今のところ使われていない）パラメータがあるからです（これらはPythonの`**kwargs`で隠すことができますが、今のところ気にしないでください）。本当に数行の "重要な "コードを追加しただけです。

1. コンストラクタ `__init__` を追加し、クエリに使用する `base` 通貨を格納するようにしました。 
1. `return {'base': self.base}` を使用して、ユーザーが入力した `base` に基づいて `?base=<base-value>` クエリパラメータをリクエストに追加しています。
1. `return [response.json()]` で、APIからのレスポンスをパースして、スキーマ `.json` ファイルのスキーマと一致させています。
1. `return "latest"` で、APIの `/latest` エンドポイントをヒットして、最新の為替レートデータを取得したいことを示します。

では、ユーザーが入力した `base` パラメータをストリームクラスに渡しましょう。

```python
def streams(self, config: Mapping[str, Any]) -> List[Stream]:
        auth = NoAuth()
        return [ExchangeRates(authenticator=auth, base=config['base'])]
```

これでAPIを叩く準備ができました！

実行する為には、[ConfiguredCatalog](https://docs.airbyte.io/understanding-airbyte/beginners-guide-to-catalog) が必要です。[ここに用意されています](https://github.com/airbytehq/airbyte/blob/master/airbyte-cdk/python/docs/tutorials/http_api_source_assets/configured_catalog.json) -- これをダウンロードして `sample_files/configured_catalog.json` に置いてください。そして、実行してください。

```text
 python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json
```

いくつかの出力行が表示されます。そのうちの1つは、APIからのレコードです。

```text
{"type": "RECORD", "record": {"stream": "exchange_rates", "data": {"base": "USD", "rates": {"GBP": 0.7196938353, "HKD": 7.7597848573, "IDR": 14482.4824162185, "ILS": 3.2412081092, "DKK": 6.1532478279, "INR": 74.7852709971, "CHF": 0.915763343, "MXN": 19.8439387671, "CZK": 21.3545717832, "SGD": 1.3261894911, "THB": 31.4398014067, "HRK": 6.2599917253, "EUR": 0.8274720728, "MYR": 4.0979726934, "NOK": 8.3043442284, "CNY": 6.4856433595, "BGN": 1.61836988, "PHP": 48.3516756309, "PLN": 3.770872983, "ZAR": 14.2690111709, "CAD": 1.2436905254, "ISK": 124.9482829954, "BRL": 5.4526272238, "RON": 4.0738932561, "NZD": 1.3841125362, "TRY": 8.3101365329, "JPY": 108.0182043856, "RUB": 74.9555647497, "KRW": 1111.7583781547, "USD": 1.0, "AUD": 1.2840711626, "HUF": 300.6206040546, "SEK": 8.3829540753}, "date": "2021-04-26"}, "emitted_at": 1619498062000}}
```

たった数行のコードで、データを読み込むストリームが完成しました。

ここで終わりにすることも出来ますが、続いてインクリメンタルな同期を追加することにも挑戦してみましょう。

## インクリメンタル同期を追加する

インクリメンタル同期を追加するために、いくつかのことを行います。 

1. ユーザが入力した `start_date` パラメータをストリームに渡します。
1. ストリームの `cursor_field` を宣言します。
1. `get_updated_state` メソッドを実装します。
1.  `stream_slices` メソッドを実装します。
1.  `path` メソッドを更新して、為替レートを取得する日付を指定します。
1.  ストリームのテスト時に `incremental` 同期を使用するよう、設定されたカタログを更新します。

これらのメソッドがそれぞれ何をするのか説明します。まず、Airbyteでインクリメンタル同期がどのように動作するか、[docs on incremental](https://docs.airbyte.io/understanding-airbyte/connections/incremental-append) を読んで理解しておくとよいでしょう。

簡潔さを保つために、一つずつ編集しながら関数だけを表示することにします。

簡単なところから始めて、`start_date`を渡してみましょう。

```python
def streams(self, config: Mapping[str, Any]) -> List[Stream]:
        auth = NoAuth()
        # 文字列から日付をパースしてdatetimeオブジェクトに変換します
        start_date = datetime.strptime(config['start_date'], '%Y-%m-%d') 
        return [ExchangeRates(authenticator=auth, base=config['base'], start_date=start_date)]
```

また、コンストラクタにこのパラメータを追加して、`cursor_field`を宣言しましょう。

```python
from datetime import datetime, timedelta


class ExchangeRates(HttpStream):
    url_base = "https://api.exchangeratesapi.io/"
    cursor_field = "date"
    primary_key = "date"

    def __init__(self, base: str, start_date: datetime, **kwargs):
        super().__init__()
        self.base = base
        self.start_date = start_date
```

`cursor_field` を宣言すると、このストリームが現在インクリメンタル同期をサポートしていることがフレームワークに通知されます。次に `python main_dev.py discover --config sample_files/config.json` を実行すると、 `supported_sync_modes` フィールドに `incremental` も含まれていることが分かります。

しかし、インクリメンタルをサポートするだけでは不十分で、実際に状態を出力する必要があります。それは `dict` で、キーは `'date'` で、値はデータを同期させた最後の日の日付です。例えば、`{'date': '2021-04-26'}` は、コネクタが以前に4月26日までのデータを読んだので、4月26日以前のデータを再読み込みしてはいけないということを表します。

`ExchangeRates` クラスに `get_updated_state` メソッドを実装して、これを実現しましょう。

```python
    def get_updated_state(self, current_stream_state: MutableMapping[str, Any], latest_record: Mapping[str, Any]) -> Mapping[str, any]:
        # このメソッドは API から返された各レコードに対して一度だけ呼び出され、そのレコードのカーソルフィールドの値と現在の状態を比較します。
        # その後、更新された状態オブジェクトを返します。初めて同期を実行した場合や、状態が渡されなかった場合、 current_stream_state は None になります。
        if current_stream_state is not None and 'date' in current_stream_state:
            current_parsed_date = datetime.strptime(current_stream_state['date'], '%Y-%m-%d')
            latest_record_date = datetime.strptime(latest_record['date'], '%Y-%m-%d')
            return {'date': max(current_parsed_date, latest_record_date).strftime('%Y-%m-%d')}
        else:
            return {'date': self.start_date.strftime('%Y-%m-%d')}
```

この実装では、最新のレコードの日付と現在の状態の日付を比較し、その最大値を「新しい」状態オブジェクトとして取得します。

ストリーム状態が存在すれば、それに基づいてデータを取得すべき日付のリストを返すために、`stream_slices`メソッドを実装します。

```python
 def _chunk_date_range(self, start_date: datetime) -> List[Mapping[str, any]]:
        """
        開始日から現在までの各日付のリストを返します。
        戻り値はdicts {'date': date_string}のリストです。
        """
        dates = []
        while start_date < datetime.now():
            dates.append({'date': start_date.strftime('%Y-%m-%d')})
            start_date += timedelta(days=1)
        return dates

    def stream_slices(self, sync_mode, cursor_field: List[str] = None, stream_state: Mapping[str, Any] = None) -> Iterable[
        Optional[Mapping[str, any]]]:
        start_date = datetime.strptime(stream_state['date'], '%Y-%m-%d') if stream_state and 'date' in stream_state else self.start_date
        return self._chunk_date_range(start_date)
```

それぞれのスライスで、APIにHTTPリクエストが行われます。 `stream_slice` パラメータ (上記の `stream_slices` で作成したリストの要素) に含まれる情報を使用して、 `path` や `request_params` など、送信するリクエストに関する他の設定を行うことができます。ストリームスライスの詳細については、[the slicing docs](https://docs.airbyte.io/connector-development/cdk-python/stream-slices)を参照してください。

特定の日付のデータを取得するために、Exchange Rates APIではURLのパスコンポーネントとして日付を渡す必要があります。これを実現するために、`path`メソッドをオーバーライドしましょう。

```python
def path(self, stream_state: Mapping[str, Any] = None, stream_slice: Mapping[str, Any] = None, next_page_token: Mapping[str, Any] = None) -> str:
    return stream_slice['date']
```

これらの変更により、あなたの実装は[こちらのファイル](https://github.com/airbytehq/airbyte/blob/master/airbyte-integrations/connectors/source-python-http-tutorial/source_python_http_tutorial/source.py)のようになるはずです。

最後に、 `sample_files/configured_catalog.json` にある `sync_mode` フィールドを `incremental` に変更します。

```text
"sync_mode": "incremental",
```

これで、インクリメンタルシンクの実装ができたはずです。

試してみましょう。

```text
python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json
```

沢山の `RECORD` メッセージと `STATE` メッセージが表示されるはずです。インクリメンタル同期が機能していることを確認するために、入力された状態をコネクタに戻し、再度実行してください。

```text
# 最新の状態をsample_files/state.jsonに保存します。
python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json | grep STATE | tail -n 1 | jq .state.data > sample_files/state.json

# 最新状態メッセージで読み取り操作を実行します
python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json --state sample_files/state.json
```

最後の日付のレコードだけが同期されていることがわかるはずです。Airbyteはレコードの配信を最低1回にする必要があるので、最後のレコードを2回繰り返すのは問題ありません。

これでコネクタにインクリメンタル同期が実装されました。

# Step 7: Airbyteのコネクタを使用する

コネクタを使用するには、`docker build . -t airbyte/source-python-http-example:dev`を実行してコンテナ用の docker イメージをビルドします。その後、[building a Python source tutorial](https://docs.airbyte.io/connector-development/tutorials/building-a-python-source#step-11-add-the-connector-to-the-api-ui)の説明に従って、Airbyte UI でコネクタを使用するために、適宜名前を置き換えて使用するようにします。

注意：ビルドした docker イメージは、Airbyte ノードで動作している `docker` デーモンからアクセス可能である必要があります。このチュートリアルをローカルで行う場合は、この手順で十分です。そうでない場合は、Dockerhub に Docker イメージをプッシュする必要があるかもしれません。


# Step 8: 単体テストまたは結合テストの作成

## 単体テスト

関連するユニットテストを `unit_tests` ディレクトリに追加してください。**ユニットテストはどんなsecretにも依存してはいけません**。

テストは `python -m pytest -s unit_tests` を使って実行することができます。

## 結合テスト

統合テストは `integration_tests` ディレクトリに置き、[pytest](https://docs.pytest.org/en/6.2.x/goodpractices.html#conventions-for-python-test-discovery)で検出できるようにします。

## 標準テスト

標準テストは、すべてのAirbyteソースコネクタが通過しなければならない、Airbyteが提供する固定テストセットです。Airbyte にコネクタを提出する場合のみ必要ですが、どのような場合でも役に立つと思います。[コネクタのテスト](https://docs.airbyte.io/connector-development/testing-connectors)を参照してください。

このコネクタをAirbyteのデフォルトコネクタとして登録する場合は、[Pythonソースチェックリスト](https://docs.airbyte.io/connector-development/tutorials/building-a-python-source#step-8-set-up-standard-tests)のステップ8以降に進んでください。



