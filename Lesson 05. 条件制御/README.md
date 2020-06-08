# Lesson 05. 条件制御

## IFの基本

IFは集約関数の章でも利用しましたが，改めて説明します。

```sql
IF(condition, true_value, false_value)
```

IFは判定式conditionがTRUEを返す場合には真値（true_value），そうでない（FALSEとNULL）の場合には偽値（false_value）を返します。つまり，IFがNULLを返すことはありません。以下のクエリは，数字とNULLの値に対して奇数か偶数（2で割り切れるか否か）を判定式とし，前者なら「odd」，後者なら「even」を返します。 NULLの場合にも「even」が返されることが重要です。IFで真値および偽値以外の第三の値を返すことはできません。
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6,NULL ) AS t(number) )

SELECT number,
  IF(number%2=1,'odd','even') AS number_type
FROM numbers

上記の結果に満足できない場合は，CASE文によってさらに多くの条件分岐ルートを設けることが可能です。「CASEの基本」で解決法を紹介します。
IFの結果の集計横展開
IF結果を集計して縦方向にテーブル出力
先程の例で奇数および偶数ごとに数を集計するクエリは以下のように簡単に書けます。
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) )

SELECT number_type, COUNT(1) AS cnt
FROM
(
  SELECT number, IF(number%2=1,'odd','even') AS number_type
  FROM numbers
)
GROUP BY number_type

IFの結果を集計して横方向にテーブル出力
上記のクエリの結果では集計結果が行ごとに出力されています。一方，IFとCOUNTを上記のクエリとは違う形で組合せることで，1つの行にカラムとして出力することができます。これは比率を求める際などに何かと役立ちます。注意すべきは，条件を満たさない場合には0やFALSEではなくNULLを指定することです！ そうしないと，それもカウントされてしまいます。
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6,NULL ) AS t(number) )

SELECT
  COUNT(IF(number%2=1,1,NULL)) AS num_odd,
  COUNT(IF(number%2=0,1,NULL)) AS num_even,
  COUNT(IF(number IS NULL,1,NULL)) AS num_null,
  COUNT(1) AS num_total,
  COUNT(IF(number%2=0,1,0)) AS num_even_wrong, --IFの結果にかかわらずカウントされてしまう悪い例
  COUNT(number) AS num_total_wrong --NULLをカウントしない全件カウント
FROM numbers

CASEの基本
CASE expression
    WHEN value THEN result
    [ WHEN ... ]
    [ ELSE result ]
END
CASEによる条件分岐ではIFよりも細かく設定でき，大変便利です。expressionには，カラムそのものか，カラム（複数可）の計算式を指定できます。その値によ応じてそれぞれresultを持つ条件分岐を設定できます。
CASEにおける条件分岐は常に上から順に評価され，どれにも当てはまらないものはELSEで拾えます。
条件にNULLの値がきた場合は，（この書き方では）ELSEの分岐に入ることになります。途中で「WHEN NULL」と書いても捕捉されません。
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6,NULL ) AS t(number) )

SELECT number,
  CASE number%2
    WHEN 1 THEN 'odd'
    WHEN 0 THEN 'even'
    WHEN NULL THEN 'null'
    ELSE 'other'
  END
  AS number_type
FROM numbers

NULLを捕捉するには，WHENの箇所に条件を丁寧に記述する以下の書き方を使います。WHENの1つひとつに条件式（それぞれ異なる条件式でもよい）を記述していく方式です。
CASE
    WHEN condition THEN result
    [ WHEN ... ]
    [ ELSE result ]
END
この書き方であれば，以下のようにしてNULLを捕捉できます。
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6,NULL ) AS t(number) )

SELECT number,
  CASE 
    WHEN number%2=1 THEN 'odd'
    WHEN number%2=0 THEN 'even'
    WHEN number%2 IS NULL THEN 'null'
    ELSE 'other'
  END
  AS number_type
FROM numbers

CASEとセグメント
CASEは集計結果に応じたセグメントを作る際によく使われます。以下のクエリはユーザーごとの年間の購入総額によってセグメントを作る例です。
WITH sales_table AS
(
  SELECT member_id, category, SUM(price*amount) AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, category
)

SELECT member_id, category, sales,
  CASE
    WHEN 0      <= sales AND sales < 10000   THEN 'Low'
    WHEN 10000  <= sales AND sales < 100000  THEN 'Mid'
    WHEN 100000 <= sales AND sales < 1000000 THEN 'High'
    ELSE 'Extreme'
  END AS seg_sales
FROM sales_table

CASEで作ったセグメントで集計する際は，セグメント名で並び替え可能になることを意識したほうが賢明でしょう。
WITH sales_table AS
(
  SELECT member_id, category, SUM(price*amount) AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, category
)

SELECT seg_sales, COUNT(1) AS cnt
FROM
(
  SELECT member_id, category, sales,
    CASE
      WHEN 0      <= sales AND sales < 10000   THEN '01_Low'
      WHEN 10000  <= sales AND sales < 100000  THEN '02_Mid'
      WHEN 100000 <= sales AND sales < 1000000 THEN '03_High'
      ELSE '04_Extreme'
    END AS seg_sales
  FROM sales_table
)
GROUP BY seg_sales
ORDER BY seg_sales

CASEとFM分析
CASEの応用として，sales_slipのユーザーをFrequency（購入頻度）とMonetary（購入総額）でそれぞれセグメント分けした後にクロステーブルとして出力するFM分析の例を紹介します。
WITH sales_table AS
(
  SELECT member_id, category, SUM(price*amount) AS sales, COUNT(1) AS cnt
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, category
)

SELECT seg_monetary,
  COUNT(1) AS freq,
  COUNT(IF(seg_frequency='01_Low',1,NULL))     AS freq_low,
  COUNT(IF(seg_frequency='02_Mid',1,NULL))     AS freq_mid,
  COUNT(IF(seg_frequency='03_High',1,NULL))    AS freq_high,
  COUNT(IF(seg_frequency='04_Extreme',1,NULL)) AS freq_extreme
FROM
(
  SELECT member_id, category, sales,
    CASE
      WHEN 0      <= sales AND sales < 10000   THEN '01_Low'
      WHEN 10000  <= sales AND sales < 100000  THEN '02_Mid'
      WHEN 100000 <= sales AND sales < 1000000 THEN '03_High'
      ELSE '04_Extreme'
    END AS seg_monetary,
    CASE
      WHEN 1  <= cnt AND cnt < 3   THEN '01_Low'
      WHEN 3  <= cnt AND cnt < 10  THEN '02_Mid'
      WHEN 10 <= cnt AND cnt < 100 THEN '03_High'
      ELSE '04_Extreme'
    END AS seg_frequency
  FROM sales_table
)
GROUP BY seg_monetary
ORDER BY seg_monetary

COALESCE
COALESCE関数は，第1引数で指定したカラムの値にNULLがあった場合に，第2引数の値で置き換えてくれるもので，（特にCOUNTやAVGで意図しない集計になるのを防ぐうえで）非常に有用です。
例えば，平均と合計とレコード数を求める以下のクエリで，NULLの場合も0という値として集計したいとします。何も考えずに以下のクエリを実行しても，目的に合致する結果になるのはSUMの場合だけであることがわかります。
SELECT AVG(a) AS ag, COUNT(a) AS cnt, SUM(a) AS sm
FROM ( VALUES 1, 1, 1, 1, NULL, NULL ) AS t(a)

目的を果たすには，COALESCE関数によって，NULLを0に変えてから集計する必要があります。
SELECT AVG(COALESCE(a, 0)) AS ag, COUNT(COALESCE(a, 0)) AS cnt, SUM(COALESCE(a, 0)) AS sm
FROM ( VALUES 1, 1, 1, 1, NULL, NULL ) AS t(a)

