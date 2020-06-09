# Lesson 09. PivotとUnpivot

## 縦持ちテーブルと横持ちテーブルの相互変換

### Pivot（縦持ち → 横持ち）

今，td_client_idごとにタイトルごとのページビューを集計した下記のようなテーブルがあるとします。

```sql
SELECT td_client_id, td_title AS key, COUNT(1) AS value
FROM sample_accesslog
GROUP BY td_client_id, td_title
ORDER BY td_client_id
```

このような，それぞれのidごとに複数のkeyに対応する値（value）を別々の行として保持するテーブルを「縦持ち」テーブルと呼びます。縦持ちテーブルでは，keyのバリエーション数だけ同じidの行が存在するので，GROUP BYで簡単に値を取り出せます。一方，結果の解釈にはやや難のあるテーブルです。

以降では，下記のような一般化した縦持ちテーブルを作成し，それを例に説明します。ここでは，「縦持ちテーブルは1つのkeyに対して値が1つ（複数の値を持たない）」としておきます。

```sql
WITH vtable AS 
( SELECT id,key,value
  FROM ( VALUES
    (1,'col1',11),(1,'col2',12),(1,'col3',13),
    (2,'col1',21),(2,'col2',22),(2,'col3',23),
    (3,'col1',31),(3,'col2',32),(3,'col3',33)
  ) AS t(id,key,value) 
)
SELECT id, key, value
FROM vtable
```

縦持ちテーブルに対し，idが1行にしか現れないように異なるkeyの値を別々のカラムとして横方向に展開した以下のようなテーブルのことを，「横持ちテーブル」と呼びます。

縦持ちテーブルから横持ちテーブルへの変換（Pivot）を実行するクエリを考えましょう。縦持ちテーブルの各行をMAPにkeyと値のペアを登録することで，簡単に横持ちテーブルへと展開可能です。

```sql
WITH vtable AS 
( SELECT id,key,value
  FROM ( VALUES
    (1,'col1',11),(1,'col2',12),(1,'col3',13),
    (2,'col1',21),(2,'col2',22),(2,'col3',23),
    (3,'col1',31),(3,'col2',32),(3,'col3',33)
  ) AS t(id,key,value) 
)

SELECT
  id,
  kv['col1'] AS col1,
  kv['col2'] AS col2,
  kv['col3'] AS col3
FROM (
  SELECT id, MAP_AGG(key, value) AS kv
  FROM vtable
  GROUP BY id
)
ORDER BY id
```

上記のクエリは，テンプレートとして理解するには問題ありませんが，実は欠陥があり，すべてのidがすべてのkeyに対する値を持ち合わせている場合のみ使えます。例えば，id=3がcol3というkeyに対応する値を持っていない場合に，次のように上記のクエリを実行したとしましょう。

```sql
WITH vtable AS 
( SELECT id,key,value
  FROM ( VALUES
    (1,'col1',11),(1,'col2',12),(1,'col3',13),
    (2,'col1',21),(2,'col2',22),(2,'col3',23),
    (3,'col1',31),(3,'col2',32)--,(3,'col3',33)
  ) AS t(id,key,value) 
)

SELECT
  id,
  kv['col1'] AS col1,
  kv['col2'] AS col2,
  kv['col3'] AS col3
FROM (
  SELECT id, MAP_AGG(key, value) AS kv
  FROM vtable
  GROUP BY id
)
ORDER BY id
```

すると，id=3において，「key='col3'がMAPに存在しない」と怒られます。
```sql
Key not present in map: col3
```

このようなエラーを回避するには，ELEMENT_AT関数がkeyが存在しない場合にNULLを返すことを利用して，次のようなIF文でカバーしておきます。

```sql
WITH vtable AS 
( SELECT id,key,value
  FROM ( VALUES
    (1,'col1',11),(1,'col2',12),(1,'col3',13),
    (2,'col1',21),(2,'col2',22),(2,'col3',23),
    (3,'col1',31),(3,'col2',32)--,(3,'col3',33)
  ) AS t(id,key,value) 
)

SELECT
  id,
  IF(ELEMENT_AT(kv,'col1') IS NOT NULL, kv['col1'], NULL) AS col1,
  IF(ELEMENT_AT(kv,'col2') IS NOT NULL, kv['col2'], NULL) AS col2,
  IF(ELEMENT_AT(kv,'col3') IS NOT NULL, kv['col3'], NULL) AS col3
FROM (
  SELECT id, MAP_AGG(key, value) AS kv
  FROM vtable
  GROUP BY id
)
ORDER BY id
```

### TD_PIVOT（Hive2019.1以降のみ）
Hive2019.1でTD_PIVOT関数が使えるようになりました。この関数では，PrestoのMAP_AGG(key,value)と同じように，第1引数と第2引数にはそれぞれkeyと値のカラムを指定します。さらに第3引数に，カラムとして取り出したいkeyをカンマ区切りで列挙して全体を（個々にでなく）「' '」で囲みます。TD_PIVOTは，GROUP BYと同じ文脈に記述するので，カラムを取り出す際には（今回の例では値は1つですが）集約関数（MAXやMINなど）を挟む必要があります。

```sql
WITH vtable AS 
( 
  SELECT STACK(
    9, --行数をはじめに定義！
    1,'col1',11,1,'col2',12,1,'col3',13,
    2,'col1',21,2,'col2',22,2,'col3',23,
    3,'col1',31,3,'col2',32,3,'col3',33
  ) AS (id,key,value)
)

SELECT id, MAX(t.col1) AS col1, MAX(t.col2) AS col2, MAX(t.col3) AS col3
FROM vtable
LATERAL VIEW TD_PIVOT(key,value,'col1,col2,col3') t
GROUP BY id
ORDER BY id
```

この関数のメリットは，一部のkeyが欠損していても，NULLで埋めてエラーなく結果を返してくれることです。

```sql
WITH vtable AS 
( 
  SELECT STACK(
    8, --行数をはじめに定義！
    1,'col1',11,1,'col2',12,1,'col3',13,
    2,'col1',21,2,'col2',22,2,'col3',23,
    3,'col1',31,3,'col2',32--,3,'col3',33
  ) AS (id,key,value)
)

SELECT id, MAX(t.col1) AS col1, MAX(t.col2) AS col2, MAX(t.col3) AS col3
FROM vtable
LATERAL VIEW TD_PIVOT(key,value,'col1,col2,col3') t
GROUP BY id
ORDER BY id
```

さらに，この関数の強みとして，あるkeyに対する値が複数あっても実行できます。MAP_AGGを利用したPivotでは，1つのkeyに複数の値を持たせられないというMAPの性質から，これは実現不可能です。
keyに対する値が複数ある場合は，集約の際，MAXではなくCOLLECT_SETを使って集合（重複しない値のLIST）として取り出すことに注意してください。

```sql
WITH vtable AS 
( 
  SELECT STACK(
    15, --行数をはじめに定義！
    1,'col1',11,1,'col2',12,1,'col3',13,
    2,'col1',21,2,'col2',22,2,'col3',23,
    3,'col1',31,3,'col2',32,3,'col3',33,
    1,'col1',111,1,'col2',121,1,'col3',131,
    2,'col1',212,2,'col2',222,2,'col3',232
  ) AS (id,key,value)
)

SELECT id, COLLECT_SET(t.col1) AS col1, COLLECT_SET(t.col2) AS col2, COLLECT_SET(t.col3) AS col3
FROM vtable
LATERAL VIEW TD_PIVOT(key,value,'col1,col2,col3') t
GROUP BY id
ORDER BY id
```

### Unpivot（横持ち → 縦持ち）
次は，横持ちテーブルから縦持ちテーブルへの変換（Unpivot）を考えます。改めて説明すると，「横持ちテーブル」とは，あるid（キー）に対する同じ型（というより同じ意味の）の複数の項目を別々のカラムとして横（列）方向に展開して保持しておくテーブルのことです。以下に横持ちテーブルの例を示します。

```sql
WITH htable AS 
( SELECT id,col1,col2,col3
  FROM ( VALUES
    (1,11,12,13),(2,21,22,23),(3,31,32,33)
  ) AS t(id,col1,col2,col3) 
)

SELECT id,col1,col2,col3 FROM htable
```

この横持ちテーブルを縦持ちテーブルに変換するには，カラムを1つのLISTにしてからUNNESTで縦に伸ばしてやります。

```sql
WITH htable AS 
( SELECT id,col1,col2,col3
  FROM ( VALUES
    (1,11,12,13),(2,21,22,23),(3,31,32,33)
  ) AS t(id,col1,col2,col3) 
)

SELECT t1.id, t2.key, t2.value
FROM htable t1
CROSS JOIN UNNEST (
  array['col1','col2','col3'],
  array[ col1,  col2,  col3 ]
) t2 (key, value)
```

このクエリを理解するために，まずはUNNESTを理解しましょう。UNNESTは下記のようにLISTを行に展開します。

```sql
WITH list_table AS
(
  SELECT ARRAY[1,2,3,4] AS list
)

SELECT x
FROM list_table
CROSS JOIN UNNEST(list) AS t(x)
```
WITH ORDINALITYを付け加えることで，LISTにおける位置を同時にカラムとして保存しておくことができます。

```sql
WITH list_table AS
(
  SELECT ARRAY['a','b','c','d'] AS list
)

SELECT id, x
FROM list_table
CROSS JOIN UNNEST(list) WITH ORDINALITY AS t(id,x)
```

MAPもUNNESTにより2列の行に展開することができます。

```sql
WITH map_table AS
(
  SELECT MAP( ARRAY['a','b','c'], ARRAY[1,2,3] ) AS mp
)

SELECT k,v
FROM map_table
CROSS JOIN UNNEST(mp) AS t(k,v)
```
テーブルもUNNESTできます。テーブルの場合には，UNNESTにより，複数のカラム（型は同じ）を行に展開することができます。

```sql
WITH htable AS 
( SELECT id,col1,col2,col3
  FROM ( VALUES
    (1,11,12,13),(2,21,22,23),(3,31,32,33)
  ) AS t(id,col1,col2,col3) 
)

SELECT id,value
FROM htable
CROSS JOIN UNNEST( ARRAY[col1,col2,col3] ) AS t(value)
```

カラムを結合する際には，元のカラムが何だったかに関する情報があると便利です。MAPの展開を思い出すと，以下のように書けます。

```sql
WITH htable AS 
( SELECT id,col1,col2,col3
  FROM ( VALUES
    (1,11,12,13),(2,21,22,23),(3,31,32,33)
  ) AS t(id,col1,col2,col3) 
)

SELECT id,key,value
FROM htable
CROSS JOIN UNNEST( MAP(ARRAY['col1','col2','col3'], ARRAY[col1,col2,col3]) ) AS t(key,value)
```

UNNESTでは，上記と同じ意図を以下のように書くことでも実現できます。同じ結果になることがわかりますね。

```sql
WITH htable AS 
( SELECT id,col1,col2,col3
  FROM ( VALUES
    (1,11,12,13),(2,21,22,23),(3,31,32,33)
  ) AS t(id,col1,col2,col3) 
)

SELECT id,key,value
FROM htable
CROSS JOIN UNNEST( 
  ARRAY['col1','col2','col3'], 
  ARRAY[col1,col2,col3] 
) AS t(key,value)
```

UNNESTでは3つ以上のLISTの列挙も可能なので，結果の列をさらに増やすこともできます（Pivotの文脈では意味がありませんが）。

```sql
WITH htable AS 
( SELECT id,col1,col2,col3
  FROM ( VALUES
    (1,11,12,13),(2,21,22,23),(3,31,32,33)
  ) AS t(id,col1,col2,col3) 
)

SELECT id,x,key,value
FROM htable
CROSS JOIN UNNEST( ARRAY['1','2','3'], ARRAY['col1','col2','col3'], ARRAY[col1,col2,col3] ) AS t(x,key,value)
```

### TD_UNPIVOT （Hive2019.1以降のみ）
Hive2019.1（Hive2.0以降）でTD_UNPIVOT関数が使えるようになりました。TD_UNPIVOTの第1引数には，keyとなるカラム名をカンマ区切りでまとめて「' '」で囲んで指定します。個々のカラム名を「' '」で囲むわけではないので注意してください。

```sql
WITH htable AS 
( 
  SELECT STACK(
    3, --行数をはじめに定義！
    1,11,12,13,
    2,21,22,23,
    3,31,32,33
  ) AS (id,col1,col2,col3) 
)

SELECT id, t.key, t.value
FROM htable
LATERAL VIEW TD_UNPIVOT(
 'col1, col2, col3', --一括して''で包む！
  col1, col2, col3
) t
```

## 月別カテゴリ別売上のPivot

### 集計結果の縦持ちテーブル展開
はじめに，sales_slipから2011年におけるカテゴリ別，月別の売上集計を縦持ちテーブルとして出力する例を紹介します。

```sql
SELECT category, TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, SUM(price*amount) AS sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY category, TD_TIME_FORMAT(time,'yyyy-MM-01','JST')
ORDER BY category, m
```

### 集計結果の横持ちテーブル展開
応用例として，sales_slipから2011年におけるカテゴリ別の売上総額を取り出し，各行のカラムに各月の売上総額が入った横持ちテーブルを出力する例をSQLだけで書いてみます。SUMとIFの組合せなので，条件に満たない場合はNULLでも0でもかまいません。
1つ目のSUM（salesカラム）は年全体の売上総額であり，以降のSUM(IF)の各カラムの値の総和になるはずですが，漏れがある場合などは等しい値になりません。この記述では漏れがあるかどうかを自分で注意する以外ないことに注意してください。（ちなみに，縦持ちのGROUP BYなら漏れなく列挙できます。）

```sql
SELECT category,
  SUM(price*amount) AS sales,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='01',price*amount,0)) AS sales_01,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='02',price*amount,0)) AS sales_02,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='03',price*amount,0)) AS sales_03,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='04',price*amount,0)) AS sales_04,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='05',price*amount,0)) AS sales_05,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='06',price*amount,0)) AS sales_06,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='07',price*amount,0)) AS sales_07,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='08',price*amount,0)) AS sales_08,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='09',price*amount,0)) AS sales_09,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='10',price*amount,0)) AS sales_10,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='11',price*amount,0)) AS sales_11,
  SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='12',price*amount,0)) AS sales_12
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY category
ORDER BY category
```

### Pivot（縦持ち → 横持ち）
はじめから縦持ちか横持ちで集計するなら複雑ではないのですが，一度集計されてしまったものを相互変換するのはなかなか骨が折れます。まずは縦持ちから横持ちへの変換例を紹介します。

```sql
WITH vtable AS
(
  SELECT category, TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, SUM(price*amount) AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY category, TD_TIME_FORMAT(time,'yyyy-MM-01','JST')
  ORDER BY category, m
)

SELECT
  category, 
  sales,
  IF(ELEMENT_AT(kv,'2011-01-01') IS NOT NULL,kv['2011-01-01'],0) AS sales_01,
  IF(ELEMENT_AT(kv,'2011-02-01') IS NOT NULL,kv['2011-02-01'],0) AS sales_02,
  IF(ELEMENT_AT(kv,'2011-03-01') IS NOT NULL,kv['2011-03-01'],0) AS sales_03,
  IF(ELEMENT_AT(kv,'2011-04-01') IS NOT NULL,kv['2011-04-01'],0) AS sales_04,
  IF(ELEMENT_AT(kv,'2011-05-01') IS NOT NULL,kv['2011-05-01'],0) AS sales_05,
  IF(ELEMENT_AT(kv,'2011-06-01') IS NOT NULL,kv['2011-06-01'],0) AS sales_06,
  IF(ELEMENT_AT(kv,'2011-07-01') IS NOT NULL,kv['2011-07-01'],0) AS sales_07,
  IF(ELEMENT_AT(kv,'2011-08-01') IS NOT NULL,kv['2011-08-01'],0) AS sales_08,
  IF(ELEMENT_AT(kv,'2011-09-01') IS NOT NULL,kv['2011-09-01'],0) AS sales_09,
  IF(ELEMENT_AT(kv,'2011-10-01') IS NOT NULL,kv['2011-10-01'],0) AS sales_10,
  IF(ELEMENT_AT(kv,'2011-11-01') IS NOT NULL,kv['2011-11-01'],0) AS sales_11,
  IF(ELEMENT_AT(kv,'2011-12-01') IS NOT NULL,kv['2011-12-01'],0) AS sales_12
FROM (
  SELECT category, map_agg(m, sales) AS kv, SUM(sales) AS sales
  FROM vtable
  GROUP BY category
)
ORDER BY category
```

### Unpivot（横持ち → 縦持ち）

次は，横持ちから縦持ちへの変換例です。

```sql
WITH htable AS
(
  SELECT category,
    SUM(price*amount) AS sales,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='01',price*amount,0)) AS sales_01,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='02',price*amount,0)) AS sales_02,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='03',price*amount,0)) AS sales_03,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='04',price*amount,0)) AS sales_04,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='05',price*amount,0)) AS sales_05,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='06',price*amount,0)) AS sales_06,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='07',price*amount,0)) AS sales_07,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='08',price*amount,0)) AS sales_08,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='09',price*amount,0)) AS sales_09,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='10',price*amount,0)) AS sales_10,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='11',price*amount,0)) AS sales_11,
    SUM(IF(TD_TIME_FORMAT(time,'MM','JST')='12',price*amount,0)) AS sales_12
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY category
  ORDER BY category
)

SELECT t1.category, t2.key AS m, t2.value AS sales
FROM htable t1
CROSS JOIN UNNEST (
  array['2011-01-01','2011-02-01','2011-03-01','2011-04-01','2011-05-01','2011-06-01','2011-07-01','2011-08-01','2011-09-01','2011-10-01','2011-11-01','2011-12-01'],
  array[sales_01,sales_02,sales_03,sales_04,sales_05,sales_06,sales_07,sales_08,sales_09,sales_10,sales_11,sales_12]
) t2 (key, value)
ORDER BY category, m
```

