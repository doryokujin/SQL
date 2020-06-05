# Lesson 01. SELECTによる「単純」抽出

## 全行，全カラムの抽出 [ SELECT * ]
「SELECT *」という記述は，FROM節で指定したテーブルにあるすべてのカラムを個別に指定することなく抽出するということを意味しています。
```sql
SELECT * 
FROM sample_accesslog
```

|td_client_id|td_global_id             |td_title        |td_browser|td_host|td_path            |td_url    |td_referrer|td_ip|td_os     |td_language|time      |
|------------|-------------------------|----------------|----------|-------|-------------------|----------|-----------|-----|----------|-----------|----------|
|59523c90-2724-4d52-9157-1b93cd89f2a2|                         |リソース - Treasure Data|Chrome    |www.treasuredata.com|/jp/resources      |https://www.treasuredata.com/jp/resources|https://www.treasuredata.com/jp/|133.250.236.113|Windows 7 |ja         |1465977533|
|37c30508-18fa-422b-cdca-48ef68cbd29d|                         |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Mobile Safari|www.treasuredata.com|/jp/               |https://www.treasuredata.com/jp/|https://www.google.co.jp/|103.5.140.189|iOS       |ja-jp      |1465977276|
|e6d199fe-3367-4fb7-ab41-5422e30ca5c9|                         |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Mobile Safari|www.treasuredata.com|/jp/               |https://www.treasuredata.com/jp/?gclid=CMX_uou8qc0CFdgmvQodBrMD5w|           |106.133.82.194|iOS       |ja-jp      |1465974373|

SELECT文では，まず引用するテーブル（参照するデータ）をFROM節によって指定します。そして「SELECT time, td_title,...」という形で，テーブル内のどのカラムを抽出してくるのかを選択します。すべてのカラムを取得したい場合，1つひとつのカラムの記述が大変になるので，その場合には「*」のみで済ますという省略法があります。

「SELECT」から始まるSQLは，データを「抽出」するために使われる『文』（Statement）です。文はSQLの1つの実行単位を意味し，1つの文の実行によって1つの結果テーブルが得られます。
また，SELECT文内における「FROM」や「WHERE」等の構成要素は『節（句）』（Clause）と呼ばれます。節は先程の「（図）実行順序イメージ」における各ブロックとほぼ一致します。

## 全行，全カラムの抽出 [ SELECT col ]

テーブル内に存在するカラム名のうち，必要なものだけを列挙してデータを抽出しましょう。一般にテーブルにはたくさんのカラムがありますが，一度に全部利用することは稀です。パフォーマンス的にも非効率なので，「*」はあまり使用しないようにしましょう。

```sql
SELECT time, td_client_id, td_title
FROM sample_accesslog
```
|time   |td_client_id             |td_title        |
|-------|-------------------------|----------------|
|1466410889|2705c701-3b7c-4ef3-d88d-50529a3d1060|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1466411086|a1265a65-70a0-4ee2-98fe-6671146666ee|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1466412618|83b53ce3-7c66-46aa-8c4d-cbec9e52bb86|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|

「SELECT」の直後に特定のカラム名を複数選ぶ（カンマで区切ることを忘れないでください）ことで，必要なカラムだけが選択された結果テーブルを抽出することができます。
カンマが抜けてしまっていても，実はエラーが出ません。しかしながら，抜けているカンマより前にあるカラムが抽出されないことになります。長いクエリになるとこういった凡ミスが起こり得るので，日頃から注意しておきましょう。

```sql
SELECT time, td_client_id td_title --カンマが抜けてる！
FROM sample_accesslog
```
|time   |td_title                 |
|-------|-------------------------|
|1465892286|daa933f7-6c39-4ce5-b610-d2368950c176|
|1465892317|7f47d05f-bd12-4553-e69c-763064738631|
|1465892324|93b18d92-095e-46f3-ece9-a00369945e97|

最も単純なSELECT文を見ましたが，ここで重要なのは，「SQLは抽出したい列（カラム）を指定してテーブルを見ることに特化されており，行番号を指定して見るようにはできていない」ことです。実際，カラム名はありますが行名はありません。

行番号はありますが，SQLにおいて行番号の指定はできません。もとより，データの抽出のされ方に応じて必ずしも毎回同じ行番号に同じ値のレコードがくるとは限らず，行番号はあくまでも1つのSELECT文におけるテンポラリな番号でしかありません（自分でテーブルのカラムとして行番号を入れている場合は別）。

クエリ中で行番号が意識されるのは，Window関数などの特殊なケースのみになります。

## カラムに別名を付ける [ SELECT col AS name ]

元々テーブルに入っているカラム名のスペルが間違っていたり，適切な内容を示唆するものでなかったり，抽出の意図にそぐわない名前が付いていたりする場合に，自分が実行するクエリの中においてカラム名を変更して扱いたい，もしくは出力したいことが多々あります。

以下のクエリでは，td_client_idを重複して2つ抽出する際に，その区別のためにuid1およびuid2という別名を与えています。

```sql
SELECT td_client_id AS uid1, td_client_id AS uid2, td_title AS title
FROM sample_accesslog
```
|uid1   |uid2                     |title                     |
|-------|-------------------------|--------------------------|
|6dc74bd9-313d-42be-c4ba-9072f4c15312|6dc74bd9-313d-42be-c4ba-9072f4c15312|サービス概要 - Treasure Data    |
|e3dc64f2-93a8-4c0f-c9b3-2a8170065578|e3dc64f2-93a8-4c0f-c9b3-2a8170065578|サービス概要 - Treasure Data    |
|7f47d05f-bd12-4553-e69c-763064738631|7f47d05f-bd12-4553-e69c-763064738631|連携するシステム一覧 - Treasure Data|


カラムの別名は，集約計算などで新しいカラムが作られる場合にも付ける必要が出てきます。

## 抽出件数を絞る [ LIMIT n ]
大規模なテーブルにおいては，毎回全レコードを抽出するようなことはしません。多くの場合，WHEREで時間範囲を指定して対象を絞るか，以下のように取得する件数を決めておきます。
```sql
SELECT time, td_client_id, td_title
FROM sample_accesslog
LIMIT 5 --結果を5件のみ取得
```
|time   |td_client_id             |td_title                  |
|-------|-------------------------|--------------------------|
|1467099783|33c61ec6-aa33-40bb-8f7d-326a2cba9ed7|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1467100686|b40f644b-f816-4dc6-9131-aa7f0e2044d1|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1467100698|b40f644b-f816-4dc6-9131-aa7f0e2044d1|お客様事例一覧 - Treasure Data   |
|1467097949|7d34de55-b53d-472f-d8a7-8eaaa43d5c03|トレジャーデータサービスが、 株式会社MonotaRO（モノタロウ）のデータ分析基盤として導入 - プレスリリース - Treasure Data|
|1467097832|3513d40c-97c3-49cb-b087-f90cb80a4c26|企業情報 - Treasure Data      |

このようにSELECT文の最後の節にLIMIT節を添えることで，抽出結果の行数を絞り込むことができます。

LIMIT節の実行順序は，常にいかなる節よりも最後にくることになっています。
LIMIT節を使う利点は，表示する行数を絞り込む際に単純な抽出クエリであれば全レコードが探索されず必要な件数の結果レコードが得られた時点で処理が終了するので，プラットフォームへの負荷削減と実行時間削減の効果があることです。

トライアンドエラーの段階で何度もクエリを実行する際には，時間と負荷の2つの軽減の意味で，LIMIT節を積極的に使うようにしてください。ただし，このようなLIMIT節の効用があるのは単純な「探索」の場合のみです。後述する「ORDER BY」や集約関数を用いる場合などではだいぶ薄まってしまいます。

## 抽出結果を並び替える [ ORDER BY col ]
クエリの結果を（保存や書き込みではなく）その場で見たい場合には，あるカラムに関して並び替えられた状態にするほうが便利なことが多々あります。

```sql
SELECT time, td_client_id, td_title
FROM sample_accesslog
ORDER BY td_client_id ASC --ASCは昇順（小さい順）
LIMIT 10                  --LIMIT節は ORDER BY節より後
```
|time      |td_client_id|td_title                                          |
|----------|------------|--------------------------------------------------|
|1461454166|000077fb-2c93-4cd7-d9d0-293866aaec31|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1461454040|000077fb-2c93-4cd7-d9d0-293866aaec31|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1461454057|000077fb-2c93-4cd7-d9d0-293866aaec31|企業情報 - Treasure Data                              |


ORDER BY節は，一連の「SELECT FROM」で抽出されたレコードに対して，結果を特定のカラム名の値で並び替える役割を果たします。その際，昇順か降順のどちらにするかは，ASCもしくはDESCオプションで指定できます。どちらのオプションも指定しない場合はASCになります。なお，LIMIT節と併用する場合には，ORDER BY節はその前にないといけません。

改めて先の結果を見てみましょう。ORDER BYにより，td_client_idに関して降順で整列された結果が返ってきました。td_client_idが同じレコードは連続して現れていますが，これらのレコード間では順序がつかないので，並び順には保証がありません。もし毎回の実行結果にこの意味でも同じ並びを保証したいならば，さらに他のカラム（例えばtimeカラム）をORDER BYに加える必要があります。

以下は，DESC（降順）による並び替えの例です。

```sql
SELECT time, td_client_id, td_title
FROM sample_accesslog
ORDER BY td_client_id DESC --DESCは降順（大きい順）
LIMIT 10                   --ORDER BY節との併用では負荷削減効果は薄い
```
|time      |td_client_id                        |td_title                                          |
|----------|------------------------------------|--------------------------------------------------|
|1466494001|fffe877a-759d-4f67-a5c4-ac5ea1ee78b5|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1466494000|fffe877a-759d-4f67-a5c4-ac5ea1ee78b5|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1463156253|fffe6e62-edf6-45d8-b052-c119c3752c8f|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|


今回のクエリでは，DESCにより降順で並び替えを行っています。同じデータソースに対して先程と結果が異なっていることが明らかですね。

なお，どちらの例でもLIMIT 10として10件のレコードのみを抽出してきましたが，ORDER BY節とLIMIT節の併用では以下の点に注意が必要です。

- ORDER節とLIMIT節の順序に注意しましょう。逆だとエラーになります。
- ORDER BYが実行されると全レコード（またはWHERE節などの条件で絞り込まれた全レコード）が読み込まれてソートアルゴリズムが走ります。よって，たとえLIMIT節で少ない件数を指定していても負荷の重いクエリになる場合があります。
- ORDER BYは分散しません。シングルノードへレコードが集約（その際に通信コストもかかる）されてから処理されるため，重い処理となりがちです。
- それでも思わぬ膨大な件数の結果が返ってくるのを避けるため，いつでもLIMIT節を添える心得は大切です。
- なお，HiveではORDER BYしたいカラムはSELECT節にないといけません。
- ORDER節よりもLIMIT節の実行順序が後であることに注意してください。

ORDER BYの後には複数のカラムを指定でき（記述する順序は重要），各々について降順か昇順かを指定できます。
```sql
SELECT time, td_client_id, td_title
FROM sample_accesslog
ORDER BY td_client_id ASC, time DESC
LIMIT 10
```
|time      |td_client_id                        |td_title                                          |
|----------|------------------------------------|--------------------------------------------------|
|1461454166|000077fb-2c93-4cd7-d9d0-293866aaec31|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1461454142|000077fb-2c93-4cd7-d9d0-293866aaec31|採用情報 - Treasure Data                              |
|1461454057|000077fb-2c93-4cd7-d9d0-293866aaec31|企業情報 - Treasure Data                              |

この例の挙動は，まずtd_client_idについて昇順に並び替えられ，次にtimeについて降順に並び替えられます。各td_client_idに対してtimeの値がユニークであるならば，この並び替えによる結果もユニーク（実行のたびにレコードの順序が変わらない）になります。

## 特定のカラムのユニークな値/組合せのみ抽出 [ SELECT DISTINCT col ]

SELECT DISTINCT節は，1つのカラムのみを指定する場合に，そのカラムの値で重複を省いたものを抽出してくれます。やや特殊な存在なので，慣れないうちはスルーしてもかまいません。

なお，DISTINCTはシングルノードでしか動作しません（つまり分散しない）。そのためパフォーマンスが悪く，メモリも消費しがちです。

```sql
SELECT DISTINCT td_title
FROM sample_accesslog
ORDER BY td_title
LIMIT 10
```
|td_title  |
|----------|
|企業情報 - Treasure Data|
|2014年の事業戦略を発表 - プレスリリース - Treasure Data|
|2015年事業戦略発表 デジタルおよびIoT事業を中心としたグローバル市場拡大を強化 - プレスリリース - Treasure Data|


上記の結果には重複するtd_titleがないことがわかります。

SELECT DISTINCT節の後に複数のカラムをカンマ区切りで指定したクエリも記述できます。

```sql
SELECT DISTINCT td_title, td_os, td_browser
FROM sample_accesslog
ORDER BY td_title, td_os, td_browser
LIMIT 10
```
|td_title  |td_os      |td_browser|
|----------|-----------|----------|
|企業情報 - Treasure Data|Windows 8.1|Firefox   |
|2014年の事業戦略を発表 - プレスリリース - Treasure Data|Other      |Googlebot |
|2014年の事業戦略を発表 - プレスリリース - Treasure Data|Windows 7  |Chrome    |

SELECT DISTINCTの後に複数のカラムを指定した場合は，データソースの中でそれら複数のカラムの値が存在しうる組合せを列挙してくれます。結果の赤四角で囲んだ部分に着目すると，1つの td_titleの値に対してtd_osとtd_browserで取りうる組合せが4つあることがわかります。逆に言えば，今回のデータソースの中にそれ以外の組合せは存在しないことがわかります。

## 抽出した結果を続けて参照する [ SELECT FROM (SELECT) ]

ここではFROM節の使い方について別の例を紹介します。今までFROM節にはデータベースのテーブル名を指定していましたが，SELECT節で得られた結果もまた別のSELECT節で利用できます。その際，FROM節の内側にSELECT節を記述して「（）」で包むことで，外側のSELECT節でその結果を参照できるようになります。

```sql
SELECT time, td_client_id
FROM
(
  SELECT time, td_client_id, td_title
  FROM sample_accesslog
)
ORDER BY td_client_id ASC
LIMIT 10
```
|time      |td_client_id|
|----------|------------|
|1461454166|000077fb-2c93-4cd7-d9d0-293866aaec31|
|1461454040|000077fb-2c93-4cd7-d9d0-293866aaec31|
|1461454057|000077fb-2c93-4cd7-d9d0-293866aaec31|

## このLessonで注意したい点まとめ
最後に，このLessonに関して覚えておいていただきたい事項を改めて列挙します。
- SELECT節ではできるだけ「*」を使用せず，必要なカラム名のみを列挙するようにしましょう。
  - （WHY?）抽出するカラム数が少なければ少ないほどパフォーマンスが出るので，全カラム抽出の「*」は悪手にほかなりません。
- クエリの試し打ちの際は必ずLIMITで取得件数を制御しましょう。
  - （WHY?）例えばデータセットが100億件あるとすると，LIMITなしの抽出クエリはそのすべてのレコードを抽出しようとするので，時間，負荷ともに大変なことになります。
- 取得する結果がいつも同じ順序である保証はありません。
  - （WHY?）LIIMIT 10などで取得数を絞ったクエリの実行結果がいつも同じ10件のレコードであることは保証されていません。常に同じレコード結果が返ってくることを期待する場合には「ORDER BY」を使用しましょう。ただし，ORDER BYはシングルノードで処理されるのでパフォーマンスが出ず，多用も厳禁です。
