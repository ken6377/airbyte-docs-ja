# Python CDK: Creating a HTTP API Source

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
        # NoAuth just means there is no authentication required for this API and is included for completeness.
        # Skip passing an authenticator if no authentication is required.
        # Other authenticators are available for API token-based auth and Oauth2. 
        auth = NoAuth()  
        return [ExchangeRates(authenticator=auth)]
```

Having created this stream in code, we'll put a file `exchange_rates.json` in the `schemas/` folder. You can download the JSON file describing the output schema [here](https://github.com/airbytehq/airbyte/blob/master/airbyte-cdk/python/docs/tutorials/http_api_source_assets/exchange_rates.json) for convenience and place it in `schemas/`.

With `.json` schema file in place, let's see if the connector can now find this schema and produce a valid catalog:

```text
python main.py discover --config sample_files/config.json
```

you should see some output like:

```text
{"type": "CATALOG", "catalog": {"streams": [{"name": "exchange_rates", "json_schema": {"$schema": "http://json-schema.org/draft-04/schema#", "type": "object", "properties": {"base": {"type": "string"}, "rates": {"type": "object", "properties": {"GBP": {"type": "number"}, "HKD": {"type": "number"}, "IDR": {"type": "number"}, "PHP": {"type": "number"}, "LVL": {"type": "number"}, "INR": {"type": "number"}, "CHF": {"type": "number"}, "MXN": {"type": "number"}, "SGD": {"type": "number"}, "CZK": {"type": "number"}, "THB": {"type": "number"}, "BGN": {"type": "number"}, "EUR": {"type": "number"}, "MYR": {"type": "number"}, "NOK": {"type": "number"}, "CNY": {"type": "number"}, "HRK": {"type": "number"}, "PLN": {"type": "number"}, "LTL": {"type": "number"}, "TRY": {"type": "number"}, "ZAR": {"type": "number"}, "CAD": {"type": "number"}, "BRL": {"type": "number"}, "RON": {"type": "number"}, "DKK": {"type": "number"}, "NZD": {"type": "number"}, "EEK": {"type": "number"}, "JPY": {"type": "number"}, "RUB": {"type": "number"}, "KRW": {"type": "number"}, "USD": {"type": "number"}, "AUD": {"type": "number"}, "HUF": {"type": "number"}, "SEK": {"type": "number"}}}, "date": {"type": "string"}}}, "supported_sync_modes": ["full_refresh"]}]}}
```

It's that simple! Now the connector knows how to declare your connector's stream's schema. We declare only one stream since our source is simple, but the principle is exactly the same if you had many streams.

You can also dynamically define schemas, but that's beyond the scope of this tutorial. See the [schema docs](../../cdk-python/full-refresh-stream.md#defining-the-streams-schema) for more information.


# Step 6: Read Data

Describing schemas is good and all, but at some point we have to start reading data! So let's get to work. But before, let's describe what we're about to do:

The `HttpStream` superclass, like described in the [concepts documentation](../../cdk-python/http-streams.md), is facilitating reading data from HTTP endpoints. It contains built-in functions or helpers for:

* authentication
* pagination
* handling rate limiting or transient errors
* and other useful functionality

In order for it to be able to do this, we have to provide it with a few inputs:

* the URL base and path of the endpoint we'd like to hit
* how to parse the response from the API
* how to perform pagination

Optionally, we can provide additional inputs to customize requests:

* request parameters and headers
* how to recognize rate limit errors, and how long to wait \(by default it retries 429 and 5XX errors using exponential backoff\)
* HTTP method and request body if applicable
* configure exponential backoff policy

Backoff policy options:

* `retry_factor` Specifies factor for exponential backoff policy \(by default is 5\)
* `max_retries` Specifies maximum amount of retries for backoff policy \(by default is 5\)
* `raise_on_http_errors` If set to False, allows opting-out of raising HTTP code exception \(by default is True\)

There are many other customizable options - you can find them in the [`airbyte_cdk.sources.streams.http.HttpStream`](https://github.com/airbytehq/airbyte/blob/master/airbyte-cdk/python/airbyte_cdk/sources/streams/http/http.py) class.

So in order to read data from the exchange rates API, we'll fill out the necessary information for the stream to do its work. First, we'll implement a basic read that just reads the last day's exchange rates, then we'll implement incremental sync using stream slicing.

Let's begin by pulling data for the last day's rates by using the `/latest` endpoint:

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
        # The "/latest" path gives us the latest currency exchange rates
        return "latest"  

    def request_params(
            self,
            stream_state: Mapping[str, Any],
            stream_slice: Mapping[str, Any] = None,
            next_page_token: Mapping[str, Any] = None,
    ) -> MutableMapping[str, Any]:
        # The api requires that we include the base currency as a query param so we do that in this method
        return {'base': self.base}

    def parse_response(
            self,
            response: requests.Response,
            stream_state: Mapping[str, Any],
            stream_slice: Mapping[str, Any] = None,
            next_page_token: Mapping[str, Any] = None,
    ) -> Iterable[Mapping]:
        # The response is a simple JSON whose schema matches our stream's schema exactly, 
        # so we just return a list containing the response
        return [response.json()]

    def next_page_token(self, response: requests.Response) -> Optional[Mapping[str, Any]]:
        # The API does not offer pagination, 
        # so we return None to indicate there are no more pages in the response
        return None
```

This may look big, but that's just because there are lots of \(unused, for now\) parameters in these methods \(those can be hidden with Python's `**kwargs`, but don't worry about it for now\). Really we just added a few lines of "significant" code: 1. Added a constructor `__init__` which stores the `base` currency to query for. 2. `return {'base': self.base}` to add the `?base=<base-value>` query parameter to the request based on the `base` input by the user. 3. `return [response.json()]` to parse the response from the API to match the schema of our schema `.json` file. 4. `return "latest"` to indicate that we want to hit the `/latest` endpoint of the API to get the latest exchange rate data.

Let's also pass the `base` parameter input by the user to the stream class:

```python
def streams(self, config: Mapping[str, Any]) -> List[Stream]:
        auth = NoAuth()
        return [ExchangeRates(authenticator=auth, base=config['base'])]
```

We're now ready to query the API!

To do this, we'll need a [ConfiguredCatalog](../../../understanding-airbyte/beginners-guide-to-catalog.md). We've prepared one [here](https://github.com/airbytehq/airbyte/blob/master/airbyte-cdk/python/docs/tutorials/http_api_source_assets/configured_catalog.json) -- download this and place it in `sample_files/configured_catalog.json`. Then run:

```text
 python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json
```

you should see some output lines, one of which is a record from the API:

```text
{"type": "RECORD", "record": {"stream": "exchange_rates", "data": {"base": "USD", "rates": {"GBP": 0.7196938353, "HKD": 7.7597848573, "IDR": 14482.4824162185, "ILS": 3.2412081092, "DKK": 6.1532478279, "INR": 74.7852709971, "CHF": 0.915763343, "MXN": 19.8439387671, "CZK": 21.3545717832, "SGD": 1.3261894911, "THB": 31.4398014067, "HRK": 6.2599917253, "EUR": 0.8274720728, "MYR": 4.0979726934, "NOK": 8.3043442284, "CNY": 6.4856433595, "BGN": 1.61836988, "PHP": 48.3516756309, "PLN": 3.770872983, "ZAR": 14.2690111709, "CAD": 1.2436905254, "ISK": 124.9482829954, "BRL": 5.4526272238, "RON": 4.0738932561, "NZD": 1.3841125362, "TRY": 8.3101365329, "JPY": 108.0182043856, "RUB": 74.9555647497, "KRW": 1111.7583781547, "USD": 1.0, "AUD": 1.2840711626, "HUF": 300.6206040546, "SEK": 8.3829540753}, "date": "2021-04-26"}, "emitted_at": 1619498062000}}
```

There we have it - a stream which reads data in just a few lines of code!

We theoretically _could_ stop here and call it a connector. But let's give adding incremental sync a shot.

## Adding incremental sync

To add incremental sync, we'll do a few things: 1. Pass the `start_date` param input by the user into the stream. 2. Declare the stream's `cursor_field`. 3. Implement the `get_updated_state` method. 4. Implement the `stream_slices` method. 5. Update the `path` method to specify the date to pull exchange rates for. 6. Update the configured catalog to use `incremental` sync when we're testing the stream.

We'll describe what each of these methods do below. Before we begin, it may help to familiarize yourself with how incremental sync works in Airbyte by reading the [docs on incremental](../../../understanding-airbyte/connections/incremental-append.md).

To keep things concise, we'll only show functions as we edit them one by one.

Let's get the easy parts out of the way and pass the `start_date`:

```python
def streams(self, config: Mapping[str, Any]) -> List[Stream]:
        auth = NoAuth()
        # Parse the date from a string into a datetime object
        start_date = datetime.strptime(config['start_date'], '%Y-%m-%d') 
        return [ExchangeRates(authenticator=auth, base=config['base'], start_date=start_date)]
```

Let's also add this parameter to the constructor and declare the `cursor_field`:

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

Declaring the `cursor_field` informs the framework that this stream now supports incremental sync. The next time you run `python main_dev.py discover --config sample_files/config.json` you'll find that the `supported_sync_modes` field now also contains `incremental`.

But we're not quite done with supporting incremental, we have to actually emit state! We'll structure our state object very simply: it will be a `dict` whose single key is `'date'` and value is the date of the last day we synced data from. For example, `{'date': '2021-04-26'}` indicates the connector previously read data up until April 26th and therefore shouldn't re-read anything before April 26th.

Let's do this by implementing the `get_updated_state` method inside the `ExchangeRates` class.

```python
    def get_updated_state(self, current_stream_state: MutableMapping[str, Any], latest_record: Mapping[str, Any]) -> Mapping[str, any]:
        # This method is called once for each record returned from the API to compare the cursor field value in that record with the current state
        # we then return an updated state object. If this is the first time we run a sync or no state was passed, current_stream_state will be None.
        if current_stream_state is not None and 'date' in current_stream_state:
            current_parsed_date = datetime.strptime(current_stream_state['date'], '%Y-%m-%d')
            latest_record_date = datetime.strptime(latest_record['date'], '%Y-%m-%d')
            return {'date': max(current_parsed_date, latest_record_date).strftime('%Y-%m-%d')}
        else:
            return {'date': self.start_date.strftime('%Y-%m-%d')}
```

This implementation compares the date from the latest record with the date in the current state and takes the maximum as the "new" state object.

We'll implement the `stream_slices` method to return a list of the dates for which we should pull data based on the stream state if it exists:

```python
 def _chunk_date_range(self, start_date: datetime) -> List[Mapping[str, any]]:
        """
        Returns a list of each day between the start date and now.
        The return value is a list of dicts {'date': date_string}.
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

Each slice will cause an HTTP request to be made to the API. We can then use the information present in the `stream_slice` parameter \(a single element from the list we constructed in `stream_slices` above\) to set other configurations for the outgoing request like `path` or `request_params`. For more info about stream slicing, see [the slicing docs](../../cdk-python/stream-slices.md).

In order to pull data for a specific date, the Exchange Rates API requires that we pass the date as the path component of the URL. Let's override the `path` method to achieve this:

```python
def path(self, stream_state: Mapping[str, Any] = None, stream_slice: Mapping[str, Any] = None, next_page_token: Mapping[str, Any] = None) -> str:
    return stream_slice['date']
```

With these changes, your implementation should look like the file [here](https://github.com/airbytehq/airbyte/blob/master/airbyte-integrations/connectors/source-python-http-tutorial/source_python_http_tutorial/source.py).

The last thing we need to do is change the `sync_mode` field in the `sample_files/configured_catalog.json` to `incremental`:

```text
"sync_mode": "incremental",
```

We should now have a working implementation of incremental sync!

Let's try it out:

```text
python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json
```

You should see a bunch of `RECORD` messages and `STATE` messages. To verify that incremental sync is working, pass the input state back to the connector and run it again:

```text
# Save the latest state to sample_files/state.json
python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json | grep STATE | tail -n 1 | jq .state.data > sample_files/state.json

# Run a read operation with the latest state message
python main.py read --config sample_files/config.json --catalog sample_files/configured_catalog.json --state sample_files/state.json
```

You should see that only the record from the last date is being synced! This is acceptable behavior, since Airbyte requires at-least-once delivery of records, so repeating the last record twice is OK.

With that, we've implemented incremental sync for our connector!





# Step 7: Use the Connector in Airbyte

To use your connector in your own installation of Airbyte, build the docker image for your container by running `docker build . -t airbyte/source-python-http-example:dev`. Then, follow the instructions from the [building a Python source tutorial](../building-a-python-source.md#step-11-add-the-connector-to-the-api-ui) for using the connector in the Airbyte UI, replacing the name as appropriate.

Note: your built docker image must be accessible to the `docker` daemon running on the Airbyte node. If you're doing this tutorial locally, these instructions are sufficient. Otherwise you may need to push your Docker image to Dockerhub.


# Step 8: Test Connector

## Unit Tests

Add any relevant unit tests to the `unit_tests` directory. Unit tests should **not** depend on any secrets.

You can run the tests using `python -m pytest -s unit_tests`

## Integration Tests

Place any integration tests in the `integration_tests` directory such that they can be [discovered by pytest](https://docs.pytest.org/en/6.2.x/goodpractices.html#conventions-for-python-test-discovery).

## Standard Tests

Standard tests are a fixed set of tests Airbyte provides that every Airbyte source connector must pass. While they're only required if you intend to submit your connector to Airbyte, you might find them helpful in any case. See [Testing your connectors](../../testing-connectors/)

If you want to submit this connector to become a default connector within Airbyte, follow steps 8 onwards from the [Python source checklist](../building-a-python-source.md#step-8-set-up-standard-tests)



