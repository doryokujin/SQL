# Lesson 15. ヒストグラム

## ヒストグラムと他のチャートとの違い

あるサンプルデータセットを入手した際には，そのデータセットがどのような統計的性質を持つのかを知り，どのような解析アプローチをとるかを探るため，抽出クエリや集計クエリ，サンプリングなどを駆使して「データを眺める」作業を行うはずです。このフェーズでは，「ヒストグラム」によってデータの分布を確認することが大変有効です。
ヒストグラムは，それなしにはデータを語れないと言われるほど重要でありながら，しばしば棒グラフと混同されがちです。そこで，まずは棒グラフとヒストグラムの違いをそれぞれ列挙します。

### ヒストグラムの特徴
- 数値変数（連続および離散）にのみ適用可能
- 数値変数が適当な区分に分割され，その区分に属するサンプルの個数（頻度），もしくは全体頻度に対する個々の区分の割合を表現する
- 頻度の大きさは棒の高さではなく，（棒の面積）= （棒の高さ）×（棒の幅）で表現される
- ゆえに，各々の区分が等幅である必要はない（棒の高さで調節できるので）
- 隣接する棒は，頻度が0でない限り，無駄に隙間があってはいけない
- 全体頻度（または，全体割合）はすべての棒の面積の総和になり，これは区分幅の取り方によらず一定である

### 棒グラフの特徴
- カテゴリ変数（数値ではないもの，主に順序付）にも適用可能
- 頻度に限らず，データのある観測値および集計値が（=Y軸への）表現対象
- 値の大きさは棒の高さのみによって定まる
- 各カテゴリにおける棒の幅（それには意味がないので）は等しくあるべきである
- 隣接する棒には隙間があったほうがよいが，隙間がない場合も多い
- 全体の総和はすべての棒の高さの総和である


ヒストグラムでは，面積で頻度を表現するので，等幅でない度数分布では各棒の幅が異なります。一方，棒グラフでは個々の度数が1つひとつの「カテゴリ」とみなされます。カテゴリ間の隙間には興味がなく，棒と棒の間は離しておくのが一般的です。
ただし，一般的な可視化ツールは棒の幅が可変のヒストグラムを描くのには対応していません。X軸における階級をすべて等間隔にした（できれば棒同士の隙間がないように設定した）棒グラフがヒストグラムを代替することになります。

## シンプルなHISTOGRAM
HISTOGRAM関数は，ヒストグラムの中でも最もシンプルで，値ごとの数を集計してくれるだけです。GROUP BYでも同様のことが簡単にできますが，そちらは返り値がMAPとなります。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,4,4,4,4,5,5,5,5,5,6,6,6,6,7,7,7,8,8,9 ) AS t(val)
),
histo AS (
  SELECT HISTOGRAM(val) AS res_map
  FROM sample
)

SELECT val, freq
FROM histo
CROSS JOIN UNNEST (
  res_map
) t2 (val, freq)
```

この結果には1から9までの値が欠損なく入っているので，そのまま棒グラフを描けばそれがヒストグラムにもなります。

## 欠損値の補完
HISTOGRAM関数でいくつかの値に欠損がある場合への対処を考えてみます。
```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 2,2,3,3,3,5,5,5,5,5,6,6,6,6,9 ) AS t(val)
),
histo AS (
  SELECT HISTOGRAM(val) AS res_map
  FROM sample
)

SELECT val, freq
FROM histo
CROSS JOIN UNNEST (
  res_map
) t2 (val, freq)
```

この結果を率直に棒グラフにしても，それはヒストグラムにはなりません。X軸の値の配置が等間隔ではないからです。

そこで，欠損している「1,4,7,8」のvalをfreq = 0として補完することにします。そのためには，別テーブルとして，「1,2,3,...,10000000」までの連番がtimeカラムに入った「serial_numbers」テーブルを参照します。まず，取りうる値の範囲の中で欠損する値を見つけるクエリを書きます。
```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 2,2,3,3,3,5,5,5,5,5,6,6,6,6,9 ) AS t(val)
)

SELECT time AS val
FROM serial_numbers
WHERE time < (SELECT MAX(val) FROM sample)
AND time NOT IN (SELECT val FROM sample)
```

この結果を先程の結果にUNION ALLすることで，ヒストグラム用の結果テーブルを出力します。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 2,2,3,3,3,5,5,5,5,5,6,6,6,6,9 ) AS t(val)
),
histo AS (
  SELECT HISTOGRAM(val) AS res_map
  FROM sample
)

SELECT val, freq
FROM histo
CROSS JOIN UNNEST (
  res_map
) t2 (val, freq)

UNION ALL
SELECT time AS val, 0 AS freq
FROM serial_numbers
WHERE time < (SELECT MAX(val) FROM sample)
AND time NOT IN (SELECT val FROM sample)

ORDER BY val
```

この棒グラフはヒストグラムを代替しているといえます。

### 年間購買分布
具体例として，sales_slipテーブルから2011年のmember_idごとの購買の分布を見てみることにします。まず，年間の購買額が100万円以上のサンプルは外れ値として除外しておくことにします。HISTOGRAM関数では，値が少しでも異なると集計が異なるため，大変な数のbin（「棒」のこと）を持ったヒストグラムになります。
```sql
with sales_table AS
(
  SELECT member_id, SUM(price*amount) AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  HAVING SUM(price*amount) <= 1000000
),
histo AS
(
  SELECT HISTOGRAM(sales) AS res_map
  FROM sales_table
)

SELECT val, freq
FROM histo
CROSS JOIN UNNEST (
  res_map
) t2 (val, freq)
UNION ALL
SELECT time AS val, 0 AS freq
FROM serial_numbers
WHERE time < (SELECT MAX(sales) FROM sales_table)
AND time NOT IN (SELECT sales FROM sales_table)
ORDER BY val
```
上記の結果テーブルは欠損値を埋める前のものです。かなりの欠損値が発生していることが見てとれます。欠損値を補完して1から1000までの極めて狭い範囲でヒストグラム化したのが右図です。1つひとつの値を棒として表現することには限界があるのがわかりますね。

そこで次は，ある程度の範囲を1つの棒として表現することを考えます。10000円区切りで，「10000円以下，20000円以下，…,1000000円以下」の範囲ごとで頻度を数え直し，ヒストグラムを描いてみましょう。

```sql
with sales_table AS
(
  SELECT member_id, SUM(price*amount) AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  HAVING SUM(price*amount) <= 1000000
),
histo AS
(
  SELECT HISTOGRAM(sales) AS res_map
  FROM sales_table
),
freq_table AS
(
  SELECT val, freq
  FROM histo
  CROSS JOIN UNNEST (
    res_map
  ) t2 (val, freq)

  UNION ALL
  SELECT time AS val, 0 AS freq
  FROM serial_numbers
  WHERE time < (SELECT MAX(sales) FROM sales_table)
  AND time NOT IN (SELECT sales FROM sales_table)
)

SELECT   (val/10000+1)*10000 AS val, SUM(freq) AS freq
FROM freq_table
GROUP BY (val/10000+1)*10000
ORDER BY val
```

## WIDTH_BUCKETによるヒストグラム

WIDTH_BUCKET(x, bound1, bound2, n)，は（等間隔の）ヒストグラムを描くために，値をグルーピング（バケットに分ける）する関数です。bound1とbound2で決めた区間を，n等分割し，カラム値xをそれぞれが属する区間に割り当てていきます。このとき，区間外にある値は，それぞれ無条件に最小と最大のバケットに入ります。区間外に値がある場合には，左に外れている値はバケット0に，右に外れている値はバケットn+1に入るので，指定したバケットの数nよりも結果が多く出ることに注意してください。

ヒストグラムの「棒」は，英語で「bin」と呼ばれます。ヒストグラムを描く際には，binの「数」と（1つのbinの）「幅」を決める必要があります。WIDTH_BUCKET関数では，binの数は第3引数の分割数，binの幅はすべて同じ「(bound2-bound1)/n」となります。

WIDTH_BUCKET関数を使う場合には，一般にbound1をxの最小値より少し小さい値，bound2をxの最大値より少し大きい値にしておくのが無難です（例外バケットが現れなくて済む）。外れ値がある場合には，それを除いた最大と最小を指定します。また，NULLはいかなるバケットにも割り当てられず，NULLを返します。例を見てみましょう。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,4,4,4,4,5,5,5,5,5,6,6,6,6,7,7,7,8,8,9 ) AS t(val)
)

SELECT val, bucket, 
  IF(bucket=6, INFINITY(), 2+1.0*(8-2)/5*bucket) AS upper_val,
  freq
FROM
(
  SELECT val, WIDTH_BUCKET(val,2,8,5) AS bucket, COUNT(1) AS freq
FROM sample
GROUP BY val, WIDTH_BUCKET(val,2,8,5)
)
ORDER BY val
```

上記のクエリでは，1〜9までの値を取りうるデータ範囲の中で，WIDTH_BUCKET関数の引数により [2,8) の区間を指定しています。これは「2以上8より小さい」という意味です。したがって，左の区間外は「1」でバケット0，右の区間外は「8」および「9」でいずれもバケット6に入ることになります。

[2,8) の区間に属する値については，width = (8-2)/5 = 1.2ごとに分割された区間[2, 3.2)，[3.2, 4.4)，[4.4, 5.6)，[5.6, 6.8)，[6.8, 8)のそれぞれに，バケット1，2，...，5が入り，そこに属する値の数を集計しています。クエリ内の「2+1.0*(8-2)/5*bucket」は，それぞれの区間の上限値（外れバケットの場合は無限大）を計算していることになります。

WIDTH_BUCKET(x, bound1, bound2, n)の引数と同じ記号で書けば「bound1+1.0*(bound2-bound1)/n*bucket」となります。

区間に属する値が存在しない場合，そのバケット番号は結果に存在せず欠損となります。次は欠損がある例を見てみましょう。
```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,7,7,7,8,8,9 ) AS t(val)
)

SELECT val, bucket, 
  IF(bucket=6, INFINITY(), 2+1.0*(8-2)/5*bucket) AS upper_val, 
  freq
FROM
(
  SELECT val, WIDTH_BUCKET(val,2,8,5) AS bucket, COUNT(1) AS freq
FROM sample
GROUP BY val, WIDTH_BUCKET(val,2,8,5)
)
ORDER BY val
```

この場合，ヒストグラムにするには，欠損したバケット番号を補完する必要があります。バケット番号を保管したうえで各バケットの上限だけでなく下限を求め，区間を意識したテーブルを描いてみます。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,7,7,7,8,8,9 ) AS t(val)
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,2,8,5) AS bucket, COUNT(1) AS freq
  FROM sample
  GROUP BY val, WIDTH_BUCKET(val,2,8,5)
)

SELECT val, bucket, 
  IF(bucket=0,-INFINITY(), 2+1.0*(8-2)/5*(bucket-1)) AS lower_val,
  IF(bucket=6, INFINITY(), 2+1.0*(8-2)/5*bucket)     AS upper_val,
  freq
FROM
(
  SELECT val, bucket, freq
  FROM bucket_table

  UNION ALL
  SELECT NULL AS val, time AS bucket, 0 AS freq
  FROM serial_numbers
  WHERE time < (SELECT MAX(bucket) FROM bucket_table)
  AND time NOT IN (SELECT bucket FROM bucket_table)
)
ORDER BY bucket, val
```
準備が長くなりましたが，これをbucketについて集計して棒グラフを出せばヒストグラムになります。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,7,7,7,8,8,9 ) AS t(val)
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,2,8,5) AS bucket, COUNT(1) AS freq
  FROM sample
  GROUP BY val, WIDTH_BUCKET(val,2,8,5)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,-INFINITY(),2+1.0*(8-2)/5*(bucket-1)) AS lower_val,
    IF(bucket=6, INFINITY(),2+1.0*(8-2)/5*bucket) AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  )
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

## 最大／最小を両端に指定したヒストグラムテンプレートクエリ（binの数nのみ指定）
区間の最大と最小を考慮して区間外バケットが存在しないようにし，binの数を5としたヒストグラムを考えます。以下のクエリでは，statテンポラリテーブルでbinの数（n）を5に指定するだけで，その他の部分でWIDTH_BUCKETの引数を手入力する必要がありません。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,7,7,7,8,8,9 ) AS t(val)
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx, 5 AS n --WIDTH_BUCKET(mn=1,mx=10,n=5)
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```
binの数を変えてみます。先程より少ない数（n = 3）を設定してみます。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,7,7,7,8,8,9 ) AS t(val)
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx, 3 AS n --WIDTH_BUCKET(mn=1,mx=10,n=3)
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

次は，binの数nを増やし，9（取りうる値の最大値）としてみます。レコードが取りうる整数値の1つひとつが区間になった状態です。それ以上のnを指定すると，区間が1以下となり，無駄が増えてしまいます。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,7,7,7,8,8,9 ) AS t(val)
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx, 9 AS n --WIDTH_BUCKET(mn=1,mx=10,n=9)
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

ヒストグラムの区間には正解はありません。SQLで自由に設定を変更しながら繰り返し描いてみることで，適材適所なヒストグラムに巡り合えるでしょう。

### 年間購買分布
年間購買数についても，WIDTH_BUCKET関数によるヒストグラムテンプレートクエリを使って分布を見てみましょう。HISTOGRAM関数と違ってbinの区間の幅を柔軟に設定できるので，今度はbinの数を多めにして20としてみることで，すっきりとしたヒストグラムができるか試してみましょう。
先程の例のテンプレートクエリから，主にハイライトの部分（sampleはその内容も）だけを変更すればよいようになっています。

```sql
with sample AS
(
  SELECT member_id, SUM(price*amount) AS val
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  HAVING SUM(price*amount) <= 1000000
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx, 20 AS n
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

概ね5万円区切りの区間になっているのは，年間購買額の上限を10万円までに限定しているからです。

参考までに，年間購買額の上限（10万円）を設けずにヒストグラムを描いてみます。ちなみに，一番大きい値は「161925600」（1.6億）です。どういったヒストグラムになるでしょうか。

```sql
with sample AS
(
  SELECT member_id, SUM(price*amount) AS val
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  --HAVING SUM(price*amount) <= 1000000
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx, 20 AS n
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

結果は，バケット番号1にほとんどの値が集中してしまう結果となってしまいました。これではヒストグラムどころではありません。
この理由は単純で，「96〜161925600」の年間購買額の範囲が広く，かつほとんどの人の年間購買額は100万円以下であるにもかかわらず，約800万円の幅（=(161925600-96)/20）で分割してしまっているからです。この例から得られる学びは，以下のような試みはうまくいかないということです。
- 最小と最大の幅（範囲）がとても広いデータで，
- しかもデータに偏り（たいていははじめの区間に集中）があるものに対して，
- 大きい区間幅（=少ないbinの数）でヒストグラムを作ろうとする
単純な対応策は，先程のように上限を切ってしまうことです。上限を定めたくない場合には，binの数をとても多く（1区間がおよそ10000になるように，ここではn = 16192）します。
```sql
with sample AS
(
  SELECT member_id, SUM(price*amount) AS val
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  --HAVING SUM(price*amount) <= 1000000
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx, 16192 AS n
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

今度は多くの区間に値が入るようになりました。ただ，このヒストグラムは右に大きく尾を引いてしまっています。そこで，x軸をある程度の値で切った状態にするのも1つの手です。以下の図はバケット番号50までで切ったものです。

## 両端の数%を外れ値とみなして除外したヒストグラム
両端の数%を外れ値として例外バケットに入れることを考えます。以下のクエリでは，左右のboundを5%分位点と95%分位点に設定し，その範囲外（左右に5%外れている値）は例外バケットに入れるようにしています。
```sql
with sample AS
(
  SELECT member_id, SUM(price*amount) AS val
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  --HAVING SUM(price*amount) <= 1000000
),
stat AS
 (
  SELECT 
    APPROX_PERCENTILE(val,0.05) AS mn, 
    APPROX_PERCENTILE(val,0.95) AS mx,
    20 AS n
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

右に長く尾を引いたヒストグラムだったものが，上図のように途中から先が例外区間としてまとめられたことで，ある程度すっきりした外観になりました。外れバケットだけ値の範囲が大きくなっているので厳密にはヒストグラムとはいえませんが，むしろ可視性に優れているので，実用面では問題ありません。

## スタージェスの公式
ヒストグラムのbinの数（または幅）の決定方法に最適解はありません。ただし，レコード数の増加に伴って緩やかにbinの数が増えていくようなヒストグラムを描くための「スタージェスの公式」と呼ばれるものが存在します。スタージェスの公式では，binの数Nは以下の公式で与えられます。
```sql
N=log2(レコード数)-1
```sql
これに基づくと，binの幅は「MAX(値) - MIN(値) / N」とすることができます。この公式に基づいてある程度見やすい幅に修正したヒストグラムを作成するためのクエリを以下に示します。
```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,7,7,7,8,8,9 ) AS t(val)
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx,
  CAST(LOG2(COUNT(1))-1 AS INTEGER) AS n -- スタージェスの公式によるbinの数
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```
スタージェスの公式には，binの数（幅も）が自動的に決定されるので作成時に何も考えなくてよいという利点があります。レコード件数が増えてきても下記のようにbinの数が高々20程度に抑えられることも利点です。
- 100万件のbin数：	=log2(1000000)-1=19.931-1≒19
- 1000万件のbin数：	=log2(10000000)-1=23.253-1≒22 
- 1億件のbin数：	=log2(100000000)-1=26.575-1≒25
ただし，これだけの件数があれば相当なばらつき（最小と最大の幅）もあるはずで，それをこの程度のbinの数で表現することに無理がある（もっとたくさんのbinの数が必要である）場合もあるでしょう。

### 年間購買分布
スタージェスの公式では，実は年間購買分布はうまく見られません。前述の問題が如実に現れてくるからです。
- データの最小：96
- データの最大：161925600
- スタージェスの公式によるbinの数：12
- よってbinの幅：約1350万円
データの幅の割に，binの幅12は圧倒的に小さいため，次のような結果となります。
```sql
with sample AS
(
  SELECT member_id, SUM(price*amount) AS val
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  --HAVING SUM(price*amount) <= 1000000
),
stat AS
 (
  SELECT MIN(val) AS mn, MAX(val)+1 AS mx,
  CAST(LOG2(COUNT(1))-1 AS INTEGER) AS n -- スタージェスの公式によるbinの数
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```
「1.34938881E7」などの末尾の「E7」は10の7乗の意味です。2番目の区間が1.34938881 * 10^7 ≒ 1350万円以上なので，はじめの区間にほとんどの人が入ってしまっています。
とはいえ，両端の数%を外れ値とみなせばそれなりのヒストグラムになり，しかもこちらでbinの数を指定する必要もない汎用的なテンプレートクエリとなります。

```sql
with sample AS
(
  SELECT member_id, SUM(price*amount) AS val
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  --HAVING SUM(price*amount) <= 1000000
),
stat AS
 (
  SELECT 
    APPROX_PERCENTILE(val,0.05) AS mn, 
    APPROX_PERCENTILE(val,0.95) AS mx,
    CAST(LOG2(COUNT(1))-1 AS INTEGER) AS n -- スタージェスの公式によるbinの数
  FROM sample
),
bucket_table AS
(
  SELECT val, WIDTH_BUCKET(val,mn,mx,n) AS bucket, COUNT(1) AS freq
  FROM sample,stat
  GROUP BY val, WIDTH_BUCKET(val,mn,mx,n)
),
range_bucket_table AS
(
  SELECT val, bucket, 
    IF(bucket=0,  -INFINITY(),mn+1.0*(mx-mn)/n*(bucket-1)) AS lower_val,
    IF(bucket=n+1, INFINITY(),mn+1.0*(mx-mn)/n*bucket)     AS upper_val,
    freq
  FROM
  (
    SELECT val, bucket, freq
    FROM bucket_table

    UNION ALL
    SELECT NULL AS val, time AS bucket, 0 AS freq
    FROM serial_numbers
    WHERE time < (SELECT MAX(bucket) FROM bucket_table)
    AND time NOT IN (SELECT bucket FROM bucket_table)
  ), stat
)

SELECT bucket,
  CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')') AS range, 
  SUM(freq) AS freq
FROM range_bucket_table
GROUP BY bucket, CONCAT('[',CAST(ROUND(lower_val,1) AS VARCHAR),',',CAST(ROUND(upper_val,1) AS VARCHAR),')')
ORDER BY bucket
```

## NUMERIC_HISTOGRAM
NUMERIC_HISTOGRAMは，binの数を与えるだけで自動的にヒストグラムに必要なデータ（binの区間決定と集計）を作成してくれる大変便利な関数です。結果はMAPで返ってくるので，テーブル形式への展開が必要となります。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,4,4,4,4,5,5,5,5,5,6,6,6,6,7,7,7,8,8,9 ) AS t(val)
),
histo AS (
  SELECT NUMERIC_HISTOGRAM(5,val) AS res_map
  FROM sample
)

SELECT key, value
FROM histo
CROSS JOIN UNNEST (
  res_map
) t2 (key, value)
```
返ってくるのはbinの区間ではなく，中心値であることに注意してください。

### 年間購買分布
NUMERIC_HISTOGRAMなら，binの数を指定するだけで，具体例でも簡単にヒストグラムが作れます。クエリは前述のHISTOGRAMのときとほとんど変わりません。

```sql
with histo AS
(
  SELECT NUMERIC_HISTOGRAM(20,sales) AS res_map
  FROM
  (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
)

SELECT key, value
FROM histo
CROSS JOIN UNNEST (
  res_map
) t2 (key, value)
ORDER BY key
```

