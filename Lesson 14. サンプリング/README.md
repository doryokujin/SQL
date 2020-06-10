# Lesson 14. サンプリング

## TABLESAMPLE
RAND関数の有意義な活用先として，テーブルからの柔軟なランダムサンプリングが挙げられます。ここでの「ランダム」の意味は，「1回1回のサンプリングが独立して行われる（前回の結果に依存しない）」とします。
RAND関数を使う前に，まずは最も基本的で扱いやすい標準サンプリングクエリを紹介します。以下のクエリにより，sales_slipテーブルから500〜600レコードのランダムサンプリングが実行されます。
```sql
SELECT time, member_id, category, sub_category, goods_id
FROM sales_slip TABLESAMPLE BERNOULLI(0.01)
```
|time|member_id|category|sub_category|goods_id|
|----|---------|--------|------------|--------|
|1351601114|23717    |Home and Garden and Tools|Kitchen and Dining|597779  |
|1353414039|1839357  |Movies and Music and Games|Musical Instruments|603113  |
|1332844885|929459   |Automotive and Industrial|Motorcycle and Powersports|562491  |


FROM節に付与された「TABLESAMPLE BERNOULLI()」により，指定したテーブルからのランダムサンプリングが行われます。BERNOULLIとは「ベルヌーイ試行に基づいたサンプリング」を意味しています。ベルヌーイ試行とは，毎回独立して成功か失敗のいずれかの結果になる試行を繰り返していく操作のことで，代表的な例としてコイン投げがあります。コイン投げは，前回の結果に依存せず，歪んでいないコインならば表と裏が等しい確率で出現する試行だからです。

BERNOULLI関数の引数には「成功確率」をパーセントで指定します。コイン投げの場合は50を指定することになります。サンプリングの文脈に応じて取得したいサンプルの割合を指定することになります。1%のサンプルを得たいなら「1」，上のクエリ例のように0.01%のサンプルを得たいなら「0.01」と指定します。

BERNOULLI(0.01)とした場合には，各レコードごとに，成功確率0.01%のベルヌーイ試行によって成功とみなされたレコードが抽出されていきます。例えば10,000レコードあれば，期待値的には1つのレコードのみが成功になり，残り（99.99%）は失敗となって選択されません。なお，ベルヌーイ試行ではサンプリングを再実行すると成功回数（結果レコード数）が異なることに注意してください。

## WHERE節とRAND関数によるランダムサンプリング
RAND関数を使ったランダムサンプリングの最初の例は，テーブル全体から約0.01%のレコード数だけランダムサンプリングを行うシンプルなクエリです。
```sql
SELECT time, member_id, category, sub_category, goods_id
FROM sales_slip
WHERE RAND() <= 0.0001 -- 0.01%
ORDER BY time
```
|time|member_id|category|sub_category|goods_id|
|----|---------|--------|------------|--------|
|1108666186|99165    |Electronics and Computers|Video Games |105392  |
|1117873958|79502    |Sports and Outdoors|Boating and Water Sports|113720  |
|1118262459|574010   |Clothing and Shoes and Jewelry|Men         |114558  |



timeで並び替えているのは，毎回の実行で異なる結果が得られることを確認するためです。

上記のクエリのWHERE節の意味は「RANDによって一様に発生された0.0<x<=1.0の値のうち0.0001以下となるレコードが抽出される」ですから，このWHERE節は0.0001の割合（確率）で成立することになります。確率ですから，得られるサンプル自体も抽出される数も再実行のたびに異なることに注意してください。

## ORDER BY節とRAND関数によるランダムサンプリング

次に，テーブル全体からnレコードのランダムサンプリングを行いたい場合を考えましょう。毎回の得られるサンプル数を保証するのが先の例とは異なるところです（ただし毎回の結果には再現性がないことに注意してください）。面白いことに，ORDER BY節にRAND関数を添えるとランダムな並び替えが実行されるので，LIMIT節で欲しいレコード数を指定すれば簡単に目的を達成できます。
```sql
SELECT time, member_id, category, sub_category, goods_id
FROM sales_slip
ORDER BY RAND()
LIMIT 5
```
|time|member_id|category|sub_category|goods_id|
|----|---------|--------|------------|--------|
|1354147836|391139   |Movies and Music and Games|Entertainment Collectibles|602270  |
|1319643101|         |Automotive and Industrial|Lab and Scientific|109601  |
|1354963866|1638055  |Beauty and Health and Grocery|Specialty Diets|606541  |
|1265313812|282436   |Toys and Kids and Baby|For Baby    |382080  |
|1123604291|111187   |Electronics and Computers|Wearable Technology|122035  |

各レコードにはRAND関数によって一様に0.0<x<=1.0の値が付与されているので，その値で並び替えられているだけです。同じレコードでも，サンプリングを再実行すれば異なるランダムな値が付与され，その順序は前回のサンプリングとは異なるため，毎回違ったサンプルが得られることになります。SELECT節にもRAND関数を書くほうが，このクエリの本来の意味がはっきりするでしょう。

```sql
SELECT time, member_id, category, sub_category, goods_id, RAND() AS rnd
FROM sales_slip
ORDER BY rnd
LIMIT 5
```
|time|member_id|category|sub_category|goods_id|rnd                   |
|----|---------|--------|------------|--------|----------------------|
|1367296687|         |Automotive and Industrial|Lab and Scientific|109601  |2.1735940403111442e-07|
|1202236057|717558   |Electronics and Computers|Cell Phones and Accessories|249775  |2.659721006770255e-07 |
|1293285082|949366   |Beauty and Health and Grocery|Menâ€™s Grooming|109090  |7.179268182166965e-07 |
|1387234627|517345   |Electronics and Computers|Trade In Your Electronics|665739  |8.370794056800079e-07 |
|1334767187|         |Automotive and Industrial|Lab and Scientific|109601  |8.393554470353948e-07 |


## ROW_NUMBERとRAND関数によるランダムサンプリング
Window関数を使って各レコードにランダムな（かつユニークな）行番号を割り当てる方法も考えられます。
```sql
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, ROW_NUMBER()OVER(ORDER BY RAND()) AS rnd_rnk
  FROM sales_slip
)
WHERE rnd_rnk <= 5
```
|time|member_id|category|sub_category|goods_id|rnd_rnk               |
|----|---------|--------|------------|--------|----------------------|
|1319312519|         |Automotive and Industrial|Lab and Scientific|109601  |1                     |
|1310589185|1132047  |Sports and Outdoors|Sports Collectibles|512941  |2                     |
|1195516803|949366   |Beauty and Health and Grocery|Menâ€™s Grooming|109090  |3                     |
|1349567121|         |Automotive and Industrial|Lab and Scientific|109601  |4                     |
|1382551570|2259091  |Automotive and Industrial|Lab and Scientific|        |5                     |


このクエリにはLIMIT節が入らないので，UNION ALLなどでクエリを続けることができます。下記のクエリ例はサンプリングを並行して2回繰り返すものです。
```sql
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, ROW_NUMBER()OVER(ORDER BY RAND()) AS rnd_rnk
  FROM sales_slip
)
WHERE rnd_rnk <= 5
UNION ALL
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, ROW_NUMBER()OVER(ORDER BY RAND()) AS rnd_rnk
  FROM sales_slip
)
WHERE rnd_rnk <= 5
```
|time|member_id|category|sub_category|goods_id|rnd_rnk               |
|----|---------|--------|------------|--------|----------------------|
|1208484603|695856   |Beauty and Health and Grocery|All Beauty  |260052  |1                     |
|1318863299|1952695  |Electronics and Computers|Software    |533854  |2                     |
|1258663183|574971   |Home and Garden and Tools|Appliances  |364286  |3                     |
|1354367776|         |Automotive and Industrial|Lab and Scientific|109601  |4                     |
|1190403880|1104914  |Movies and Music and Games|Digital Games|233695  |5                     |
|1149882259|558829   |Toys and Kids and Baby|For Girls   |164186  |1                     |
|1352590636|         |Automotive and Industrial|Lab and Scientific|109601  |2                     |
|1325342386|         |Automotive and Industrial|Lab and Scientific|109601  |3                     |
|1250183164|554562   |Beauty and Health and Grocery|Health and Household and Baby Care|344561  |4                     |
|1300399724|90886    |Home and Garden and Tools|Arts and Crafts and Sewing|488745  |5                     |


## 層別サンプリング
Window関数を用いたランダムサンプリングの最大の効用は，グループ（層）ごとのレコード数の偏りを意識した層別サンプリングが可能なことです。

### グループごとにサンプリング

まず，テーブル全体ではなく各グループごとに単純にn件ずつランダムサンプリングする方法を考えます。
```sql
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, ROW_NUMBER()OVER(PARTITION BY category ORDER BY RAND()) AS rnd_rnk
  FROM sales_slip
)
WHERE rnd_rnk <= 5
ORDER BY category
```
|time|member_id|category|sub_category|goods_id|rnd_rnk               |
|----|---------|--------|------------|--------|----------------------|
|1169892840|940685   |Automotive and Industrial|Industrial Supplies|201438  |1                     |
|1361026836|         |Automotive and Industrial|Lab and Scientific|109601  |2                     |
|1330942517|         |Automotive and Industrial|Lab and Scientific|109601  |3                     |


この方法は，カテゴリごとにランダムサンプリングした結果を使いたい場合には有用です。しかし，カテゴリから満遍なく取得したいという目的がある場合には，それらをひとまとまりのサンプルとして扱うには難点があります。なぜなら，カテゴリごとに母集団の数が異なるからです。例えば，全部で1万レコードあるカテゴリAと100レコードしかないカテゴリBから等しく10件ずつランダムサンプリングしてくると，カテゴリBの割合が圧倒的に高いサンプルとなり，元のテーブルとは分布が異なったサンプルとなってしまいます。

### グループごとの母集団の数を意識したサンプリング

そこで，事前に category ごとの全件数を取得しておき（sample_stat），category ごとにランダムに割り振ったランクで，各件数の1/10000のみを取得するクエリを考えてみます。

```sql
WITH stat AS
(
  SELECT category, COUNT(1) AS cnt_category
  FROM sales_slip
  GROUP BY category
)
SELECT s.*
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, ROW_NUMBER()OVER(PARTITION BY category ORDER BY RAND()) AS rnd_rnk
  FROM sales_slip
) s, stat
WHERE s.category = stat.category
AND rnd_rnk <= cnt_category/10000
```
|time|member_id|category|sub_category|goods_id|rnd_rnk               |
|----|---------|--------|------------|--------|----------------------|
|1349527085|589567   |Books and Audible|Books       |595208  |1                     |
|1375748929|1675318  |Books and Audible|Textbooks   |647005  |2                     |
|1242335031|1251374  |Books and Audible|Textbooks   |325431  |3                     |


各cateogryごとの取得比率がどうなっているかを以下のクエリで確認してみましょう。

```sql
WITH stat AS
(
  SELECT category, COUNT(1) AS cnt
  FROM sales_slip
  GROUP BY category
)
SELECT s.category, MIN(stat.cnt) AS cnt_category, COUNT(1) AS cnt, 1.0*COUNT(1)/MIN(stat.cnt) AS ratio
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, ROW_NUMBER()OVER(PARTITION BY category ORDER BY RAND()) AS rnd_rnk
  FROM sales_slip
) s, stat
WHERE s.category = stat.category
AND rnd_rnk <= stat.cnt/10000
GROUP BY s.category
ORDER BY cnt DESC
```
|category|cnt_category|cnt    |ratio|
|--------|------------|-------|-----|
|Automotive and Industrial|2207193     |220    |9.967411096356322e-05|
|Beauty and Health and Grocery|894252      |89     |9.952451881572532e-05|
|Electronics and Computers|662959      |66     |9.95536677230417e-05|
|Home and Garden and Tools|589352      |58     |9.841317243345233e-05|
|Sports and Outdoors|512399      |51     |9.953181017137036e-05|
|Movies and Music and Games|323719      |32     |9.885116412691253e-05|
|Toys and Kids and Baby|293097      |29     |9.894335322435918e-05|
|Clothing and Shoes and Jewelry|222291      |22     |9.89693689803006e-05|
|Books and Audible|187086      |18     |9.621243706103075e-05|


ratioの値は「9.9...e-05=9.9...*10-5=0.000099...」を意味します。概ね0.01%取得できているのが確認できました。

### グループごとの母集団の数を意識したサンプリング（PERCENT_RANK）

本来ならば，レコード数が100倍異なるカテゴリAとBのランダムサンプリングの結果には，それぞれのレコードが100:1に近い割合で登場してほしいところです。そのためには，カテゴリごとの取得件数を，各カテゴリの母集団の数に応じて可変にできる必要があります。先ほどはROW_NUMBERにてそれを実現しましたが，より扱いやすいPERCENT_RANKを紹介します。

```sql
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, PERCENT_RANK()OVER(PARTITION BY category ORDER BY RAND()) AS per_rnk
  FROM sales_slip
)
WHERE per_rnk <= 0.0001 -- 0.01%
ORDER BY category
```
|time|member_id|category|sub_category|goods_id|per_rnk              |
|----|---------|--------|------------|--------|---------------------|
|1317696525|         |Automotive and Industrial|Lab and Scientific|109601  |9.967415612234912e-05|
|1376996118|193551   |Automotive and Industrial|Industrial Supplies|583266  |9.922109177633845e-05|
|1384775934|2259091  |Automotive and Industrial|Lab and Scientific|        |9.876802743032777e-05|


上記のクエリでは，各カテゴリごとにランダムに並び替えたうえで，（上位）0.01%を取得しています。カテゴリごとに母集団の数が異なりますが，それぞれの中での0.01%を取得しているので，サンプル全体としてのカテゴリの専有割合は元のテーブルと同じになります。これを確認するためのクエリは以下になります。

```sql
WITH stat AS
(
  SELECT category, COUNT(1) AS cnt
  FROM sales_slip
  GROUP BY category
)

SELECT s.category, stat.cnt AS cnt_category, s.cnt, 1.0*s.cnt/stat.cnt AS ratio
FROM
(
  SELECT category, COUNT(1) AS cnt
  FROM
  (
    SELECT time, member_id, category, sub_category, goods_id, PERCENT_RANK()OVER(PARTITION BY category ORDER BY RAND()) AS per_rnk
    FROM sales_slip
  )
  WHERE per_rnk <= 0.0001 -- 0.01%
  GROUP BY category
) s, stat
WHERE s.category = stat.category
ORDER BY category
```
|category|cnt_category|cnt    |ratio|
|--------|------------|-------|-----|
|Automotive and Industrial|2207193     |221    |0.00010012717510430669|
|Beauty and Health and Grocery|894252      |90     |0.00010064277183612673|
|Books and Audible|187086      |19     |0.00010155757245331024|
|Clothing and Shoes and Jewelry|222291      |23     |0.00010346797666122335|
|Electronics and Computers|662959      |67     |0.00010106205662793627|
|Home and Garden and Tools|589352      |59     |0.00010010995126851185|
|Movies and Music and Games|323719      |33     |0.00010194026300587855|
|Sports and Outdoors|512399      |52     |0.00010148341429237762|
|Toys and Kids and Baby|293097      |30     |0.00010235519299071639|


ただし，このクエリでは固定数のサンプルを得ることはできないので注意してください。

### グループごとの母集団の数を意識したサンプリング（NTILE）
先程のPERCENT_RANKはNTILEによっても実現可能です。NTILEの引数には，（各パーティションにおいて）分割するタイル数を指定するので，0.01%のサンプルが欲しければ10000分割したうちの1つのタイルを選択すればよいことになります。異なるタイル番号を選択することで他のサンプルも同時に取れることから，PERCENT_RANKより使い勝手が良いかもしれません。

```sql
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, NTILE(10000)OVER(PARTITION BY category ORDER BY RAND()) AS tile
  FROM sales_slip
)
WHERE tile <= 1 -- 0.01%
ORDER BY category
```
|time|member_id|category|sub_category|goods_id|tile|
|----|---------|--------|------------|--------|----|
|1383494533|2259091  |Automotive and Industrial|Lab and Scientific|        |1   |
|1317995754|         |Automotive and Industrial|Lab and Scientific|109601  |1   |
|1254148710|1492081  |Automotive and Industrial|Tires and Wheels|353152  |1   |


先程と同様に，各カテゴリの母集団の数を考慮したサンプル配分になっていることを確認してみましょう。

```sql
WITH stat AS
(
  SELECT category, COUNT(1) AS cnt
  FROM sales_slip
  GROUP BY category
)

SELECT s.category, stat.cnt AS cnt_category, s.cnt, 1.0*s.cnt/stat.cnt AS ratio
FROM
(
  SELECT category, COUNT(1) AS cnt
  FROM
  (
    SELECT time, member_id, category, sub_category, goods_id, NTILE(10000)OVER(PARTITION BY category ORDER BY RAND()) AS tile
    FROM sales_slip
  )
  WHERE tile <= 1 -- 0.01%
  GROUP BY category
) s, stat
WHERE s.category = stat.category
ORDER BY category
```
|category|cnt_category|cnt    |ratio|
|--------|------------|-------|-----|
|Automotive and Industrial|2207193     |221    |0.00010012717510430669|
|Beauty and Health and Grocery|894252      |90     |0.00010064277183612673|
|Books and Audible|187086      |19     |0.00010155757245331024|
|Clothing and Shoes and Jewelry|222291      |23     |0.00010346797666122335|
|Electronics and Computers|662959      |67     |0.00010106205662793627|
|Home and Garden and Tools|589352      |59     |0.00010010995126851185|
|Movies and Music and Games|323719      |33     |0.00010194026300587855|
|Sports and Outdoors|512399      |52     |0.00010148341429237762|
|Toys and Kids and Baby|293097      |30     |0.00010235519299071639|


もちろん，カテゴリの階層レベルではなく，グッズの階層にまで層を分けるということも考えられます。しかし，そこまで細かい層を考慮するようになると，今度は別の問題が発生することになります。実際，以下のクエリを実行すると，結果のレコード数が先程よりだいぶ多くなっていることに気づきます。

```sql
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, NTILE(10000)OVER(PARTITION BY category,sub_category,goods_id ORDER BY RAND()) AS tile
  FROM sales_slip
)
WHERE tile <= 1 -- 0.01%
```
```sql
386011 records
```

この原因を知るために，グッズごとの母集団の数と，サンプリングされた数を調べてみましょう。以下のクエリにより，母集団の数が少ないグッズ順に並び替えます。

```sql
WITH stat AS
(
  SELECT category, sub_category, goods_id, COUNT(1) AS cnt
  FROM sales_slip
  GROUP BY category, sub_category, goods_id
)

SELECT s.category, s.sub_category, s.goods_id, stat.cnt AS cnt_all, s.cnt, 1.0*s.cnt/stat.cnt AS ratio
FROM
(
  SELECT category, sub_category, goods_id, COUNT(1) AS cnt
  FROM
  (
    SELECT time, member_id, category, sub_category, goods_id, NTILE(10000)OVER(PARTITION BY category ORDER BY RAND()) AS tile
    FROM sales_slip
  )
  WHERE tile <= 1 -- 0.01%
  GROUP BY category, sub_category, goods_id
) s, stat
WHERE s.category = stat.category AND s.sub_category = stat.sub_category AND s.goods_id = stat.goods_id
ORDER BY cnt_all ASC
```
|category|sub_category|goods_id|cnt_all|cnt|ratio|
|--------|------------|--------|-------|---|-----|
|Automotive and Industrial|Motorcycle and Powersports|282371  |1      |1  |1.0  |
|Sports and Outdoors|Hunting and Fishing|265622  |1      |1  |1.0  |
|Electronics and Computers|Car Electronics and GPS|652662  |1      |1  |1.0  |
…
|category|sub_category|goods_id|cnt_all|cnt|ratio|
|--------|------------|--------|-------|---|-----|
|Automotive and Industrial|Industrial Supplies|583266  |23307  |3  |0.00012871669455528383|
|Beauty and Health and Grocery|Menâ€™s Grooming|109090  |547616 |48 |8.765266171916088e-05|
|Automotive and Industrial|Lab and Scientific|109601  |1525984|155|0.0001015738041814331|


上記の結果から問題点がわかります。母集団の数が1のグッズでは，サンプル数も必ず1になっています。つまり，0.01%ではなく100%の割合でサンプリングされているということです。これでは層別に偏りのあるサンプリングを行っていることになり，うまくいっていないことを意味します。

このようにグッズ単位まで層を分けると，母集団の数が少ないグッズが多々存在することになります。NTILEによる層別サンプリングの注意点は，タイルの分割数よりレコード数が少ないWindowのパーティションでは前半のタイルに偏ってしまい，指定した割合でサンプリングが行えなくなることです。

例えば，あるグッズは2件しかレコードが存在しないとします。この場合，10000分割指定のタイルでは1番目と2番目のタイルにレコードがランダムに1件ずつ割り振られます。よって，そこから1番目のタイルだけを指定して1/10000のサンプリングを行おうと思っていても，実際には1/2のサンプリングになっていることになります。逆に，10000件以上のレコードがあるグッズでは10000のタイルにできるだけ均等に分割（ただし，足が出た順にはじめのタイルに入っていく）できるため，ほぼ1/10000のサンプリングが行えます。一般に，10000に満たないm件のグッズでは， 1/mのランダムサンプリングになってしまっています。

結論として，すべてのパーティションのレコード数mが分割数nよりも大きくないと，偏りのない層別サンプリングを行うことはできません。層別サンプリングは万能ではないのです。これを理解するために具体的な例を見てみましょう。

```sql
SELECT val, NTILE(10)OVER(ORDER BY RAND()) AS tile
FROM ( VALUES 1,2,3,4 ) AS t(val)
```
|val|tile|
|---|----|
|4  |1   |
|3  |2   |
|1  |3   |
|2  |4   |


パーティションのレコード数よりも大きな数で分割したNTILEでは，並びがランダムではなく，タイルは1から順に入っていってしまいます。5以降のタイルにはレコードが割り当てられません。

PERCENT_RANKでも類似の問題が起こります。
```sql
SELECT val, PERCENT_RANK()OVER(ORDER BY RAND()) AS per_rnk
FROM ( VALUES 1,2,3,4 ) AS t(val)
```
|val|per_rnk|
|---|-------|
|1  |0.0    |
|2  |0.3333333333333333|
|4  |0.6666666666666666|
|3  |1.0    |


こちらの問題は，どんなにパーティションのレコード数が小さくても，一番はじめは0（0%），1番最後は1.0（100%）の値になってしまうことです。4レコード中の0.01%のサンプリングでも，必ず0%の値になる1レコードが抽出されてしまうということです。
一方，PERCENT_RANKとは異なる順序割合を計算するCUME_DISTでは，一番はじめは0より大きい値になります。

```sql
SELECT val, CUME_DIST()OVER(ORDER BY RAND()) AS cume
FROM ( VALUES 1,2,3,4 ) AS t(val)
```
|val|cume|
|---|----|
|2  |0.25|
|4  |0.5 |
|3  |0.75|
|1  |1.0 |


この場合も万能ではなく，上の例では25%以下のサンプリングではサンプルが「必ず」1つも得られないことになります。これは，25%の確率でサンプルが得られないこととは大きく意味が異なります。

### 時系列データを意識した層別サンプリング

時系列データにおいては，時間単位でもデータ量が変わってくることが多々あります（日単位で考えるとわかりやすいでしょう）。例えば，購買履歴やゲームのアクティビティログでは，セール日やイベント日，月初月末や給料日などでアクセス数が変わってきます。これら特異日のレコードは，層別サンプリングで得られるサンプルにも多く現れてくるはずで，特に時系列チャートを採用する場合などでは考慮が必要になってきます。ただし，カテゴリに加えて日付についてもパーティションを分けたうえで数%のサンプルを取得するということなので，パーティション上の元レコード数が少なくなりすぎないよう，少なくなる場合には取得する割合を大きくしておくことが重要です。
前回までの反省を生かして，慎重にサンプリングを考えていきましょう。まず，「カテゴリごと」と「日付ごと」の組合せのサンプル数を確認します。数の小さい順に並べます。

```sql
SELECT category, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d, COUNT(1) AS cnt
FROM sales_slip
GROUP BY category, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
ORDER BY cnt ASC
```
|category|d  |cnt|
|--------|---|---|
|Beauty and Health and Grocery|2005-01-05|1  |
|Automotive and Industrial|2005-01-02|1  |
|Toys and Kids and Baby|2005-01-05|1  |


残念ながら，1件しかない組合せが多数存在してしまっています。そこで，日単位を諦めて月単位で考えることにします。

```sql
SELECT category, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS m, COUNT(1) AS cnt
FROM sales_slip
GROUP BY category, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
ORDER BY cnt ASC
```
|category|m  |cnt|
|--------|---|---|
|Clothing and Shoes and Jewelry|2004-12-01|22 |
|Automotive and Industrial|2004-12-01|28 |
|Books and Audible|2004-12-01|51 |


最低でも22件のカテゴリと月の組合せがあることがわかりました。上記の結果を見たうえで，ここではサンプリング方針を以下のように定めます。
- 2004年のレコードは無視する
- 1/379 < 0.25%のサンプリングを行う（2005年以降の最小サンプル数379件を考慮）
もし2004年も考慮するならば，1/22<4.5%より大きいサンプル数で行うことになるでしょう。今回はNTILEを使うので，単純に最低レコード数の379を分割数として，そのうちの1つのタイルを抽出する形になります。

```sql
SELECT *
FROM
(
  SELECT time, member_id, category, sub_category, goods_id, 
    NTILE(379)OVER(PARTITION BY category,TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') ORDER BY RAND()) AS tile
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2005-01-01', NULL, 'JST')
)
WHERE tile <= 1 -- 1/379 ≒ 0.25%
```
|time|member_id|category|sub_category    |goods_id|tile|
|----|---------|--------|----------------|--------|----|
|1166384297|949366   |Beauty and Health and Grocery|Menâ€™s Grooming|109090  |1   |
|1166003653|949366   |Beauty and Health and Grocery|Menâ€™s Grooming|109090  |1   |
|1167134792|949366   |Beauty and Health and Grocery|Menâ€™s Grooming|109090  |1   |


いつも通り，パーティションごとの母数とサンプル数を確認してみます。

```sql
WITH stat AS
(
  SELECT category,TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS m, COUNT(1) AS cnt
  FROM sales_slip
  GROUP BY category,TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
)
SELECT s.category, stat.m, stat.cnt AS cnt_all, s.cnt AS cnt_sample, 1.0*s.cnt/stat.cnt AS ratio
FROM
(
  SELECT category,TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS m, COUNT(1) AS cnt
  FROM
  (      SELECT time, member_id, category, sub_category, goods_id, 
      NTILE(379)OVER(PARTITION BY category,TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') ORDER BY RAND()) AS tile
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2005-01-01', NULL, 'JST')
  )
  WHERE tile <= 1 -- 1/379 ≒ 0.25%
  GROUP BY category,TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
) s, stat
WHERE s.category = stat.category AND s.m = stat.m
ORDER BY cnt_all ASC
```
|category|m  |cnt_all|cnt_sample      |ratio |
|--------|---|-------|----------------|------|
|Books and Audible|2005-01-01|379    |1               |0.002638522427440633|
|Books and Audible|2005-02-01|467    |2               |0.004282655246252677|
|Clothing and Shoes and Jewelry|2005-01-01|478    |2               |0.0041841004184100415|
|Movies and Music and Games|2005-01-01|482    |2               |0.004149377593360996|
|Automotive and Industrial|2005-01-01|504    |2               |0.003968253968253968|
|...     |   |       |                |      |
|Automotive and Industrial|2013-07-01|68485  |181             |0.0026429145068263124|
|Automotive and Industrial|2011-11-01|153386 |405             |0.0026403974287092695|
|Automotive and Industrial|2011-10-01|296840 |784             |0.002641153483358038|


小さいところでは，0.40%と2倍ほどのサンプリングになってしまっていますが，大きいところではうまく0.26%のサンプリングとなっています。
