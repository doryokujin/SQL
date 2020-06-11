# Lesson 16. A/B テスト

トレジャーデータ上では，クエリは複雑になるものの，A/Bテストも実践できます。複雑なクエリも，テンプレートにすると思えばそれほど苦ではないでしょう。A/Bテストにはたくさんの手法があり，様々な条件と仮定（母分散が異なるか否かなど）があるため，ここでは理論面の解説は省き，母平均の比較（Welchの片側検定（有意水準0.05））を行う実践可能なクエリと実例を紹介するのみにとどめます。

A/Bテストの概念をきちんと理解するには，統計についての勉強も必要です。参考までに，ここでは筆者の過去のブログ記事だけ紹介します（「A/Bテストの数理 - 第3回：テストの基本的概念と結果の解釈方法について -」）。以下の思考フレームワークなどの説明も，難しければ最初は読み飛ばしてもらってかまいません。

## 統計的検定の思考フレームワーク

1. 棄却を目的として，帰無仮説をこしらえる
2. 帰無仮説のもと，標本からある分布（正規分布，t分布，F分布，2分布など）に従う統計量Tを求める
3. 統計量Tがその分布のもとで稀なケースかどうかをチェックする
4. 稀なケースであり，帰無仮説を疑うほうが妥当であると判断できる場合は帰無仮説を棄却し，対立仮説を採択する。そうでないなら何もわからないと判断する

3.で「稀なケース」と判断する手段について，p値の役割と共に説明します。
p値とは，「帰無仮説のもとで実際にデータから計算される統計量よりも極端な（大きな）統計量が観測される確率」で表される値です。例えば，p値が0.05のときは，「あなたが計算した計算量Tが，（仮定する）分布から得られる確率は0.05である」と述べていることになります。テストのスタンスは，帰無仮説を棄却することです。つまり，p値が小さければ小さいほど，Tがこの分布に従っているという帰無仮説をより強く疑うことができます。
そのうえで「稀なケース」と判断する場合とは，両側検定であればp値が0.025より小さいとき（有意水準0.05の両側検定の場合），片側検定であればp値が0.05より小さいとき（有意水準0.05の片側検定の場合）です。

統計量Tにおけるp値は，Tが標準正規分布に従う場合，下図に示す標準正規分布の白色の面積で，左右0.025ずつあります。この区間から値が得られることは「非常に稀」と考えられます。標準正規分布においては，T=1.96のときにp値が0.025となるので，Tが1.96より大きければ（あるいは反対側でTが-1.96より小さければ），この分布において「稀なケース」と判断します。

また下図のt分布では，自由度nによって分布の形が異なり，自由度が無限大で上の標準正規分布と同じになります。特に自由度が小さいところでのt分布は平べったく裾野の広い分布となり，例えば自由度が1においてはT=12.706のときにp値が0.025となるので，「稀なケース」と判断されにくくなり，帰無仮説を棄却できずにテストが何も言えずで終わります。t分布の自由度はサンプル数が増えると増加するので，サンプル数が多い場合の方がTの値が小さくても棄却しやすくなり，テストが機能してきます。
 
## Welch の片側検定
テキストを参照ください。

## テストに必要なインプットテーブル形式
まず，A/Bテストにかけるためのテーブル形式を定義します。
- レコードはAとBの2行
- AとBの平均値（平均アクセス数，平均売上，平均コンバージョン）が統計的な意味で差異があるか否かを検定する
- A，Bそれぞれの「数」，「平均」，「分散」を求めたテーブルがテンプレート

## 購買履歴での事例
具体的な例で見ていきましょう。以下は，（あまり比較の意味がないですが）sales_slipのmember_idの偶数と奇数でAとBに分け，2011年における平均購入額に差異があるのかを検定
するクエリです。
```sql
SELECT 
  IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
  COUNT(1) AS cnt,
  AVG(sales)    AS ag,
  STDDEV(sales) AS sd
FROM (
  SELECT member_id, SUM(price*amount) AS sales
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
)
GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
```

これから4421人のAグループと4408人のBグループで，165,884円と204,696円の平均購入額に統計的に差異があるのかを検定します。16万と20万の差は割と大きいと思われますがどうでしょうか？ インプットテーブルがこの形式であれば，後はテンプレートクエリによってテストの結果を返すことができます。
まず，この基本統計値から後でTの計算に使う値をueachカラムとして準備しておきます。

```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1) AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
)

SELECT *, (cnt/(cnt-1))*sd*sd AS ueach
FROM ab_table
```

## A/Bレコードのマージ
次に，AとBのレコードのカラム値同士を計算できるようにするために，この2行のレコードを1行のレコードにマージします。
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1) AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
)

SELECT 
  ab    AS a,       LEAD(ab)OVER(ORDER BY ab)    AS b,
  cnt   AS cnt_a,   LEAD(cnt)OVER(ORDER BY ab)   AS cnt_b,
  ag    AS avg_a,   LEAD(ag)OVER(ORDER BY ab)    AS avg_b,
  sd    AS sd_a,    LEAD(sd)OVER(ORDER BY ab)    AS sd_b,
  ueach AS ueach_a, LEAD(ueach)OVER(ORDER BY ab) AS ueach_b
FROM
(
  SELECT *, (cnt/(cnt-1))*sd*sd AS ueach
  FROM ab_table
)
```

この結果，2番目のレコードは必要なくなるので，次のクエリでは必要な1行目だけを抽出するようにします。

## 統計値Tと自由度m
重要な統計値T（t_stat）と自由度mを求めます。結果は，いったんwelchテーブルとして持っておきます。
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1) AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
),
ueach_table AS
(
  SELECT 
    ab    AS a,       LEAD(ab)OVER(ORDER BY ab)    AS b,
    cnt   AS cnt_a,   LEAD(cnt)OVER(ORDER BY ab)   AS cnt_b,
    ag    AS avg_a,   LEAD(ag)OVER(ORDER BY ab)    AS avg_b,
    sd    AS sd_a,    LEAD(sd)OVER(ORDER BY ab)    AS sd_b,
    ueach AS ueach_a, LEAD(ueach)OVER(ORDER BY ab) AS ueach_b
  FROM
  (
    SELECT *, (cnt/(cnt-1))*sd*sd AS ueach
    FROM ab_table
  )
)

SELECT *,
  ABS( (avg_a-avg_b)/SQRT(sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)) ) AS t_stat,
  ROUND( POW((sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)),2) / ( POW(sd_a,4)/POW(cnt_a-1,3) + POW(sd_b,4)/POW(cnt_b-1,3) ) ) AS m
FROM ueach_table
WHERE a = 'A'
```

## テストテンプレート
テーブルとして持っているt分布テーブルの該当する自由度の値を抽出し，先程求めたwelchテーブルのt_statと比較して，その大小で検定をRejectする（平均が等しいという仮設を棄却＝平均に差異があることを認める）かどうかを判定します。これでテンプレートクエリの完成です。
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1) AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
),
ueach_table AS
(
  SELECT 
    ab    AS a,       LEAD(ab)OVER(ORDER BY ab)    AS b,
    cnt   AS cnt_a,   LEAD(cnt)OVER(ORDER BY ab)   AS cnt_b,
    ag    AS avg_a,   LEAD(ag)OVER(ORDER BY ab)    AS avg_b,
    sd    AS sd_a,    LEAD(sd)OVER(ORDER BY ab)    AS sd_b,
    ueach AS ueach_a, LEAD(ueach)OVER(ORDER BY ab) AS ueach_b
  FROM
  (
    SELECT *, (cnt/(cnt-1))*sd*sd AS ueach
    FROM ab_table
  )
),
welch AS
(
  SELECT *,
    ABS( (avg_a-avg_b)/SQRT(sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)) ) AS t_stat,
    ROUND( POW((sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)),2) / ( POW(sd_a,4)/POW(cnt_a-1,3) + POW(sd_b,4)/POW(cnt_b-1,3) ) ) AS m
  FROM ueach_table
  WHERE a = 'A'
),
t AS
(
  SELECT MIN_BY(t_dist.val,t_dist.m) AS t_val
  FROM welch, t_dist
  WHERE welch.m <= t_dist.m
)

SELECT cnt_a, cnt_b, ROUND(avg_a) AS avg_a, ROUND(avg_b) AS avg_b, 
  t_stat, t_val, IF(t_stat>t_val,'REJECT','NOT REJECT') AS test_res
FROM welch, t
```
上記のテスト結果は「NOT REJECT：差異が認められない」となりました。

## サンプル数と標準偏差が変化するとテストの結果はどう変わるか？

先ほどは，
1. （サンプル数）A：4400人，B：4400人
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：220000，B：2450000
```sql
t_stat = 1.05 < 1.96 = t_val ⇒ 「NOT REJECT」
```sql
という結果でした。下図はこの状況を可視化したものです。

ここで，テストの知見を深めるために以下の考察を行いましょう。

1. A,B 双方のサンプル数がもっと多い/少ない場合，テストの結果はどう変わるか？
2. Aのサンプル数がBに対して圧倒的に少ない場合，テストの結果はどう変わるか？
3. A,B 双方の標準偏差がもっと大きい/小さい場合，テストの結果はどう変わるか？
4. Aの標準偏差がBに対して圧倒的に大きい場合，テストの結果はどう変わるか？
これらをその他の条件は変えずに検証（t_statとt_valueの値とその位置関係がどう変わるか？）したいと思います。

### 1-a. A,B 双方のサンプル数がもっと多い場合
まず，サンプル数がもっと多い場合を先に考えます。
1. （サンプル数）A：440000人，B：440000人 ←100倍にしてみる
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：220000，B：2450000
テストテンプレートの一部を人工的に変更して結果を確認してみます。
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1) * 100 AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
),
...
```
この場合，t_statの値が10倍ほど大きくなり，t_valを超えて棄却される（差があるとみなす）結果となりました。サンプル数が大きくなればなるほど，今まで棄却できなかった（差があると言えなかった）ものが棄却されやすくなります。これは，サンプル数が大きくなればテストの自信が増すことを意味しています。また本来はt_valの値もより小さくなるのですが，元のサンプル数も十分に大きいため，ほとんど変化が見えない結果となっています。

### 1-b. A,B 双方のサンプル数がもっと少ない場合
次に，サンプル数がずっと少ない場合を先に考えます。
1. （サンプル数）A：44人，B：44人 ←1/100にしてみる
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：220000，B：2450000
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1) / 100 AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
),
...
```
この場合，先程とは逆に，t_statの値が1/10ほどになり，t_valは大きくなりました。結果的に棄却域よりも大きく離れることになり，棄却できない結果となりました。

### 2. Aのサンプル数がBに対して圧倒的に少ない場合
1. （サンプル数）A：44人，B：4400人 ←Aを1/100にしてみる
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：220000，B：2450000

```sql
WITH ab_table AS
(
  SELECT ab, IF(ab='A',cnt/100, cnt) AS cnt, ag, sd
  FROM
  (
    SELECT 
      IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
      COUNT(1) AS cnt,
      AVG(sales)    AS ag,
      STDDEV(sales) AS sd
    FROM (
      SELECT member_id, SUM(price*amount) AS sales
      FROM sales_slip
      WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
      GROUP BY member_id
    )
    GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
  )
),
...
```

この場合，t_statが元よりもそれなりに小さな値となり，t_valも若干大きくなりました（自由度mは208）。これによって，t_statが棄却域から遠ざかることになり，棄却できない結果となりました。次にAとBの極端さをさらに大きくしてみます。

1. （サンプル数）A：4人，B：440000人 ←Aを1/1000，Bを100倍
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：220000，B：2450000

この場合，t_statの値は元よりも遥かに小さくなり，特筆すべきはt_valが遥かに大きくなった（t分布の自由度は3!）ことです。これは片方のサンプルが極端に小さければそちらに影響され，他方がどれだけ大きくてもほとんど差異を判定できないテストとなってしまうことを示しています。

### 3-a. A,B 双方の標準偏差がもっと大きい場合

1. （サンプル数）A：4400人，B：4400人
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：22000000，B：245000000 ←100倍にしてみる
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1)      AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) * 100 AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
),
...
```
標準偏差が大きくなると，t_statの方に圧倒的に小さくなり，ほとんど棄却できない状態となります。標準偏差（分散）が大きい平均値は，サンプルのとる値に大きなブレがあるということなので，テストの自信をなくす方向に働くことになります。また，標準偏差を変えてもt_valには大きな影響はありません。
### 3-b. A,B 双方の標準偏差がもっと小さい場合
1. （サンプル数）A：4400人，B：4400人
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：2200，B：24500 ←1/100にしてみる
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    COUNT(1)      AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) / 100 AS sd
  FROM (
    SELECT member_id, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
),
...
```
標準偏差が大きくなると，t_statの方に圧倒的に大きくなり，棄却しやすい状態になります。これは，標準偏差の小さいサンプルからの平均値は信頼できるものなので，16.6万円と20.5万円でははっきりと差異があると言えることを意味しています。
### 4. Aの標準偏差がBに対して圧倒的に大きい場合，テストの結果はどう変わるか？
1. （サンプル数）A：4400人，B：4400人
2. （平均購入額）A：16.6万円，B：20.5万円
3. （標準偏差）A：22000000，B：24500 ←Aを100倍，Bを1/100にしてみる
```sql
WITH ab_table AS
(
  SELECT ab, cnt, ag, IF(ab='A',sd*100, sd/100) AS sd
  FROM
  (
    SELECT 
      IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
      COUNT(1) AS cnt,
      AVG(sales)    AS ag,
      STDDEV(sales) AS sd
    FROM (
      SELECT member_id, SUM(price*amount) AS sales
      FROM sales_slip
      WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
      GROUP BY member_id
    )
    GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B')
  )
),
...
```
この結果は，片方の標準偏差がどれだけ小さくても（信頼できても），他方のそれが圧倒的に大きい場合にはt_statが遥かに小さくなり，全く棄却できないテストとなってしまうことを示しています。

## GROUP BYに対応したテンプレートクエリ
このテンプレートクエリでは，GROUP BYによるカテゴリやURLごとのテストを同時に行うことができます。以下の例では，sub_categoryごとに同様のテストを行っています。テンプレート内のGROUP BY節，PARTITION BY節，SELECT節などの該当箇所にsub_categoryを追加します。
```sql
WITH ab_table AS
(
  SELECT 
    IF(CAST(member_id AS INTEGER)%2=0,'A','B') AS ab, 
    sub_category,
    COUNT(1) AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM (
    SELECT member_id, sub_category, SUM(price*amount) AS sales
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
    GROUP BY member_id, sub_category
  )
  GROUP BY IF(CAST(member_id AS INTEGER)%2=0,'A','B'), sub_category
),
ueach_table AS
(
  SELECT 
    ab    AS a,       LEAD(ab)OVER(PARTITION BY sub_category ORDER BY ab)    AS b,
    sub_category,
    cnt   AS cnt_a,   LEAD(cnt)OVER(PARTITION BY sub_category ORDER BY ab)   AS cnt_b,
    ag    AS avg_a,   LEAD(ag)OVER(PARTITION BY sub_category ORDER BY ab)    AS avg_b,
    sd    AS sd_a,    LEAD(sd)OVER(PARTITION BY sub_category ORDER BY ab)    AS sd_b,
    ueach AS ueach_a, LEAD(ueach)OVER(PARTITION BY sub_category ORDER BY ab) AS ueach_b
  FROM
  (
    SELECT *, (cnt/(cnt-1))*sd*sd AS ueach
    FROM ab_table
  )
),
welch AS
(
  SELECT *,
    ABS( (avg_a-avg_b)/SQRT(sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)) ) AS t_stat,
    ROUND( POW((sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)),2) / ( POW(sd_a,4)/POW(cnt_a-1,3) + POW(sd_b,4)/POW(cnt_b-1,3) ) ) AS m
  FROM ueach_table
  WHERE a = 'A'
),
t AS
(
  SELECT sub_category, MIN_BY(t_dist.val,t_dist.m) AS t_val
  FROM welch, t_dist
  WHERE welch.m <= t_dist.m
  GROUP BY sub_category
)

SELECT welch.sub_category, cnt_a, cnt_b, ROUND(avg_a) AS avg_a, ROUND(avg_b) AS avg_b, 
  t_stat, t_val, IF(t_stat>t_val,'REJECT','NOT REJECT') AS test_res
FROM welch JOIN t
ON welch.sub_category = t.sub_category
ORDER BY t_stat DESC
```
上の結果からは，例えば以下のようなことがわかります。
- Pet SuppliesのA：5110円とB：4572円には差異がある（REJECT）
- Hunting and FishingのA：4874円とB：5352円には差異が認められない（NOT REJECT）
今回のテストで差異が認められたのは3件だけでしたが，これはそもそもA/Bの分類が適当だったことに起因します。また，この判定は単純な平均値の近さだけでなく，各々のレコード数と分散を考慮した結果になっていることに注意してください。

## アクセスログでの事例
最後に，sample_accesslogにおいてtd_hostごとにその長さの偶数奇数でAとBに分け，平均PVを計算し，td_titleごとにテストする例を紹介します。A，Bともに50人に満たないtd_titleは対象外としています。今回の平均PVでは小数点第1位まで表示しています。
```sql
WITH ab_table AS
(
  SELECT 
    td_title,
    IF(LENGTH(td_ip)%2=0,'A','B') AS ab, 
    COUNT(1) AS cnt,
    AVG(pv)    AS ag,
    STDDEV(pv) AS sd
  FROM (
    SELECT td_ip, td_title, COUNT(1) AS pv
    FROM sample_accesslog
    GROUP BY td_ip, td_title
  )
  GROUP BY IF(LENGTH(td_ip)%2=0,'A','B'), td_title
),
ueach_table AS
(
  SELECT 
    ab    AS a,       LEAD(ab)OVER(PARTITION BY td_title ORDER BY ab)    AS b,
    td_title,
    cnt   AS cnt_a,   LEAD(cnt)OVER(PARTITION BY td_title ORDER BY ab)   AS cnt_b,
    ag    AS avg_a,   LEAD(ag)OVER(PARTITION BY td_title ORDER BY ab)    AS avg_b,
    sd    AS sd_a,    LEAD(sd)OVER(PARTITION BY td_title ORDER BY ab)    AS sd_b,
    ueach AS ueach_a, LEAD(ueach)OVER(PARTITION BY td_title ORDER BY ab) AS ueach_b
  FROM
  (
    SELECT *, (cnt/(cnt-1))*sd*sd AS ueach
    FROM ab_table
  )
),
welch AS
(
  SELECT *,
    ABS( (avg_a-avg_b)/SQRT(sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)) ) AS t_stat,
    ROUND( POW((sd_a*sd_a/(cnt_a-1)+sd_b*sd_b/(cnt_b-1)),2) / ( POW(sd_a,4)/POW(cnt_a-1,3) + POW(sd_b,4)/POW(cnt_b-1,3) ) ) AS m
  FROM ueach_table
  WHERE a = 'A'
),
t AS
(
  SELECT td_title, MIN_BY(t_dist.val,t_dist.m) AS t_val
  FROM welch, t_dist
  WHERE welch.m <= t_dist.m
  GROUP BY td_title
)

SELECT welch.td_title, cnt_a, cnt_b, ROUND(avg_a,1) AS avg_a, ROUND(avg_b,1) AS avg_b, 
  t_stat, t_val, IF(t_stat>t_val,'REJECT','NOT REJECT') AS test_res
FROM welch JOIN t
ON welch.td_title = t.td_title
WHERE 50 <= cnt_a AND 50 <= cnt_b
ORDER BY t_stat DESC
```

この例では，すべて差異が認められない結果となりました。
