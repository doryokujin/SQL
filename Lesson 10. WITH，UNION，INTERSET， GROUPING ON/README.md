# Lesson 10. WITH，UNION，INTERSET， GROUPING ON

## WITH節

WITH節は，既に何度か登場しているように，一連のクエリの中で（保存されたテーブルではなく）テンポラリに作成したデータを参照する際に使用します。
下記のクエリでは，VALUESでサンプルデータを作成し，それをWITH節で参照しています。

```sql
WITH numbers AS
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) )

SELECT number FROM numbers
```
|number                                     |
|-------------------------------------------|
|1                                          |
|2                                          |
|3                                          |
|4                                          |
|5                                          |
|6                                          |


カンマで区切ることにより，複数のテンポラリテーブルを参照できます。

```sql
WITH numbers AS
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) ),
alphabets AS
( SELECT number,alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c'),(4,'d'),(5,'e'),(6,'f') ) AS t(number,alphabet) )

SELECT numbers.number, alphabet 
FROM numbers, alphabets
WHERE numbers.number = alphabets.number
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|1                                          |a       |
|4                                          |d       |
|2                                          |b       |
|3                                          |c       |
|5                                          |e       |
|6                                          |f       |


WITH節は，その可読性のよさから，既存のテーブルからSUMやAVG，MAX，MINなどの統計値を取り出して保存しておき，再度同じテーブルからレコードごとにその統計値を一緒に参照するという形でも利用されます。

```sql
WITH numbers AS
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) ),
stats AS
( SELECT MAX(number) AS max_num FROM numbers )

SELECT number, 1.0*number/max_num AS ratio_from_max
FROM numbers, stats
```
|number                                     |ratio_from_max|
|-------------------------------------------|--------------|
|1                                          |0.16666666666666666|
|2                                          |0.3333333333333333|
|3                                          |0.5           |
|4                                          |0.6666666666666666|
|5                                          |0.8333333333333334|
|6                                          |1.0           |


この例では，「FROM numbers, stats」の箇所でCROSS JOINを行っています。ただし，statsのレコードが1行しかないので，numbersの各レコードにstatsのレコードが付与された形になっています。

実例でも試してみましょう。下記のクエリは，月次での売上を計算しつつ，その売上値を過去の最大および最小値と常に比べられるように，WITH節で「最大および最小値とそれらの値となる月」を保持しています。

```sql
WITH stats AS (
  SELECT MAX_BY(m,sales) AS max_m, MAX(sales) AS max_sales, 
    MIN_BY(m,sales) AS min_m, MIN(sales) AS min_sales
  FROM
  (
    SELECT 
      TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, SUM(price*amount) AS sales
    FROM  sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-01','JST')
  )
)

SELECT 
  TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, 
  SUM(price*amount) AS sales, 
  1.0*SUM(price*amount)/stats.max_sales AS ratio_from_max
FROM  sales_slip, stats
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-01','JST'), stats.max_sales
ORDER BY m
```
|m                                          |sales|ratio_from_max    |
|-------------------------------------------|-----|------------------|
|2011-01-01                                 |139568438|0.6463141781486311|
|2011-02-01                                 |119880374|0.5551425989159564|
|2011-03-01                                 |109971888|0.5092583354137223|


## テーブル同士を結合する節
UNION節は，テーブルとテーブルを縦にくっつける役割を果たします。

### UNION ALL

```sql
SELECT number1 FROM ( VALUES 1,2,3 ) AS t(number1)
UNION ALL
SELECT number2 FROM ( VALUES 1,3,5 ) AS t(number2)
```
|number1                                    |
|-------------------------------------------|
|1                                          |
|2                                          |
|3                                          |
|1                                          |
|3                                          |
|5                                          |


UNION ALLは，挟まれた前後の2つのSELECT文の結果を縦にそのままつなげます。テーブル名の違いは1番目の名前に吸収されますが，以下のように型が異なれば同じカラムには収まりません。

```sql
SELECT number FROM ( VALUES 1,2,3 ) AS t(number)
UNION ALL
SELECT alphabet FROM ( VALUES 'a','b','c' ) AS t(alphabet)
```
```sql
column 1 in UNION query has incompatible types: integer, varchar(1)
```
また，異なる列構成のテーブル同士も結合できません。
```sql
SELECT number FROM ( VALUES 1,2,3 ) AS t(number)
UNION ALL
SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
```sql
```sql
UNION query has different number of fields: 1, 2
```

さらに，元のテーブルの列構成は同じでも，SELECT節で並べる順序が異なればうまくUNION ALLできません。

```sql
SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
UNION ALL
SELECT alphabet, number FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
```
```sql
column 1 in UNION query has incompatible types: integer, varchar(1)
```

以下は正しく動きます。
```sql
SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
UNION ALL
SELECT number, alphabet FROM ( VALUES (1,'a'),(3,'c'),(5,'d') ) AS t(number, alphabet)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|1                                          |a       |
|2                                          |b       |
|3                                          |c       |
|1                                          |a       |
|3                                          |c       |
|5                                          |d       |


### UNION

UNIONとUNION ALLは挙動が異なります。UNION ALLが無条件に2つのテーブルをくっつけるのに対して，UNIONは重複を省いてテーブルをくっつけてくれます。

```sql
SELECT number1 FROM ( VALUES 1,2,3 ) AS t(number1)
UNION
SELECT number2 FROM ( VALUES 1,3,5 ) AS t(number2)
```
|number1                                    |
|-------------------------------------------|
|5                                          |
|1                                          |
|3                                          |
|2                                          |


複数の列構成では，SELECT節で列挙されたすべてのカラムの組合せの値が一致する場合が重複とみなされます。

```sql
SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
UNION
SELECT number, alphabet FROM ( VALUES (1,'a'),(3,'d'),(5,'e') ) AS t(number, alphabet)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|2                                          |b       |
|3                                          |c       |
|3                                          |d       |
|5                                          |e       |
|1                                          |a       |


### INTERSECT

UNIONが重複を省いたのに対し，INTERSECTは重複しているレコードのみを結合結果とします。

```sql
SELECT number1 FROM ( VALUES 1,2,3 ) AS t(number1)
INTERSECT
SELECT number2 FROM ( VALUES 1,3,5 ) AS t(number2)
```
|number1                                    |
|-------------------------------------------|
|1                                          |
|3                                          |


```sql
SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
INTERSECT
SELECT number, alphabet FROM ( VALUES (1,'a'),(3,'d'),(5,'e') ) AS t(number, alphabet)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|1                                          |a       |


### EXCEPT

EXCEPTは，1番目のテーブルから「2番目のテーブルで同じレコードの組合せのもの」を除外します。EXCEPTの結果はテーブルの記述順序で異なります。
```sql
SELECT number1 FROM ( VALUES 1,2,3 ) AS t(number1)
EXCEPT
SELECT number2 FROM ( VALUES 1,3,5 ) AS t(number2)
```
|number1                                    |
|-------------------------------------------|
|2                                          |


```sql
SELECT number2 FROM ( VALUES 1,3,5 ) AS t(number2)
EXCEPT
SELECT number1 FROM ( VALUES 1,2,3 ) AS t(number1)
```
|number2                                    |
|-------------------------------------------|
|5                                          |


```sql
SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
EXCEPT
SELECT number, alphabet FROM ( VALUES (1,'a'),(3,'d'),(5,'e') ) AS t(number, alphabet)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|2                                          |b       |
|3                                          |c       |


```sql
SELECT number, alphabet FROM ( VALUES (1,'a'),(3,'d'),(5,'e') ) AS t(number, alphabet)
EXCEPT
SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|5                                          |e       |
|3                                          |d       |


## GROUPING SETS，CUBE，ROLLUP

これから紹介するGROUP BYの応用は，データウェアハウスなどでは当然のごとく行われている処理群です。これらの処理では多くのNULLが発生するため，元々あったNULLとの違いを区別する必要がありますが，まずはそれを考慮しないで説明していきます。

### GROUPING SETS

GROUPING SETSは，異なるディメンジョンの組合せでのGROUP BY集計の結合を，UNIONよりもシンプルに記述できる記法です。

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (category),
  (category,sub_category),
  (category,sub_category,goods_id)
)
```
|category                                   |sub_category|goods_id|sales   |
|-------------------------------------------|------------|--------|--------|
|Automotive and Industrial                  |Safety      |542003  |32062   |
|Electronics and Computers                  |Wearable Technology|NULL        |16601999|
|Electronics and Computers                  |Home Audio and Theater|540139  |72390   |


上記のクエリでは，以下を一度に計算しています。

- categoryごとの売上集計
- category × sub_categoryごとの売上集計
- category × sub_category × goods_idごとの売上集計

本来ならば下記のようにUNION ALLを用いる必要があります。

```sql
SELECT category, NULL AS sub_category, NULL AS goods_id, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY category

UNION ALL
SELECT category, sub_category, NULL AS goods_id, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY category, sub_category
  
UNION ALL
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY category, sub_category, goods_id
```
|category                                   |sub_category|goods_id|sales   |
|-------------------------------------------|------------|--------|--------|
|Electronics and Computers                  |Car Electronics and GPS|494460  |25551   |
|Books and Audible                          |Textbooks   |494913  |38136   |
|Sports and Outdoors                        |Golf        |496492  |57600   |


上記の書き方には，UNION ALLでカラムを揃えるためにディメンジョンに入らないカラムについてSELECT節内で適宜NULLを入れるといった考慮が必要であり，冗長であるという難点があります。また，上の例では全体の売上集計は含まれませんが，

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (),
  (category),
  (category,sub_category),
  (category,sub_category,goods_id)
)
```
|category                                   |sub_category|goods_id|sales   |
|-------------------------------------------|------------|--------|--------|
|Electronics and Computers                  |Musical Instruments|530651  |18000   |
|Sports and Outdoors                        |Leisure Sports and Game Room|NULL        |12641876|
|Home and Garden and Tools                  |Home Automation|NULL        |14593758|


と記述することによってそれも含むことが可能です。
GROUPING SETSは便利ですが，今のところSETS内でTD_TIME_FORMATなどの関数を用いることはできないようです。

```sql
SELECT TD_TIME_FORMAT(time,'yyyy-MM-01','JST'),category, sub_category, goods_id, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (TD_TIME_FORMAT(time,'yyyy-MM-01','JST'),category),
  (TD_TIME_FORMAT(time,'yyyy-MM-01','JST'),category,sub_category),
  (TD_TIME_FORMAT(time,'yyyy-MM-01','JST'),category,sub_category,goods_id)
)
```
```sql
mismatched input '('. Expecting: ')', ',', '.'
```

上記のクエリと同等のことを実現するためには以下のように書きます。

```sql
SELECT m, category, sub_category, goods_id, SUM(sales) AS sales
FROM 
(
  SELECT TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m,category, sub_category, goods_id, price*amount AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
)
GROUP BY GROUPING SETS(
  (m,category),
  (m,category,sub_category),
  (m,category,sub_category,goods_id)
)
```
|m                                          |category|sub_category|goods_id|sales  |
|-------------------------------------------|--------|------------|--------|-------|
|2011-10-01                                 |Movies and Music and Games|Musical Instruments|532989  |25755  |
|2011-10-01                                 |Automotive and Industrial|Automotive Tools and Equipment|531186  |25840  |
|2011-10-01                                 |Home and Garden and Tools|Hardware    |NULL        |1870255|

なお，最初に紹介したクエリでは「category ⊃ sub_category ⊃ goods_is」という階層を意識した記述をしていますが，ディメンジョン間の階層を意識せずに下記のように記述することも可能です。

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (category),
  (category,sub_category),
  (goods_id)
)
```
|category                                   |sub_category|goods_id|sales   |
|-------------------------------------------|------------|--------|--------|
|NULL                                       |NULL        |526973  |495282  |
|NULL                                       |NULL        |526422  |53074   |
|NULL                                       |NULL        |524133  |151480  |


最後に，元々カラムの値としてあったNULLと，GROUPING SETSによって生じたNULLとを区別する方法を紹介します。下記のように多少複雑です。

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales,
  GROUPING(category,sub_category,goods_id) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (category),
  (category,sub_category),
  (category,sub_category,goods_id)
)
```
|category                                   |sub_category|goods_id|sales   |group_id|
|-------------------------------------------|------------|--------|--------|--------|
|Clothing and Shoes and Jewelry             |Boys        |540286  |13328   |0       |
|Clothing and Shoes and Jewelry             |Women       |539249  |13328   |0       |
|Clothing and Shoes and Jewelry             |Boys        |NULL    |15730594|1       |


GROUPING(category,sub_category,goods_id)により，引数に指定した1番右のディメンジョンから順に以下のルールで値が割り振られます。

- goods_id = 0^2 = 1
- sub_category → 1^2 = 2
- category → 2^2 = 4

そして，各レコードには，GROUPING上でNULLとなった（オリジナルでない）カラムに対する上記の値の足し算の結果が入ります。つまり，以下の各値が入ることになります。
- category  →  sub_category, goods_idがNULL → 1+2 = 3
- category × sub_category → goods_idがNULL → 1
- category × sub_category × goods_id → NULLカラムなし → 0

### ROLLUP
ROLLUPは，GROUPING SETSの特殊版で，階層を意識した集計を適宜行います。引数として記述したディメンジョンのすべてについて，サブ集計を行います。まず引数が2つの場合を見てみましょう。

```sql
SELECT category, sub_category, SUM(price*amount) AS sales,
  GROUPING(category,sub_category) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY ROLLUP(category,sub_category)
ORDER BY group_id, category, sub_category
```
|category                                   |sub_category|goods_id|sales   |group_id|
|-------------------------------------------|------------|--------|--------|--------|
|Automotive and Industrial                  |Lab and Scientific|538429  |14940   |0       |
|Movies and Music and Games                 |Digital Games|538349  |155690  |0       |
|Electronics and Computers                  |NULL            |NULL        |290630163|3       |


上記のクエリは，下記の要領でサブ集計を巻き上げてくれます。
- category × sub_category ごとの売上集計（group_id = 0）
- category ごとの売上集計（group_id = 1）
- 全体の売上集計（group_id = 3）

上記のクエリをGROUPING SETSで書くと下記のようになります。2番目に指定しているsub_category単体の集計は行われていないことに注意してください（つまり指定する順序が重要です）。

```sql
SELECT category, sub_category, SUM(price*amount) AS sales,
  GROUPING(category,sub_category) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (category,sub_category),
  (category),
  ()
)
ORDER BY group_id, category, sub_category
```
|category                                   |sub_category|sales |group_id|
|-------------------------------------------|------------|------|--------|
|Automotive and Industrial                  |Automotive Parts and Accessories|15404860|0       |
|Automotive and Industrial                  |Automotive Tools and Equipment|15047016|0       |
|Automotive and Industrial                  |Car/Vehicle Electronics and GPS|14112998|0       |


次に，3つのディメンジョンのROLLUPを考えましょう。

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales,
  GROUPING(category,sub_category) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY ROLLUP(category,sub_category,goods_id)
ORDER BY group_id, category, sub_category, goods_id
```
|category                                   |sub_category|goods_id|sales   |group_id|
|-------------------------------------------|------------|--------|--------|--------|
|Automotive and Industrial                  |Automotive Parts and Accessories|466889  |24572   |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|466983  |3800    |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|467077  |3410    |0       |


上記のクエリは，下記の要領でサブ集計を巻き上げてくれます。

- category × sub_category × goods_idごとの売上集計（group_id = 0）
- category × sub_categoryごとの売上集計（group_id = 1）
- categoryごとの売上集計（group_id = 3）
- 全体の売上集計（group_id = 7）

なお，下記のようなクエリにより，上記の組合せについてGROUPING関数で割り当てられる値を知ることができます。

```sql
WITH groups1
AS
( SELECT name,id
  FROM (
  VALUES 
    ('1_category',    POWER(2,2)),
    ('2_sub_category',POWER(2,1)),
    ('3_goods_id',    POWER(2,0)),
    ('1_NULL',                 0),
    ('2_NULL',                 0),
    ('3_NULL',                 0)
  ) AS t(name,id)
)

SELECT groups1.name AS name1, groups2.name AS name2, groups3.name AS name3, 
  CAST( (4+2+1-groups1.id-groups2.id-groups3.id) AS INTEGER ) AS group_id
FROM groups1
JOIN
( SELECT name, id FROM groups1 ) AS groups2
ON groups1.name <> groups2.name 
AND SUBSTR(groups1.name,1,1) ='1' AND SUBSTR(groups2.name,1,1) ='2'
JOIN
( SELECT name, id FROM groups1 ) AS groups3
ON groups2.name <> groups3.name AND groups1.name <> groups3.name
AND SUBSTR(groups3.name,1,1) ='3'
WHERE (4+2+1-groups1.id-groups2.id-groups3.id) IN (0,1,3,7)
ORDER BY group_id
```
|name1                                      |name2|name3 |group_id|
|-------------------------------------------|-----|------|--------|
|1_category                                 |2_sub_category|3_goods_id|0       |
|1_category                                 |2_sub_category|3_NULL|1       |
|1_category                                 |2_NULL|3_NULL|3       |
|1_NULL                                     |2_NULL|3_NULL|7       |


3引数の場合のROLLUPクエリの例は，以下のGROUPING SETSと同じです。

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales,
  GROUPING(category,sub_category) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (category,sub_category,goods_id),
  (category,sub_category),
  (category),
  ()
)
ORDER BY group_id, category, sub_category, goods_id
```
|category                                   |sub_category|goods_id|sales   |group_id|
|-------------------------------------------|------------|--------|--------|--------|
|Automotive and Industrial                  |Automotive Parts and Accessories|466889  |24572   |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|466983  |3800    |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|467077  |3410    |0       |


最後に，TD_TIME_FORMATを使ったROLLUPを考えます。こちらはサブクエリを書く必要があります。また，TD_TIME_FORMATを必ず先頭のディメンジョンにすると目的にかなう場合が多いようです。

```sql
SELECT m, category, sub_category, goods_id, SUM(sales) AS sales,
  GROUPING(category,sub_category) AS group_id
FROM 
(
  SELECT TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m,category, sub_category, goods_id, price*amount AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
)
GROUP BY ROLLUP (m,category,sub_category,goods_id)
ORDER BY m,category, sub_category, goods_id
```
|m                                          |category|sub_category|goods_id|sales|group_id|
|-------------------------------------------|--------|------------|--------|-----|--------|
|2011-01-01                                 |Automotive and Industrial|Automotive Parts and Accessories|466889  |24572|0       |
|2011-01-01                                 |Automotive and Industrial|Automotive Parts and Accessories|466983  |3800 |0       |
|2011-01-01                                 |Automotive and Industrial|Automotive Parts and Accessories|467077  |3410 |0       |


上記のクエリは，下記の要領でサブ集計を巻き上げてくれます。
- m × category × sub_category × goods_idごとの売上集計（group_id = 0）
- m × category × sub_categoryごとの売上集計（group_id = 1）
- m × categoryごとの売上集計（group_id = 3）
- mごとの売上集計（group_id = 7）
- 全体の売上集計（group_id = 15）

この組合せに対するGROUPING関数の値は以下のクエリで作れます。

```sql
WITH groups1
AS
( SELECT name,id
  FROM (
  VALUES 
    ('1_month',       POWER(2,3)),
    ('2_category',    POWER(2,2)),
    ('3_sub_category',POWER(2,1)),
    ('4_goods_id',    POWER(2,0)),
    ('1_NULL',                 0),
    ('2_NULL',                 0),
    ('3_NULL',                 0),
    ('4_NULL',                 0)
  ) AS t(name,id)
)

SELECT groups1.name AS name1, groups2.name AS name2, groups3.name AS name3, groups4.name AS name4, 
  CAST( (8+4+2+1-groups1.id-groups2.id-groups3.id-groups4.id) AS INTEGER ) AS group_id
FROM groups1
JOIN
( SELECT name, id FROM groups1 ) AS groups2
ON groups1.name <> groups2.name 
AND SUBSTR(groups1.name,1,1) ='1' AND SUBSTR(groups2.name,1,1) ='2'
JOIN
( SELECT name, id FROM groups1 ) AS groups3
ON groups2.name <> groups3.name AND groups1.name <> groups3.name
AND SUBSTR(groups3.name,1,1) ='3'
JOIN
( SELECT name, id FROM groups1 ) AS groups4
ON groups3.name <> groups4.name AND groups2.name <> groups4.name AND groups1.name <> groups4.name
AND SUBSTR(groups4.name,1,1) ='4'
WHERE (8+4+2+1-groups1.id-groups2.id-groups3.id-groups4.id) IN (0,1,3,7,15)
ORDER BY group_id
```           
|name1                                      |name2|name3 |name4   |group_id|
|-------------------------------------------|-----|------|--------|--------|
|1_month                                    |2_category|3_sub_category|4_goods_id|0       |
|1_month                                    |2_category|3_sub_category|4_NULL  |1       |
|1_month                                    |2_category|3_NULL|4_NULL  |3       |
|1_month                                    |2_NULL|3_NULL|4_NULL  |7       |
|1_NULL                                     |2_NULL|3_NULL|4_NULL  |15      |

### CUBE

CUBEもGROUPING SETSの特殊版です。こちらはカラムのNULLを含めたすべての組合せを考えてくれます。ディメンジョンが2つの場合のクエリの例を下記に示します。

```sql
SELECT category, sub_category, SUM(price*amount) AS sales,
  GROUPING(category,sub_category) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY CUBE(category,sub_category)
ORDER BY group_id, category, sub_category
```
|category                                   |sub_category|sales |group_id|
|-------------------------------------------|------------|------|--------|
|Automotive and Industrial                  |Automotive Parts and Accessories|15404860|0       |
|Automotive and Industrial                  |Automotive Tools and Equipment|15047016|0       |
|Automotive and Industrial                  |Car/Vehicle Electronics and GPS|14112998|0       |


上記のクエリは，GROUPING関数で割り当てられるgroup_idごとに下記の各集計を求めてくれます。

- 全体の売上集計
- categoryごとの売上集計
- sub_categoryごとの売上集計
- category × sub_categoryごとの売上集計

GROUPING SETSで書くと下記のようになります。

```sql
SELECT category, sub_category, SUM(price*amount) AS sales,
  GROUPING(category,sub_category) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (),
  (category),
  (sub_category),
  (category,sub_category)
)
ORDER BY group_id, category, sub_category
```
|category                                   |sub_category|sales |group_id|
|-------------------------------------------|------------|------|--------|
|Automotive and Industrial                  |Automotive Parts and Accessories|15404860|0       |
|Automotive and Industrial                  |Automotive Tools and Equipment|15047016|0       |
|Automotive and Industrial                  |Car/Vehicle Electronics and GPS|14112998|0       |


次はCUBEに3つのディメンジョンを与えてみましょう。

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales,
  GROUPING(category,sub_category,goods_id) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY CUBE(category,sub_category,goods_id)
ORDER BY group_id, category, sub_category, goods_id
```
|category                                   |sub_category|goods_id|sales   |group_id|
|-------------------------------------------|------------|--------|--------|--------|
|Automotive and Industrial                  |Automotive Parts and Accessories|466889  |24572   |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|466983  |3800    |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|467077  |3410    |0       |


上記のクエリは，下記の各集計を求めてくれます。
- category × sub_category × goods_id ごとの売上集計
- category × sub_category ごとの売上集計
- category × goods_id ごとの売上集計
- sub_category × goods_id ごとの売上集計
- category ごとの売上集計
- sub_category ごとの売上集計
- goods_id ごとの売上集計
- 全体の売上集計

group_idごとに組合せを意識した書き方をすれば以下のようになります。
- category × sub_category × goods_id	（group_id = 0）
- category × sub_category × NULL	（group_id = 1）
- category × NULL × goods_id		（group_id = 2）
- NULL × sub_category × goods_id	（group_id = 4）
- category × NULL × NULL		（group_id = 1+2=3）
- NULL × sub_category × NULL	（group_id = 1+4=5）
- NULL × NULL × goods_id		（group_id = 2+4=6）
- NULL × NULL × NULL		（group_id = 1+2+4=7）

CUBEで作られる組合せとGROUPING関数の値を同時に知るには，以下のクエリを書きます。

```sql
WITH groups1
AS
( SELECT name,id
  FROM (
  VALUES 
    ('1_category',    POWER(2,2)),
    ('2_sub_category',POWER(2,1)),
    ('3_goods_id',    POWER(2,0)),
    ('1_NULL',                 0),
    ('2_NULL',                 0),
    ('3_NULL',                 0)
  ) AS t(name,id)
)

SELECT groups1.name AS name1, groups2.name AS name2, groups3.name AS name3, 
  CAST( (4+2+1-groups1.id-groups2.id-groups3.id) AS INTEGER ) AS group_id
FROM groups1
JOIN
( SELECT name, id FROM groups1 ) AS groups2
ON groups1.name <> groups2.name 
AND SUBSTR(groups1.name,1,1) ='1' AND SUBSTR(groups2.name,1,1) ='2'
JOIN
( SELECT name, id FROM groups1 ) AS groups3
ON groups2.name <> groups3.name AND groups1.name <> groups3.name
AND SUBSTR(groups3.name,1,1) ='3'
ORDER BY group_id
```
|name1                                      |name2|name3 |group_id|
|-------------------------------------------|-----|------|--------|
|1_category                                 |2_sub_category|3_goods_id|0       |
|1_category                                 |2_sub_category|3_NULL|1       |
|1_category                                 |2_NULL|3_goods_id|2       |
|1_category                                 |2_NULL|3_NULL|3       |
|1_NULL                                     |2_sub_category|3_goods_id|4       |
|1_NULL                                     |2_sub_category|3_NULL|5       |
|1_NULL                                     |2_NULL|3_goods_id|6       |
|1_NULL                                     |2_NULL|3_NULL|7       |


先程の3つのディメンジョンに対するCUBEによるクエリは，以下のGROUPING SETSによるクエリと等価です。CUBEは，簡潔さと自動的に組合せを考えてくれる点で有用です。

```sql
SELECT category, sub_category, goods_id, SUM(price*amount) AS sales,
  GROUPING(category,sub_category,goods_id) AS group_id
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY GROUPING SETS(
  (),
  (category),
  (sub_category),
  (goods_id),
  (category,sub_category),
  (category,goods_id),
  (sub_category,goods_id),
  (category,sub_category,goods_id)
)
ORDER BY group_id, category, sub_category, goods_id
```
|category                                   |sub_category|goods_id|sales   |group_id|
|-------------------------------------------|------------|--------|--------|--------|
|Automotive and Industrial                  |Automotive Parts and Accessories|466889  |24572   |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|466983  |3800    |0       |
|Automotive and Industrial                  |Automotive Parts and Accessories|467077  |3410    |0       |


CUBEでも，ROLLUPと同じく，TD_TIME_FORMATなどの関数を入れることはできません。必要な場合は下記のように一度サブクエリを作る書き方をします。ない，時間カラムがNULLとされる集計はここでは望ましくないので省いています。

```sql
SELECT m, category, sub_category, goods_id, SUM(sales) AS sales,
  GROUPING(category,sub_category,goods_id) AS group_id
FROM 
(
  SELECT TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m,category, sub_category, goods_id, price*amount AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
)
GROUP BY CUBE (m,category,sub_category,goods_id)
HAVING m IS NOT NULL
ORDER BY group_id, m, category, sub_category, goods_id
```
|m                                          |category|sub_category|goods_id|sales|group_id|
|-------------------------------------------|--------|------------|--------|-----|--------|
|2011-01-01                                 |Automotive and Industrial|Automotive Parts and Accessories|466889  |24572|0       |
|2011-01-01                                 |Automotive and Industrial|Automotive Parts and Accessories|466983  |3800 |0       |
|2011-01-01                                 |Automotive and Industrial|Automotive Parts and Accessories|467077  |3410 |0       |

この場合，CUBEが作る組合せとGROUPING関数の値は以下のクエリで取得できます。

```sql
WITH groups1
AS
( SELECT name,id
  FROM (
  VALUES 
    ('1_month',       POWER(2,3)),
    ('2_category',    POWER(2,2)),
    ('3_sub_category',POWER(2,1)),
    ('4_goods_id',    POWER(2,0)),
    ('1_NULL',                 0),
    ('2_NULL',                 0),
    ('3_NULL',                 0),
    ('4_NULL',                 0)
  ) AS t(name,id)
)

SELECT groups1.name AS name1, groups2.name AS name2, groups3.name AS name3, groups4.name AS name4, 
  CAST( (8+4+2+1-groups1.id-groups2.id-groups3.id-groups4.id) AS INTEGER ) AS group_id
FROM groups1
JOIN
( SELECT name, id FROM groups1 ) AS groups2
ON groups1.name <> groups2.name 
AND SUBSTR(groups1.name,1,1) ='1' AND SUBSTR(groups2.name,1,1) ='2'
JOIN
( SELECT name, id FROM groups1 ) AS groups3
ON groups2.name <> groups3.name AND groups1.name <> groups3.name
AND SUBSTR(groups3.name,1,1) ='3'
JOIN
( SELECT name, id FROM groups1 ) AS groups4
ON groups3.name <> groups4.name AND groups2.name <> groups4.name AND groups1.name <> groups4.name
AND SUBSTR(groups4.name,1,1) ='4'
ORDER BY group_id
```
|name1                                      |name2|name3 |name4   |group_id|
|-------------------------------------------|-----|------|--------|--------|
|1_month                                    |2_category|3_sub_category|4_goods_id|0       |
|1_month                                    |2_category|3_sub_category|4_NULL  |1       |
|1_month                                    |2_category|3_NULL|4_goods_id|2       |
|1_month                                    |2_category|3_NULL|4_NULL  |3       |
|1_month                                    |2_NULL|3_sub_category|4_goods_id|4       |
|1_month                                    |2_NULL|3_sub_category|4_NULL  |5       |
|1_month                                    |2_NULL|3_NULL|4_goods_id|6       |
|1_month                                    |2_NULL|3_NULL|4_NULL  |7       |
|1_NULL                                     |2_category|3_sub_category|4_goods_id|8       |
|1_NULL                                     |2_category|3_sub_category|4_NULL  |9       |
|1_NULL                                     |2_category|3_NULL|4_goods_id|10      |
|1_NULL                                     |2_category|3_NULL|4_NULL  |11      |
|1_NULL                                     |2_NULL|3_sub_category|4_goods_id|12      |
|1_NULL                                     |2_NULL|3_sub_category|4_NULL  |13      |
|1_NULL                                     |2_NULL|3_NULL|4_goods_id|14      |
|1_NULL                                     |2_NULL|3_NULL|4_NULL  |15      |


ここまでに紹介したGROUPING SETS，ROLLUP，CUBEは，さらに併用して使うことも可能です。しかし，カラムの組合せが複雑になるため，ここではこれ以上取り上げないこととします。

## EXISTS，IN，=：WHERE節のサブクエリと比較

### EXISTS
EXISTSにより，片方のテーブルに存在するkeyで他方のテーブルの行を抽出できます。
```sql
WITH t1 AS
( SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet) ),
t2 AS
( SELECT number FROM ( VALUES 1, 3, 5 ) AS t(number) )

SELECT number, alphabet
FROM t1
WHERE EXISTS (SELECT * FROM t2 WHERE t1.number = t2.number)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|1                                          |a       |
|3                                          |c       |


### IN
前述のEXISTSは，INを用いることでシンプルかつ可読性が良い形に書き換えることができます。

```sql
WITH t1 AS
( SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet) ),
t2 AS
( SELECT number FROM ( VALUES 1, 3, 5 ) AS t(number) )

SELECT number, alphabet
FROM t1
WHERE number IN (SELECT number FROM t2)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|1                                          |a       |
|3                                          |c       |


### Scalarサブクエリ

```sql
WITH t1 AS
( SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet) ),
t2 AS
( SELECT number FROM ( VALUES 1, 3, 5 ) AS t(number) )

SELECT number, alphabet
FROM t1
WHERE number = (SELECT MIN(number) FROM t2)
```
|number                                     |alphabet|
|-------------------------------------------|--------|
|1                                          |a       |


```sql
--結果なし（t2のMAX=5に対してt1には5の値のnumberカラムが存在しない）
WITH t1 AS
( SELECT number, alphabet FROM ( VALUES (1,'a'),(2,'b'),(3,'c') ) AS t(number, alphabet) ),
t2 AS
( SELECT number FROM ( VALUES 1, 3, 5 ) AS t(number) )

SELECT number, alphabet
FROM t1
WHERE number = (SELECT MAX(number) FROM t2)
```
