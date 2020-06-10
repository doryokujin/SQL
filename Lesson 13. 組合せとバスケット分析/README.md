# Lesson 13. 組合せとバスケット分析

ここでは，巨大なデータテーブル同士の力技，JOINによる組合せ計算を扱います。ポイントは，様々な条件下でのカラム値とカラム値の組合せを，SQLを使って柔軟に求めるところから始めることです。

## Numbers：2つ数値の組合せ

今，1〜6までの数値が入った「numbers」テーブルがあるとします。このテーブルの1〜6までの数字を使って，2つの数字の組合せをSQLで作ってみましょう。
本章では，様々な条件での組合せを広い意味で「組合せ」と呼ぶことにします。すなわち，数学では区別される順列（nr, P(n,r)）と組合せ（C(n,r)）の両方をすべて含む広い意味で「組合せ」という言葉を使います。

### 1. 2つの数値の組合せをSQLですべて求める
ある1つのカラムに対して，存在する値の組合せを求めることは，SQLで同じテーブルを参照するCROSS JOINを用いて簡単に記述できます。CROSS JOINは以下のように書けます。

```sql
FROM table1 n1
CROSS JOIN table2 n2
```

さらに，ここでは WITH 節で作ったサンプルテーブル同士の全組合せを算出するので，以下のようにCROSS JOINを書きます。

```sql
WITH numbers AS
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) )

SELECT n1.number, n2.number
FROM numbers n1, numbers n2
ORDER BY n1.number, n2.number
```
|number|number|
|------|------|
|1     |1     |
|1     |2     |
|...   |      |
|6     |5     |
|6     |6     |


上記の結果からわかるように，36個の数値の組合せ結果がすべて得られました。

### 2. 2つの数値の組合せで，同じ数値の組合せは避ける

WHERE節を使うことで，同じ数値を除外した組合せを求められます。

```sql
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) )

SELECT n1.number, n2.number
FROM numbers n1, numbers n2
WHERE n1.number <> n2.number
ORDER BY n1.number, n2.number
```
|number|number|
|------|------|
|1     |2     |
|1     |3     |
|...   |      |
|6     |4     |
|6     |5     |


今度は30個の数値の組合せ結果が得られました。

### ※重要：「CROSS JOIN+WHERE」と「JOIN+ON」の違い
2.のクエリ例は「CROSS JOIN+WHERE」の組合せです。これは「JOIN+ON」の組合せでも書けます。
```sql
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) )

SELECT n1.number, n2.number
FROM numbers n1
JOIN numbers n2
ON n1.number <> n2.number
ORDER BY n1.number, n2.number
```

前章の「WHEREとONの違い」と同じく，JOINの前に実行できるON節と，JOINの後で実行されるWHERE節とでは，前者のほうがパフォーマンスが良くなります。ただし，ある程度大きなデータを（特にHiveで）扱う場合以外には，これを意識することはあまりないでしょう。このドキュメントでも，「CROSS JOIN+WHERE」のほうを好んで使っている場面がいくつかあります。
なお，Hive0.13においては，ON節の両テーブルのキー同士の比較で「=」以外の「<>」や「<=」などが使えません。そのため，以下のHiveクエリはエラーとなります（片側のテーブルと定数の比較ならばANDやORで追記して書けます）。Hiveでは「CROSS JOIN+WHERE」を使うようにしましょう。

```sql
-- Hive
WITH numbers AS 
( SELECT STACK(6,1,2,3,4,5,6) AS number  )

SELECT n1.number, n2.number
FROM numbers n1
JOIN numbers n2
ON n1.number <> n2.number
ORDER BY n1.number, n2.number
```

上記のクエリをHiveで実行すると
```sql
SemanticException : Both left and right aliases encountered in JOIN 'number'
```
というエラーになります。

### 3. 2つの数値の組合せで順序を気にしない
今度は，（1, 2）と（2, 1）を同じものとして扱った組合せを求めます。さらに，2.と同様に同じ数値同士の場合も除外します。また，JOINを使った同様のクエリを並列しておきます。

```sql
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) )

SELECT n1.number, n2.number
FROM numbers n1, numbers n2
WHERE n1.number < n2.number
ORDER BY n1.number, n2.number
```
```sql
WITH numbers AS 
( SELECT number FROM ( VALUES 1,2,3,4,5,6 ) AS t(number) )

SELECT n1.number, n2.number
FROM numbers n1 JOIN numbers n2
ON n1.number < n2.number
ORDER BY n1.number, n2.number
```

|number|number|
|------|------|
|1     |2     |
|1     |3     |
|...   |      |
|4     |5     |
|4     |6     |
|5     |6     |


30個の結果が得られた2.に対し，半分の15個の組合せが得られました。

## Matching：食器のマッチング
次は下記のような「tableware」テーブル（皿：2枚，カップ：3つ）を使って組合せを考えてみましょう。
|id |type|
|---|----|
|d1 |dish|
|d2 |dish|
|c1 |cup |
|c2 |cup |
|c3 |cup |

数値の組合せの例の3.と同様に，考えうるペア（組合せ）をすべて求めてみましょう。

```sql
WITH tableware AS 
( SELECT id, type FROM ( 
  VALUES 
('d1','dish'),('d2','dish'),('c1','cup'),('c2','cup'),('c3','cup')
) AS t(id,type) )

SELECT n1.id, n2.id, n1.id || ' & ' || n2.id
FROM tableware n1, tableware n2
WHERE n1.id < n2.id
ORDER BY n1.id, n2.id
```
|id |id |_col2  |
|---|---|-------|
|c1 |c2 |c1 & c2|
|c1 |c3 |c1 & c3|
|c1 |d1 |c1 & d1|
|c1 |d2 |c1 & d2|
|c2 |c3 |c2 & c3|
|c2 |d1 |c2 & d1|
|c2 |d2 |c2 & d2|
|c3 |d1 |c3 & d1|
|c3 |d2 |c3 & d2|
|d1 |d2 |d1 & d2|


全部で10の組合せが得られました。

## 4. 可能な皿とカップのマッチングを考える

今度は，typeの値が異なるもののみを結合するようにします。
```sql
WITH tableware AS 
( SELECT id, type FROM ( 
  VALUES 
('d1','dish'),('d2','dish'),('c1','cup'),('c2','cup'),('c3','cup')
) AS t(id,type) )

SELECT n1.id, n2.id, n1.id || ' & ' || n2.id
FROM tableware n1, tableware n2
WHERE n1.id < n2.id
AND n1.type <> n2.type
ORDER BY n1.id, n2.id
```
|id |id |_col2  |
|---|---|-------|
|c1 |d1 |c1 & d1|
|c1 |d2 |c1 & d2|
|c2 |d1 |c2 & d1|
|c2 |d2 |c2 & d2|
|c3 |d1 |c3 & d1|
|c3 |d2 |c3 & d2|


## トランプの組合せ
今度はジョーカーを除く52枚のトランプの組合せを考えてみましょう。

### 5. 同じ絵柄のカードであるが，異なる数字の組合せを考える
同じ絵柄同士での異なる数字の組合せを考え，登場回数（すべて1回ですが）を数えてみます。
```sql
WITH trump AS 
( SELECT number, symbol FROM ( 
  VALUES 
('♦',1),('♦',2),('♦',3),('♦',4),('♦',5),('♦',6),('♦',7),('♦',8),('♦',9),('♦',10),('♦',11),('♦',12),('♦',13),
('♤',1),('♤',2),('♤',3),('♤',4),('♤',5),('♤',6),('♤',7),('♤',8),('♤',9),('♤',10),('♤',11),('♤',12),('♤',13),
('♣',1),('♣',2),('♣',3),('♣',4),('♣',5),('♣',6),('♣',7),('♣',8),('♣',9),('♣',10),('♣',11),('♣',12),('♣',13),
('♡',1),('♡',2),('♡',3),('♡',4),('♡',5),('♡',6),('♡',7),('♡',8),('♡',9),('♡',10),('♡',11),('♡',12),('♡',13)
) AS t(symbol,number) )

SELECT n1.symbol, n1.number, n2.number, COUNT(1) AS cnt
FROM trump n1, trump n2
WHERE n1.symbol = n2.symbol
AND n1.number < n2.number
GROUP BY n1.symbol, n1.number, n2.number
ORDER BY cnt DESC, n1.symbol, n1.number, n2.number
```
|symbol|number|number |cnt|
|------|------|-------|---|
|♡     |1     |2      |1  |
|♡     |1     |3      |1  |
|...   |      |       |   |
|♦     |11    |13     |1  |
|♦     |12    |13     |1  |


結果は312通りとなりました。

### 6. 絵柄を区別せず，数字の組合せの登場回数を考える
今度は，同じ絵柄の考えられる数字の組合せを求め，絵柄に関係なく各数字の組合せの登場回数を考えます。

```sql
WITH trump AS 
( SELECT number, symbol FROM ( 
  VALUES 
('♦',1),('♦',2),('♦',3),('♦',4),('♦',5),('♦',6),('♦',7),('♦',8),('♦',9),('♦',10),('♦',11),('♦',12),('♦',13),
('♤',1),('♤',2),('♤',3),('♤',4),('♤',5),('♤',6),('♤',7),('♤',8),('♤',9),('♤',10),('♤',11),('♤',12),('♤',13),
('♣',1),('♣',2),('♣',3),('♣',4),('♣',5),('♣',6),('♣',7),('♣',8),('♣',9),('♣',10),('♣',11),('♣',12),('♣',13),
('♡',1),('♡',2),('♡',3),('♡',4),('♡',5),('♡',6),('♡',7),('♡',8),('♡',9),('♡',10),('♡',11),('♡',12),('♡',13)
) AS t(symbol,number) )

SELECT number1, number2, COUNT(1) AS cnt
FROM
(
  SELECT n1.symbol AS symbol, n1.number AS number1, n2.number AS number2
  FROM trump n1, trump n2
  WHERE n1.symbol = n2.symbol
  AND n1.number < n2.number
) tmp
GROUP BY number1, number2
ORDER BY cnt DESC, number1, number2
```
|number1|number2|cnt    |
|-------|-------|-------|
|1      |2      |4      |
|1      |3      |4      |
|...    |       |       |
|11     |13     |4      |
|12     |13     |4      |


すべての数字のペアが4回ずつ登場しているのがわかります。
## バスケット分析

次のような対応関係で，トランプの例（5.および6.）をsales_slipに当てはめて考えてみましょう。
- 絵柄 → receit_id
- 数字 → goods_id
- データ → sales_slip

すると，5.の例は「同時購入されたgoods_idペアの考えうる組合せ」を考えていることになります。そして6.の例は，「同時購入されたgoods_idペアの登場回数（共起回数）」を数えていることになります。
さらに，次のような対応関係をsales_slipに当てはめて考えてみましょう。
- 絵柄 → member_id
この場合，5.の例は「各member_idが過去購入したgoods_idの考えうる組合せ」を考えていることになります。そして6.の例は，「同じ人に購入されたgoods_idペアの登場回数（共起回数）」を数えていることになります。

### 共起回数 | A ∩ B | （一定期間内での共起）

それでは応用してみましょう。同じユーザーに購入されたグッズのペアが，多くのユーザーで現れるならば，そのペアは何らかの関係（一緒に買われている根拠）があるといえるでしょう。ここでは，各ユーザーが「同時」に購入したグッズのペアを1回の「共起」として捉え，その回数を見ていきます。ショッピングサイトの以下のような文言として見覚えがあるかもしれません。
「このアイテムを買ったユーザーは同時にこのグッズを買っています」
本ドキュメントでは，この文言における「同時」の概念を2種類紹介しています。「一定期間内」と「同日」です。まずは前者から見ていきましょう。
以下のクエリでは，一定期間中（2011年）に購入されたユーザーごとのグッズを（ユニークに）リストアップし，リスト内の異なるグッズの組合せを数え上げ，全体で集計しています。一定期間内に何回も同じペアが共起したとしても，それは1回とカウントされます。
```sql
WITH gm_stat AS
(
  SELECT member_id, goods_id
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, goods_id
)

SELECT g1.goods_id AS goods_id1, g2.goods_id AS goods_id2, COUNT(1) AS cnt
FROM gm_stat g1, gm_stat g2
WHERE g1.member_id=g2.member_id
AND g1.goods_id<g2.goods_id
GROUP BY g1.goods_id, g2.goods_id
ORDER BY cnt DESC
LIMIT 10
```
|goods_id1|goods_id2|cnt    |
|---------|---------|-------|
|541456   |547453   |158    |
|546452   |547453   |153    |
|531458   |547453   |139    |

また，JOINでの同様のクエリも併記しておきます。

```sql
WITH gm_stat AS
(
  SELECT member_id, goods_id
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, goods_id
)

SELECT g1.goods_id AS goods_id1, g2.goods_id AS goods_id2, COUNT(1) AS cnt
FROM gm_stat g1 JOIN gm_stat g2
ON g1.member_id=g2.member_id
AND g1.goods_id<g2.goods_id
GROUP BY g1.goods_id, g2.goods_id
ORDER BY cnt DESC
LIMIT 10
```

このクエリを，グッズ単体や全体の登場回数とともに，WITH節を用いてシンプルにまとめます。
```sql
WITH gm_stat AS
(
  SELECT member_id, goods_id
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, goods_id
),
goods_stat AS
(
  SELECT goods_id, COUNT(1) AS cnt
  FROM gm_stat
  GROUP BY goods_id
),
stat AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM goods_stat
),
combi AS
(
  SELECT g1.goods_id AS goods_id1, g2.goods_id AS goods_id2, COUNT(1) AS combi_cnt
  FROM gm_stat g1, gm_stat g2
  WHERE g1.member_id = g2.member_id
  AND g1.goods_id<g2.goods_id
  GROUP BY g1.goods_id, g2.goods_id
)

SELECT goods_id1, goods_id2, g1.cnt AS cnt1, g2.cnt AS cnt2, combi_cnt, total_cnt
FROM combi, goods_stat g1, goods_stat g2, stat
WHERE combi.goods_id1=g1.goods_id AND combi.goods_id2=g2.goods_id
ORDER BY combi_cnt DESC
LIMIT 10
```
|goods_id1|goods_id2|cnt1   |cnt2|combi_cnt|total_cnt|
|---------|---------|-------|----|---------|---------|
|541456   |547453   |413    |1700|158      |445222   |
|546452   |547453   |259    |1700|153      |445222   |
|531458   |547453   |394    |1700|139      |445222   |


また，アイテム同士の共起の可視化として，以下の様なグラフ表現も有効です（以下のグラフは今回の結果とは異なるものであることをご了承ください）。各ノードがアイテムを表し，共起回数を辺の太さで表現しています。

### 共起回数 | A ∩ B | （同日購入を共起とする）
次に，一定期間内ではなく，同日に購入されたペアを「同時購入」としてカウントしてみます。同じユーザーでも，異なる日に同時に購入していれば，そのペアは別にカウントされることになります。
```sql
WITH goods_stat AS
(
  SELECT goods_id, COUNT(1) AS cnt
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY goods_id
),
orders AS
(
  SELECT DENSE_RANK()OVER(PARTITION BY member_id ORDER BY d) AS shopping_order, goods_id, member_id
  FROM
  (
    SELECT goods_id, member_id, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY goods_id, member_id, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
  )
),
stat AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM
  (
    SELECT member_id, MAX(shopping_order) AS cnt
    FROM orders
    GROUP BY member_id
  )
),
combi AS
(
  SELECT 
    o1.goods_id AS goods_id1, o2.goods_id AS goods_id2, COUNT(1) AS combi_cnt
  FROM orders o1, orders o2
  WHERE  o1.member_id = o2.member_id 
  AND o1.shopping_order = o2.shopping_order 
  AND o1.goods_id < o2.goods_id
  GROUP BY o1.goods_id, o2.goods_id
)

SELECT goods_id1, goods_id2, g1.cnt AS cnt1, g2.cnt AS cnt2, combi_cnt, total_cnt
FROM combi, goods_stat g1, goods_stat g2, stat
WHERE combi.goods_id1=g1.goods_id AND combi.goods_id2=g2.goods_id
ORDER BY combi_cnt DESC
LIMIT 10
```
|goods_id1|goods_id2|cnt1   |cnt2|combi_cnt|total_cnt|
|---------|---------|-------|----|---------|---------|
|546452   |547453   |327    |2213|110      |202354   |
|538660   |540882   |371    |216 |87       |202354   |
|545690   |547453   |247    |2213|84       |202354   |


### 遷移回数 | A → B |，| B → A |
今度は，「同時」ではなく，（同じユーザーの）「その次」の買い物で登場したグッズとペアリングすることで，ショッピングサイトでは次のような文言で表されるような意味を持ったペアを抽出することを考えてみます。
「このグッズを買ったユーザーは次にこのグッズを買っています」

ここでは，このペアを「遷移」と呼ぶことにします。そして，あるユーザーのn回目の買い物で購入されたグッズAと，n+1回目の買い物で買われたグッズBを，「グッズAからグッズBへの（1ステップ）遷移」としてカウントします。
以下のクエリは，1ステップ遷移のグッズペアをカウントするものです。1ステップ遷移では前と後で同じグッズが現れてもきちんと数えます。むしろその場合は連続して購入されている注目すべきグッズという解釈ができます。
```sql
WITH goods_stat AS
(
  SELECT goods_id, COUNT(1) AS cnt
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY goods_id
),
orders AS
(
  SELECT DENSE_RANK()OVER(PARTITION BY member_id ORDER BY d) AS shopping_order, goods_id, member_id
  FROM
  (
    SELECT goods_id, member_id, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY goods_id, member_id, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
  )
),
stat AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM
  (
    SELECT member_id, MAX(shopping_order) AS cnt
    FROM orders
    GROUP BY member_id
  )
),
trans AS
(
  SELECT 
    o1.goods_id AS goods_id1, o2.goods_id AS goods_id2, COUNT(1) AS trans_cnt
  FROM orders o1, orders o2
  WHERE o1.member_id = o2.member_id 

  AND o1.shopping_order+1 = o2.shopping_order
  GROUP BY o1.goods_id, o2.goods_id
)

SELECT goods_id1, goods_id2, g1.cnt AS cnt1, g2.cnt AS cnt2, trans_cnt, total_cnt
FROM trans, goods_stat g1, goods_stat g2, stat
WHERE trans.goods_id1=g1.goods_id AND trans.goods_id2=g2.goods_id
ORDER BY trans_cnt DESC
LIMIT 10
```
|goods_id1|goods_id2|cnt1   |cnt2|trans_cnt|total_cnt|
|---------|---------|-------|----|---------|---------|
|547453   |547453   |2213   |2213|238      |202354   |
|109601   |109601   |540470 |540470|199      |202354   |
|543766   |547453   |356    |2213|51       |202354   |


次は，「1ステップ」の遷移に限らず，同じユーザーがグッズAを購入した以降にグッズBを購入した場合にカウントする，より汎用的なクエリを紹介します。注意点として，1人のユーザーに対して「グッズA→グッズB」は1回のみカウントすることとします（つまり，グッズAの後にグッズBの購入が複数回あっても1回，グッズA→グッズB→...→グッズA→グッズBのような場合も1回とみなします）。また，この場合の全体の数（total_cnt）は概念が難しいので省略しています。

```sql
WITH goods_stat AS
(
  SELECT goods_id, COUNT(1) AS cnt
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY goods_id
),
orders AS
(
  SELECT goods_id, member_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST') 
  GROUP BY goods_id, member_id
),
trans AS 
(
  SELECT 
    o1.goods_id AS goods_id1, o2.goods_id AS goods_id2, COUNT(1) AS trans_cnt
  FROM orders o1, orders o2
  WHERE o1.member_id = o2.member_id 
  AND ( o1.min_time < o2.min_time OR o2.min_time < o1.max_time )
  AND o1.goods_id <> o2.goods_id
  GROUP BY o1.goods_id, o2.goods_id
)

SELECT goods_id1, goods_id2, g1.cnt AS cnt1, g2.cnt AS cnt2, trans_cnt
FROM trans, goods_stat g1, goods_stat g2
WHERE trans.goods_id1=g1.goods_id AND trans.goods_id2=g2.goods_id
ORDER BY trans_cnt DESC
LIMIT 10
```
|goods_id1|goods_id2|cnt1   |cnt2|trans_cnt|
|---------|---------|-------|----|---------|
|547453   |541456   |2213   |843 |142      |
|541456   |547453   |843    |2213|140      |
|531458   |547453   |658    |2213|139      |


「グッズA→グッズB」と「グッズA→グッズB」では意味が異なるので，同じ値になっていないことに注目してください。とはいえ，遷移回数が上位のグッズペアは期間を通して複数回買われているものが多く，その中で両方の向きの遷移が起こっているので，近い回数の値になっていると考えられます。

### 共起係数
共起回数は「絶対的」な数値です。ペア同士に強い関係があったとしても，双方の登場回数がそもそも少なければ，結果の上位にくることはありません。一方，共起係数という「相対的」な指標を使えば，絶対的な大きさにとらわれずに関係の強いペアを上位に見出すことができます。
ここでは，共起係数を表す指標として代表的なJaccard係数，Simpson係数，Cosine係数，Dice係数の4つを説明します。これらの係数に優劣はありません。それぞれの特徴を把握してうまく使い分けることが肝要です。
まず以下のクエリにより，共起回数の説明と同じ例題で4つの係数を求めます。ベースとする共起回数は「一定期間内」のほう，すなわち，1つ目に説明した「同じユーザーでも一定期間の間に複数回共起した場合は1とする」方針のものです。
```sql
WITH gm_stat AS
(
  SELECT member_id, goods_id
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, goods_id
),
goods_stat AS
(
  SELECT goods_id, COUNT(1) AS cnt
  FROM gm_stat
  GROUP BY goods_id
),
stat AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM goods_stat
),
combi AS
(
  SELECT g1.goods_id AS goods_id1, g2.goods_id AS goods_id2, COUNT(1) AS combi_cnt
  FROM gm_stat g1, gm_stat g2
  WHERE g1.member_id = g2.member_id
  GROUP BY g1.goods_id, g2.goods_id
  HAVING g1.goods_id<g2.goods_id
)

SELECT goods_id1, goods_id2, g1.cnt AS cnt1, g2.cnt AS cnt2, combi_cnt, total_cnt,
  1.0*combi_cnt/(g1.cnt+g2.cnt-combi_cnt)       AS jaccard_coeff,
  1.0*combi_cnt/IF(g1.cnt<g2.cnt,g1.cnt,g2.cnt) AS simpson_coeff,
  1.0*combi_cnt/SQRT(g1.cnt*g2.cnt)             AS cos_coeff,
  1.0*combi_cnt/(g1.cnt+g2.cnt)                 AS dice_coeff
FROM combi, goods_stat g1, goods_stat g2, stat
WHERE combi.goods_id1=g1.goods_id AND combi.goods_id2=g2.goods_id
ORDER BY combi_cnt DESC
LIMIT 10
```
|goods_id1|goods_id2|cnt1   |cnt2|combi_cnt|total_cnt|jaccard_coeff      |simpson_coeff      |cos_coeff          |dice_coeff         |
|---------|---------|-------|----|---------|---------|-------------------|-------------------|-------------------|-------------------|
|541456   |547453   |413    |1700|158      |445222   |0.08081841432225063|0.38256658595641646|0.1885634868608601 |0.07477520113582584|
|546452   |547453   |259    |1700|153      |445222   |0.08471760797342193|0.5907335907335908 |0.23057758600094497|0.0781010719754977 |
|531458   |547453   |394    |1700|139      |445222   |0.0710997442455243 |0.35279187817258884|0.16984087893220706|0.06638013371537727|


共起回数が大きいからといって共起係数も大きいとは限りません。それぞれの係数の計算方法と解釈を見ていきましょう。

#### Jaccard係数：| A ∩ B | / | A ∪ B |
| A ∩ B | と | A ∪ B | の比を見るのがJaccard係数です。この係数は上右図のように，AとBのサイズが同じでかつ深く交わっている場合に大きな値を取ります。
#### Simpson係数：| A ∩ B | / MIN ( | A |, | B | )
Simpson係数では，値の小さいほうのアイテムAの登場回数を分母に取るので，共起回数と少ないほうのアイテムの登場回数が近ければ近いほど値が大きくなります。上右図で理解するならば，小さいほうのアイテムが大きいほうに深く入り込んでいる場合にSimpson係数は極めて高くなります。
それゆえに，Simpson係数が大きいペアにおける小さいほうのアイテムAは，単独では登場せずに必ず他のアイテムと共起して現れてくる特殊なアイテムを表しています。
#### Cosine係数：| A ∩ B | / SQRT ( | A | * | B | )
Cosine係数は，ペアの単独での登場回数がほぼ近い関係にあって共起回数が多いものを発見してくれます。どちらかの登場回数に引っ張られる共起回数や，少ないほうに引っ張られるSimpson係数に比べて，バランスの良い係数といえます。
#### Dice係数：2 * | A ∩ B | / ( | A | + | B | )
Dice係数もCosine係数と同様の性質を持っています。

### レコメンデーション係数
レコメンデーション係数が共起係数と大きく異なるのは，AとBの方向が考慮される点です。共起係数からは，AとBが一緒に購入されたことは見えますが，「Aを買った後にBを買った」のか（またその逆か）は言及されません。レコメンデーション係数では，この意味でAとBに方向性があり，購入の順番が重要視されるときに利用されます。レコメンデーション係数には3種類ありますが，いずれも単体で有効な指標となることはあまりなく，3つの指標を同時に眺める必要があります。まずは各種レコメンデーション係数を求めるクエリを下記に示します。
```sql
WITH gm_stat AS
(
  SELECT member_id, goods_id
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, goods_id
),
goods_stat AS
(
  SELECT goods_id, COUNT(1) AS cnt
  FROM gm_stat
  GROUP BY goods_id
),
stat AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM goods_stat
),
combi AS
(
  SELECT g1.goods_id AS goods_id1, g2.goods_id AS goods_id2, COUNT(1) AS combi_cnt
  FROM gm_stat g1, gm_stat g2
  WHERE g1.member_id = g2.member_id
  GROUP BY g1.goods_id, g2.goods_id
  HAVING g1.goods_id<>g2.goods_id
)

SELECT goods_id1, goods_id2, g1.cnt AS cnt1, g2.cnt AS cnt2, combi_cnt, total_cnt,
  1.0*combi_cnt/g1.cnt AS confidence,
  1.0*combi_cnt/total_cnt AS support,
  (1.0*combi_cnt/g1.cnt) / (1.0*g2.cnt/total_cnt) AS lift
FROM combi, goods_stat g1, goods_stat g2, stat
WHERE combi.goods_id1=g1.goods_id AND combi.goods_id2=g2.goods_id
ORDER BY combi_cnt DESC
LIMIT 10
```
|goods_id1|goods_id2|cnt1   |cnt2|combi_cnt|total_cnt|confidence         |support            |lift               |
|---------|---------|-------|----|---------|---------|-------------------|-------------------|-------------------|
|547453   |541456   |1700   |413 |158      |445222   |0.09294117647058824|0.00035487913894641324|100.1923885486398  |
|541456   |547453   |413    |1700|158      |445222   |0.38256658595641646|0.00035487913894641324|100.1923885486398  |
|546452   |547453   |259    |1700|153      |445222   |0.5907335907335908 |0.00034364878644810906|154.71034749034752 |


#### Confidence係数：| A ∩ B | / | A |

信頼度を表すConfidenceは「グッズAを買った人が，グッズBも買う確率」で表現されます。この式の分母から推察できるように，（遷移回数と同様に）AとBには方向性があることに注意してください。
#### Support係数：| A ∩ B | / | Ω |

支持度を表すSupportは，全体の中でグッズAとグッズBがどれくらい一緒に買われているかを見る指標です。全グッズの登場回数を|Ω|の値が大きい場合は，この係数は非常に小さくなりがちなので，注意してください。

#### Lift係数：( | A ∩ B | / | A | ) / ( | B | / | Ω | )
Lift係数は，以下の比を表したものです。
「分母」：グッズA を買った人がグッズBも買う割合
「分子」：グッズBの全グッズの登場回数に対する割合
すなわちLift係数は，「Aと一緒にBも購入した人の割合（Confidence）が，Bの全グッズの登場回数に対する割合よりどれだけ多いか」を倍率で示したものです。 
 
