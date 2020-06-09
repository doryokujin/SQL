# Lesson 11. Window関数

## 集計を「集約」ではなく「付与」する世界観

Window関数は，別名OLAP（分析）関数とも呼ばれ，データウェアハウスにおける分析のために使用されてきた，まさに「データ分析」のための関数といえます。HiveでもPrestoでもWindow関数が使えるようになったのはつい最近ですが，Window関数を理解し，使いどころを極めれば，データ分析の幅がこれまでの何倍にも広まると言っても過言ではありません。

## 概要

Window関数は，テーブルを区間ごとに集計，参照，ランキングし，それらを各行に新しいカラム値として付与する機能です。集約関数に似ていますが，Window関数では複数の行がまとめられることはなく，行それぞれに集計された値カラムが追加されて返されます。また，処理中の行以外の行の値を読み取ることも可能です。Window関数は基本的には以下の構文を組合せて使います。

- PARTITION BY col：区間に分割
- ORDER BY col：並び替え
- ROWS BETWEEN...：範囲指定
関数としては，上記に挙げたもののほか，通常の集約関数（COUNTやSUMなど）が使用できます。 

## ランキング系
Window関数の中でも利用頻度が高い，ランキング系の関数を紹介します。

### ROW_NUMBER：行番号
ROW_NUMBER関数は，区間ごとに順序付けされたレコードに対して上から順に行番号を割り振っていきます。複数のレコードが同値であった場合も，登場順に番号が増えていくところが特徴的です。同値を同ランキングとみなしたい場合には，後述のRANKを使います。

```sql
SELECT val, ROW_NUMBER()OVER(ORDER BY val) AS rnk
FROM ( VALUES 1,1,2,3 ) AS t(val)
ORDER BY rnk ASC
```

#### 例：ユーザーごとに，最初のアクセスから順にレコードに番号を割り振る
指定した2ユーザーに対してtime順（最初のアクセス順）に番号を割り振るという，ROW_NUMBERの基本的な使い方を紹介します。

```sql
SELECT rnk, td_client_id, time
FROM
(
  SELECT 
    ROW_NUMBER() OVER (PARTITION BY td_client_id ORDER BY time ASC) AS rnk, td_client_id, time
  FROM sample_accesslog
) tmp
WHERE td_client_id IN ('f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a','7f47d05f-bd12-4553-e69c-763064738631')
ORDER BY td_client_id, rnk ASC
```

#### 例：ユーザーごとの直帰のアクティビティを知るために，各ユーザーの最新の5レコードを取得する
ROW_NUMBERは区間ごとに一意の番号を割り振ることから，「区間ごとに先頭の5件を取得する」といった操作が容易にできます。
```sql
SELECT rnk, td_client_id, time
FROM
(
  SELECT 
    ROW_NUMBER() OVER (PARTITION BY td_client_id ORDER BY time DESC) AS rnk, td_client_id, time
  FROM sample_accesslog
) tmp
WHERE rnk<=5
AND td_client_id IN ('f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a','7f47d05f-bd12-4553-e69c-763064738631')
ORDER BY td_client_id, rnk
```

### RANK：ランキング（同率で番号を飛ばす）
RANK関数は，区間ごとに順序付けされたレコードに対して上から順に行番号を割り振っていきます。複数のレコードが同値であった場合は同じ番号を付与し，その次の値からは番号を飛ばします。例えば1位が3件あった場合，RANKは「1,1,1,4,5,6,...」という順序付けを行います。

```sql
SELECT val, RANK()OVER(ORDER BY val) AS rnk
FROM ( VALUES 1,1,2,3 ) AS t(val)
ORDER BY rnk ASC
```

#### 例：カテゴリごとの2011年度マンスリートップセールス5品目を取得する
ECにおける各カテゴリのマンスリーでの売上品目上位を，12ヶ月分，一度に集計します。
```sql
WITH sales_table AS (
  SELECT category, sub_category, goods_id, SUM(price*amount) AS sales, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS d
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY category, sub_category, goods_id, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
)

SELECT d, rnk, category, sales, sub_category, goods_id
FROM
(
  SELECT 
    d, RANK() OVER (PARTITION BY d, category ORDER BY sales DESC) AS rnk, sales, category, sub_category, goods_id
  FROM sales_table
)
WHERE rnk<=5
ORDER BY category, d, rnk
```

#### DENSE_RANK：ランキング（同率で番号を飛ばさない）
DENSE_RANK関数は，RANK関数と挙動が似ていますが，RANKが複数の同値があった場合には次の値の番号を飛ばすのに対し，DENSE_RANKは番号を飛ばしません。例えば1位が3件あった場合，DENSE_RANKは「1,1,1,2,3,4,...」という順序付けを行います。

```sql
SELECT val, DENSE_RANK()OVER(ORDER BY val) AS rnk
FROM ( VALUES 1,1,2,3 ) AS t(val)
ORDER BY rnk ASC
```

#### 例：カテゴリごとのマンスリートップセールス5品目を取得する
ECにおける各カテゴリのマンスリーでの売上品目上位を集計します。同率で番号を飛ばさないので必ず1〜5までの番号があり，歯抜けはありません。

```sql
WITH sales_table AS
(
  SELECT category, sub_category, goods_id, SUM(price*amount) AS sales, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS d
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY category, sub_category, goods_id, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
)

SELECT d, rnk, category, sales, sub_category, goods_id
FROM
(
  SELECT 
    d, DENSE_RANK() OVER (PARTITION BY d, category ORDER BY sales DESC) AS rnk, sales, category, sub_category, goods_id
  FROM sales_table
)
WHERE rnk<=5
ORDER BY category, d, rnk
```

### PERCENT_RANK：ランキング（割合「(rank - 1) / (全行数 - 1)」で表示）
PERCENT_RANK関数は，番号を割り振るのではなく，「(rank – 1) / (全行数 – 1)」という計算をして順位の割合を出します。

```sql
SELECT val, PERCENT_RANK()OVER(ORDER BY val) AS per_rnk
FROM ( VALUES 1,1,2,3 ) AS t(val)
ORDER BY per_rnk ASC
```

#### 例：カテゴリごとにトップセールス10%に属する品目を取得する
区間ごと（=カテゴリごと）の内訳数が異なるような場合に，各々の区間で「上位10%以内の品目」といった指定をします。区間ごとに得られるレコード数が異なる（品目数が多いカテゴリは上位10%の品目数も多くなる）ところが今までの例とは違うところです。
```sql
WITH sales_table AS
(
  SELECT category, sub_category, goods_id, SUM(price*amount) AS sales, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS d
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY category, sub_category, goods_id, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
)

SELECT d, rnk, category, sales, sub_category, goods_id
FROM
(
  SELECT 
    d, PERCENT_RANK() OVER (PARTITION BY d, category ORDER BY sales DESC) AS rnk, sales, category, sub_category, goods_id
  FROM sales_table
)
WHERE rnk<=0.1
ORDER BY category, d, rnk
```

### CUME_DIST：ランキング（相対位置「(現在の行の位置) / (全行数)」で表示）
CUME_DIST関数は，PERCENT_RANKとよく似ていますが，「(現在の行の位置) / (全行数)」という計算式で相対順位を求めます。

```sql
SELECT val, CUME_DIST()OVER(ORDER BY val) AS cume
FROM ( VALUES 1,1,2,3 ) AS t(val)
ORDER BY cume ASC
```

#### 例：カテゴリごとにトップセールス10%に属する品目を取得する

```sql
WITH sales_table AS
(
  SELECT category, sub_category, goods_id, SUM(price*amount) AS sales, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS d
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY category, sub_category, goods_id, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
)

SELECT d, rnk, category, sales, sub_category, goods_id
FROM
(
  SELECT 
    d, CUME_DIST() OVER (PARTITION BY d, category ORDER BY sales DESC) AS rnk, sales, category, sub_category, goods_id
  FROM sales_table
)
WHERE rnk<=0.1
ORDER BY category, d, rnk
```

### NTILE(N)：ランキング（1..Nに分割）

NTILE関数は，レコードを等しい間隔でタイルに分割します。もし10レコードを2タイルに分ける場合，各タイルには5レコードずつとなり，ORDER BYが指定されていれば先頭5レコードが同じタイルに入り，後半5レコードが同じタイルに入ります。均等に分割できない場合（3レコードを10タイルに分割する場合の3レコード，11レコードを10タイルに分割する場合の余り1レコード）は，番号が若いタイルに優先的に収められることになります。

```sql
SELECT val, NTILE(10)OVER(ORDER BY val) AS tile
FROM ( VALUES 1,1,2,3 ) AS t(val)
ORDER BY tile ASC

SELECT val, NTILE(3)OVER(ORDER BY val) AS tile
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY tile ASC
```

#### 例：カテゴリごとのマンスリートップセールス100品目を取得し「1〜10位，11位〜20位，...，91位〜100位」という10品目ごとのバケットに分割する

```sql
WITH sales_table AS
(
  SELECT category, sub_category, goods_id, SUM(price*amount) AS sales, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS d
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY category, sub_category, goods_id, TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
)

SELECT d, rnk, NTILE(10) OVER (PARTITION BY d, category ORDER BY rnk) AS bucket, category, sales, sub_category, goods_id
FROM
(
  SELECT 
    d, RANK() OVER (PARTITION BY d, category ORDER BY sales DESC) AS rnk, sales, category, sub_category, goods_id
  FROM sales_table
)
WHERE rnk <= 100
ORDER BY category, d, rnk, bucket
```

## 参照系

Window関数のうち参照系の関数を紹介します。

### LAG：n行前の値を返す

LAG関数は，現在の行より1つ以上前にある行の値を返します。引数を指定しないデフォルトのオフセットは0（=現在の行）です。nを指定するとn行前のレコードを取得してきます。

```sql
SELECT val, LAG(val,1)OVER(ORDER BY val) AS val_lag1
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```

#### 例：あるユーザーのアクセス日一覧に対して，前回のアクセス日を付与する

```sql
WITH access_table AS
(
  SELECT td_client_id, TD_TIME_FORMAT(time,'yyyy-MM-dd', 'JST') AS d
  FROM sample_accesslog
  WHERE td_client_id IN ('10e725fc-c17f-43e1-da24-a452ef19d2f5')
  GROUP BY td_client_id, TD_TIME_FORMAT(time,'yyyy-MM-dd', 'JST')
)

SELECT rnk, td_client_id, d, d_lag
FROM
(
  SELECT 
  td_client_id, d,
  LAG(d, 1) OVER (PARTITION BY td_client_id ORDER BY d) AS d_lag,
  ROW_NUMBER()OVER (PARTITION BY td_client_id ORDER BY d) AS rnk
  FROM access_table
)
ORDER BY td_client_id, rnk, d DESC, d_lag DESC
```

上記の結果は，ある1人のユーザーのアクセス日を，初回アクセス日を1として順に番号を割り振りつつ，各レコードに前回のアクセス日の情報を付与して番号順に並べたものです。
初回にアクセスした「2016-04-25」については，前の日付が存在しないためLAGが取得できず，NULLが返っています（LAGの第3引数で，存在しない場合の既定値を設定できます）。
以下のクエリでは，ある特定の日付（2016-06-01）にアクセスしたユーザーを対象に，前回のアクセス日の分布を得ることができます。

```sql
SELECT d, d_lag, COUNT(1) AS cnt
FROM
(
  SELECT 
  td_client_id, d,
  LAG(d, 1) OVER (PARTITION BY td_client_id ORDER BY d) AS d_lag
  FROM 
  (
    SELECT td_client_id, TD_TIME_FORMAT(time,'yyyy-MM-dd', 'JST') AS d
    FROM sample_accesslog
    GROUP BY td_client_id, TD_TIME_FORMAT(time,'yyyy-MM-dd', 'JST')
  ) t1
)
WHERE d = '2016-06-16' AND d_lag IS NOT NULL
GROUP BY d, d_lag
ORDER BY d DESC, d_lag DESC
```

この結果から，このデータでは前日にアクセスしている人が最も多く，3日以内前にアクセスしている人が全体の大きな割合を占めていることを示しています。

### LEAD： n行後の値を返す

LEAD関数は，LAG関数と逆で，現在の行より1つ以上後にある行の値を返します。引数を指定しないデフォルトのオフセットは0（=現在の行）です。nを指定するとn行後のレコードを取得してきます。
```sql
SELECT val, LEAD(val,1)OVER(ORDER BY val) AS val_lead1
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```

#### 例：ページの平均閲覧時間を求める
ユーザーごとに閲覧したURLが時系列に並んだレコードで，1つ後ろの行のtimeを見ることができれば，そのURLが見られていた時間の長さを知ることができます。LEAD関数を利用して各レコードごとにその値を求め，各URL（正規化済み）について平均を取れば，そのページの平均閲覧時間を知ることができます。
まずは各レコードのtd_urlカラムについて，閲覧時間を何も考えずに求めてみます。
```sql
SELECT td_client_id,
  REGEXP_REPLACE(REGEXP_REPLACE(td_url, '\?(.)*',''), 'http(.)*://|/$','') AS td_url, time,
  LEAD(time) OVER (PARTITION BY td_client_id ORDER BY time) - time AS diff
FROM sample_accesslog
ORDER BY td_client_id, time
```
閲覧時間（diff）がNULLとなっているのは，そのユーザーの最終アクセスレコードで次のレコードがないためです。これは排除しておきたいですね。
また，閲覧時間がとても大きくなっているのは，そのレコードに該当する閲覧でユーザーがサイトの回遊を止め，数時間や数日後に再びそのURLにアクセスしたためです。このようなケースは意味がある閲覧時間を表していないため除外する必要があります。ここでは，セッションを30分として，閲覧時間が30分以上のレコードを排除することにします。
```sql
SELECT td_client_id, td_url, time, diff
FROM
(
  SELECT td_client_id,
    REGEXP_REPLACE(REGEXP_REPLACE(td_url, '\?(.)*',''), 'http(.)*://|/$','') AS td_url, time,
    LEAD(time) OVER (PARTITION BY td_client_id ORDER BY time) - time AS diff
  FROM sample_accesslog
)
WHERE diff <= 60*30 AND diff IS NOT NULL
ORDER BY td_client_id, time
```
ここで，WHERE節（およびHAVING節）では集計関数のように直接SELECT節のWindow関数を載せることができません。一度サブクエリで包んであげて，外側からWHERE節を書いてあげる必要がある点に注意してください。
上記の結果に対し，URLごとに平均を取ってあげれば，平均閲覧時間が求められます。その上位10件を見てみましょう。ただし，10件以上のレコード数がないURLは，サンプル数が少ないとして集計から除外とします。
```sql
SELECT td_url, AVG(diff) AS avg_diff, COUNT(1) AS cnt
FROM
(
  SELECT td_client_id,
    REGEXP_REPLACE(REGEXP_REPLACE(td_url, '\?(.)*',''), 'http(.)*://|/$','') AS td_url, time,
    LEAD(time) OVER (PARTITION BY td_client_id ORDER BY time) - time AS diff
  FROM sample_accesslog
)
WHERE diff <= 60*30 AND diff IS NOT NULL
GROUP BY td_url
HAVING 10 <= COUNT(1)
ORDER BY avg_diff DESC
```

### FIRST_VALUE：最初の行の値を返す
FIRST_VALUE関数は，並び替えられた各区間の最初の行の値を返します。

```sql
SELECT val, FIRST_VALUE(val)OVER(ORDER BY val) AS val_first
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```

#### 例：マンスリーの各グッズの売上について，そのカテゴリでトップの売上額に対する割合を求める（各カテゴリ上位5件まで）

```sql
WITH sales_table AS
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, category, sub_category, goods_id, SUM(price*amount) AS sales
  FROM  sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-01','JST') , category, sub_category, goods_id
)

SELECT m, category, rnk, goods_id, sales, best_goods_id, best_sales, 1.0*sales/best_sales AS ratio
FROM
(
  SELECT m, category, goods_id, sales, FIRST_VALUE(sales) OVER (PARTITION BY m,category ORDER BY sales DESC) as best_sales,
    FIRST_VALUE(goods_id) OVER (PARTITION BY m,category ORDER BY sales DESC) as best_goods_id,
    RANK() OVER (PARTITION BY m,category ORDER BY sales DESC) as rnk
  FROM sales_table
)
WHERE rnk <= 5
ORDER BY m, category, ratio DESC
```

### LAST_VALUE：最後の行の値を返す
LAST_VALUE関数は，並び替えられた各区間の最後の行の値を返します。
多くのWindow関数では範囲が省略されており，デフォルトの範囲として「ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW」（区間の先頭から現在の行（自己））が使われますが，LAST_VALUEはこのデフォルトの範囲では失敗するWindow関数です。
```sql
SELECT val, LAST_VALUE(val)OVER(ORDER BY val) AS val_last
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```
上記のクエリでは，並び替えられた区間の先頭から現在の行（自己の値）のレコードしか見ません。この範囲における最後の値は常に「自己の値」となってしまいます。（後述するNTH_VALUEでも同様です。）
そこで，以下の例のように区間の範囲を指定してこれを回避します。区間内のすべてのレコードを範囲とすれば，その区間内の最後の値が取得できます。
```sql
SELECT val, LAST_VALUE(val)OVER(ORDER BY val ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS val_last
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```
サンプルデータでもこの挙動の違いを確認してみましょう。ここでは，FIRST_VALUE関数の例と同じような結果を得ることを目的とします。まずは間違った例を紹介します。FIRST_VALUEの例では単純に降順に並び替えて先頭を取ったので，同じように考えて，昇順に並び替えて最後を取る以下のようなクエリを思い浮かべるかもしれません。
```sql
WITH sales_table AS
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, category, sub_category, goods_id, SUM(price*amount) AS sales
  FROM  sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-01','JST') , category, sub_category, goods_id
)

SELECT m, category, rnk, goods_id, sales, best_goods_id, best_sales, 1.0*sales/best_sales AS ratio
FROM
(
  SELECT m, category, goods_id, sales, LAST_VALUE(sales) OVER (PARTITION BY m,category ORDER BY sales) as best_sales,
    LAST_VALUE(goods_id) OVER (PARTITION BY m,category ORDER BY sales) as best_goods_id,
    RANK() OVER (PARTITION BY m,category ORDER BY sales DESC) as rnk
  FROM sales_table
)
WHERE rnk <= 5
ORDER BY m, category, ratio DESC
```
しかし，残念ながら下記のように間違った（常に自身のsalesとbest_salesが同じ値）結果が得られてしまいます。

正解のクエリは次のように書きます。ROWSによる範囲指定を追加していることに注目してください。
```sql
WITH sales_table AS
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, category, sub_category, goods_id, SUM(price*amount) AS sales
  FROM  sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-01','JST') , category, sub_category, goods_id
)

SELECT m, category, rnk, goods_id, sales, best_goods_id, best_sales, 1.0*sales/best_sales AS ratio
FROM
(
  SELECT m, category, goods_id, sales, 
    LAST_VALUE(sales) OVER (PARTITION BY m,category ORDER BY sales ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as best_sales,
    LAST_VALUE(goods_id) OVER (PARTITION BY m,category ORDER BY sales ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as best_goods_id,
    RANK() OVER (PARTITION BY m,category ORDER BY sales DESC) as rnk
  FROM sales_table
)
WHERE rnk <= 5
ORDER BY m, category, ratio DESC
```

特に参照系のWindow関数で注意したいこととして，デフォルトでは，各レコードをPARTITIONED BYで区切りORDER BYで並び替えた区間において，すべての範囲のレコードを見られるような設定にはなっていません。明示的にROWSを記述しない限り，以下の範囲しか見られません。
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW：区間のはじめから現在の行（自己）まで
```
つまり，昇順に並び替えられた区間におけるLAST_VALUEはCURRENT ROW（自己）までの最後であり，これは区間全体の最後とは異なります。
自己ではなく，全体としてのLAST_VALUEを得るためには，以下の範囲としてROWSを明示する必要があります。
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```
これは，自己レコードに対して，「前すべて」から「先すべて」までの範囲を指定します。
なお，降順に並び替えられた区間におけるFIRST_VALUEは，昇順に並び替えられたLAST_VALUEになるだけでなく，必ず区間全体の最後を取るように動作することから，先頭や最後の値を取る際には常にFIRST_VALUEを用いるのが賢明です。

### NTH_VALUE：n番目の行の値を返す

NTH_VALUE関数は，区間のn番目の行の値を返します。LAST_VALUEと同様の制約があるので，有用な結果を得るためにはデータの範囲を指定する必要があります。
```sql
SELECT val, NTH_VALUE(val,3)OVER(ORDER BY val ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS val_nth
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```

#### 例：マンスリーの各グッズの売上に関して，そのカテゴリでトップの売上額に対する割合を求める（各カテゴリ上位5件まで）
NTH_VALUEの引数（何番目の行の値を返すか）を1に指定すれば，FIRST_VALUEを代用できます。
```sql
WITH sales_table AS
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, category, sub_category, goods_id, SUM(price*amount) AS sales
  FROM  sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-01','JST') , category, sub_category, goods_id
)

SELECT m, category, rnk, goods_id, sales, best_goods_id, best_sales, 1.0*sales/best_sales AS ratio
FROM
(
  SELECT m, category, goods_id, sales, 
    NTH_VALUE(sales,1) OVER (PARTITION BY m,category ORDER BY sales DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as best_sales,
    NTH_VALUE(goods_id,1) OVER (PARTITION BY m,category ORDER BY sales DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as best_goods_id,
    RANK() OVER (PARTITION BY m,category ORDER BY sales DESC) as rnk
  FROM sales_table
)
WHERE rnk <= 5
ORDER BY m, category, ratio DESC
```
結果は省略します。
ちなみに，行の最後の値が何番目かは一般にわからないので，NTH_VALUE でLAST_VALUEの代用はできません。

## 集約関数系

Window関数の枠組みの中でも集約関数を利用できます。ただし注意点として，GROUP BYによってレコードが「集約」されるのではなく，各々のレコードに集約された値が「付与される」ことが一般の集約関数とは大きく異なります。
集約関数系では，前と先のいくつまでを集約の範囲とするかを指定する以下のような書き方をよくします。
- 集約関数(...) OVER (PARTITION BY ... ORDER BY ... ROWS ... BETWEEN ... )

「ROWS...BETWEEN」句の書き方のパターンは以下の説明を参照してください。
### ROWS BETWEENパターン
- ROWS BETWEEN m PRECEDING AND n FOLLOWING
：自己レコードに対して「m個前」から「n個先」までの範囲を指定します。
- ROWS BETWEEN UNBOUNDED PRECEDING AND n FOLLOWING
：自己レコードに対して「前すべて」から「n個先」までの範囲を指定します。
- ROWS BETWEEN m PRECEDING AND UNBOUNDED FOLLOWING
：自己レコードに対して「m個前」から「先すべて」までの範囲を指定します。
- ROWS BETWEEN m PRECEDING AND CURRENT ROW
：自己レコードに対して「m個前」から「自己」までの範囲を指定します。
- ROWS BETWEEN CURRENT ROW AND n FOLLOWING
：自己レコードに対して「自己」から「n個先」までの範囲を指定します。
- ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
：自己レコードに対して「前すべて」から「自己」までの範囲を指定します。これがデフォルトの範囲です。
- ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
：自己レコードに対して「自己」から「先すべて」までの範囲を指定します。
- ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
：自己レコードに対して「前すべて」から「先すべて」までの範囲を指定します。

### ROWSとRANGEの違い
ここで重要な注意点を述べます。多くのドキュメントなどでは，ROWSではなくRANGEが使われている場合があります。（例：RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW）
一見，すべての範囲でRANGEはROWSを代替できそうですが，実はRANGEのほうが使える構文が限らます。RANGEが使えるのは「前後がUNBOUNDEDかCURRENT ROWの場合」のみに限定されます。つまり，RANGEがROWに書き換えられるのは，以下のパターンの場合のみです
- RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW（デフォルト）
- RANGE BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
- RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
多くのドキュメントでは，これらRANGEで書き換えられる範囲が多いことから，意味としてわかりやすいRANGEを何気なく使っているものと考えられます。本ドキュメントでもRANGEを用いたクエリを使う場面が存在するかもしれませんが，万能なROWSで書くことを覚えておいたほうが賢明でしょう。

### RUNNING SUM：はじめから現在までの累計和を求める ※この名前の関数はありません

Window関数で集約関数が用いられることが多いのは，時系列データを扱う場合です。RUNNING SUM（累計和）やMOVING AVERAGE（移動平均）がその代表例です。ただし，これらの名前が付いた関数は存在しないことに注意してください。すべての集約関数はOVER節を付けることによってWindow関数にできます。以下に最も単純な例を示します。
```sql
SELECT val, SUM(val)OVER(ORDER BY val ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS run_sum --RANGEは省略可能
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```
注意点として，同順のレコードが生じた場合には重複分だけ和が取られます。つまり，同順のレコードが並ぶ場合，最後の累積和の値が平等に付与されます。これは累積和の特性ではなく，「ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW」と記述することによるものです。
```sql
SELECT val, SUM(val)OVER(ORDER BY val ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS run_sum --RANGEは省略可能
FROM ( VALUES 1,2,2,2,5 ) AS t(val)
ORDER BY val ASC
```
ROWSの書き方を「自身より3つ前まで」となるようにすると，これはMOVING SUM（移動和）となります。
```sql
SELECT val, SUM(val)OVER(ORDER BY val ASC ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) AS moving_avg
FROM ( VALUES 1,2,2,2,5 ) AS t(val)
ORDER BY val ASC
```
この場合の同順のレコードの扱いは先程とは異なります。今度はROWS BETWEEN 3 PRECEDINGの制約が強く働き，同順でもレコードの並びによって値が異なってきます。

#### 例：categoryごとに，各日の売上額と併せて月初から当日までの累計和を表示する
この場合，ROWS指定はデフォルト値でよいので省略できます。
```sql
WITH sales_table AS
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-dd','JST') AS d, TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m_from, category, SUM(price*amount) AS sales
  FROM  sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-dd','JST'), TD_TIME_FORMAT(time,'yyyy-MM-01','JST'), category
)

SELECT d, m_from, category, sales, 
  SUM(sales) OVER (PARTITION BY category,m_from ORDER BY d ASC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as sales_running_sum
FROM sales_table
ORDER BY category, d, m_from
```
日ごとに月初からの売上が積み上がっており，かつ月が変わるとリセットされていることがわかります。

### MOVING AVERAGE：移動平均，nレコードの平均値を求める

移動平均もWindow関数と集約関数で実現可能です。「移動」平均という場合は，一般に先頭からというより，自身よりn件前までを範囲とした平均を指します。
```sql
SELECT val, AVG(val)OVER(ORDER BY val ASC ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) as moving_avg
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC

ROWS BETWEEN 3 PRECEDINGの制約によって，同順でもレコードの並びが違えば異なる値になります。
SELECT val, AVG(val)OVER(ORDER BY val ASC ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) as moving_avg
FROM ( VALUES 1,2,2,2,5 ) AS t(val)
ORDER BY val ASC
```

#### 例：cateogryごとに，各日の売上額と併せて直近5日間の移動平均を表示する
```sql
SELECT d, category, sales, AVG(sales) OVER (PARTITION BY category ORDER BY d ASC ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) as sales_moving_avg
FROM
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-dd','JST') AS d, category, SUM(price*amount) AS sales
  FROM  sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-dd','JST'), category
) t
ORDER BY category, d
```
もし過去4日分のレコードがない場合は，存在する過去分から現在値での平均が求められます。

### LOCAL MAX，MIN：特定の区間での最大，最小を求める
各々のレコードに，該当するパーティション内における局所的な最大値と最小値を付与できます。
```sql
SELECT val, MAX(val)OVER(ORDER BY val ASC) AS local_max
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```

現在の行までの最大を取ってくるので，LAST_VALUEと同様に意図しない結果になる可能性があります。これを避けるには範囲を指定して以下のように書きます。
```sql
SELECT val, MAX(val)OVER(ORDER BY val ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS val_max
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```
ORDER BYを指定しなければ，範囲を意識しなくてもパーティション内の最大を求めることができます。
```sql
SELECT val, MAX(val)OVER() AS val_max
FROM ( VALUES 1,2,3,4,5 ) AS t(val)
ORDER BY val ASC
```
#### 例： category ごとに，月間の最大売上額/最小売上額を各々のレコードに付与する
```sql
WITH sales_table AS
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-dd','JST') AS d, TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, category, SUM(price*amount) AS sales
  FROM  sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-dd','JST'), TD_TIME_FORMAT(time,'yyyy-MM-01','JST'), category
)

SELECT m, d, category, sales, 
  MAX(sales) OVER (PARTITION BY category, m) as max_sales,
  FIRST_VALUE(sales)OVER (PARTITION BY category, m ORDER BY sales DESC) as max_sales2,
  LAST_VALUE(sales)OVER (PARTITION BY category, m ORDER BY sales ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as max_sales3,
  MIN(sales) OVER (PARTITION BY category, m) as min_sales,
  FIRST_VALUE(sales)OVER (PARTITION BY category, m ORDER BY sales ASC) as min_sales2,
  LAST_VALUE(sales)OVER (PARTITION BY category, m ORDER BY sales DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as min_sales3
FROM sales_table
ORDER BY category, m, d, sales DESC
```
1月のデイリーの売上の最大は1月29日，最小は1月3日となっています。
