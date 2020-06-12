# Lesson 20. EC ログ分析クエリ

## Predictive Scoring（1ヶ月後の退会予測）
Lesson.19のアクティビティの推移における，sample_accesslogにおけるアクティビティセグメントをsales_slipにも同様に適用します。このログへ適用した場合，継続や休眠，新規というセグメントはそれぞれ（次の月において）継続購入，購入中断，新規購入というように「アクセス」ではなく「購入」についてのアクティビティセグメントになることに注意してください。
今回は以下の3つのセグメントに着目します：
- 新規→継続（新規購入者が次月も購入してくれた）
- 新規→休眠（新規購入者が次月は購入してくれなかった）
- 新規（新規購入者，１ヶ月後には継続？休眠？）
今，最後の新規というのは次月になると上2つの継続か休眠の上のどちらかになります。数ヶ月や数年に渡ってこのデータを蓄積していけば，新規のユーザーが次にどちら（継続か休眠か）になるのかを，次月の結果を待たずに予測できるかもしれません。また，もし精度良く予測ができるならば，休眠可能性のある新規購入者に対してプロモーションを行うことができます。
今回の目的はSQLではなく，Predictive Scoringの紹介にありますのでクエリの細かな説明は割愛します。また，サンプルデータのため予測精度は低いことをご了承ください。

紹介するPredictive Scoringでは，各「新規」購入者の，次月において「新規→休眠」となる確率（0-100のスコア）を求めます。上のScore Distributionにおけるサマリ画面では予測対象者のスコアの分布を表示しています。今回は75%を超えるような確率の高い購入者は見つけられませんでした。

### スコアリング用データの作成
以下のクエリをTD_SCHEDULED_TIMEの日付を'2005-03-01'から'2006-02-01'まで1月ずつ動かしながら12回繰り返します。これによってsales_slipにおけるアクティビティセグメントの内訳をactivity_ecテーブルに12ヶ月分追記していることになります。
```sql
/* ログの範囲 [2004-12-24, 2013-12-19]
/* TD_SCHEDULED_TIME() = '2005-03-01'〜'2006-02-01' */
CREATE TABLE IF NOT EXISTS activity_ec(time bigint);
INSERT INTO activity_ec
WITH before_1month AS
  (
  SELECT member_id
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-10y/-1M', 'JST') /* 1ヶ月前より過去 */
  AND member_id IS NOT NULL
  GROUP BY member_id
),
before_2month AS
(
  SELECT member_id
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-10y/-2M', 'JST') /* 2ヶ月前より過去 */
  AND member_id IS NOT NULL
  GROUP BY member_id
),
past_1month AS
(
  SELECT member_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-1M', 'JST') /* 前月1ヶ月間 */
  AND member_id IS NOT NULL
  GROUP BY member_id
),
past_2month AS
(
  SELECT member_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-1M/-1M', 'JST') /* 前々月1ヶ月間 */
  AND member_id IS NOT NULL
  GROUP BY member_id
),
activity_past_1month AS
(
  SELECT
    IF(b1.member_id IS NOT NULL, b1.member_id, p1.member_id) AS member_id,
    CASE 
      WHEN b1.member_id IS NOT NULL AND p1.member_id IS NOT NULL THEN 'active'
      WHEN b1.member_id IS NOT NULL AND p1.member_id IS     NULL THEN 'non_active'
      WHEN b1.member_id IS     NULL AND p1.member_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p1.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p1.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_1month b1
  FULL OUTER JOIN past_1month p1
  ON b1.member_id = p1.member_id
),
activity_past_2month AS
(
  SELECT
    IF(b2.member_id IS NOT NULL, b2.member_id, p2.member_id) AS member_id,
    CASE 
      WHEN b2.member_id IS NOT NULL AND p2.member_id IS NOT NULL THEN 'active'
      WHEN b2.member_id IS NOT NULL AND p2.member_id IS     NULL THEN 'non_active'
      WHEN b2.member_id IS     NULL AND p2.member_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p2.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p2.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_2month b2
  FULL OUTER JOIN past_2month p2
  ON b2.member_id = p2.member_id
),
activity_users AS
(
  SELECT 
    TD_TIME_FORMAT(TD_DATE_TRUNC('month', TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(),'JST')-1, 'JST'),'yyyy-MM-dd','JST') AS target_month,
    TD_TIME_FORMAT(
      TD_DATE_TRUNC( 'month',
        TD_DATE_TRUNC('month', TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(),'JST')-1, 'JST')-1,'JST'),'yyyy-MM-dd','JST') AS target_pre_month,
    activity_past_1month.member_id, 
    activity_past_1month.segment AS activity_past_1,
    activity_past_2month.segment AS activity_past_2,
    activity_past_1month.min_time AS min_time_past_1, activity_past_1month.max_time AS max_time_past_1,
    activity_past_2month.min_time AS min_time_past_2, activity_past_2month.max_time AS max_time_past_2
  FROM activity_past_1month
  LEFT OUTER JOIN activity_past_2month /* 集合 activity_past_1month は activity_past_2month を内包する */
  ON activity_past_1month.member_id = activity_past_2month.member_id
),
master_activity AS
(
  SELECT activity_past_1,activity_past_2,segment_name
  FROM
  ( VALUES
    ('active','active','継続'),
    ('active','non_active','休眠→復活'),
    ('active','welcome','新規→継続'),
    ('non_active','active','継続→休眠'),
    ('non_active','non_active','休眠'),
    ('non_active','welcome','新規→休眠'),
    ('welcome',NULL,'新規')
  ) AS t(activity_past_1,activity_past_2,segment_name) 
)

SELECT target_month, target_pre_month, member_id, segment_name
FROM activity_users a
JOIN master_activity m
ON a.activity_past_1 = m.activity_past_1 AND a.activity_past_2 IS NOT DISTINCT FROM m.activity_past_2
```

12ヶ月分のレコードが入っているか，以下のクエリで確認してみましょう。

```sql
SELECT target_month, COUNT(1) AS cnt
FROM activity_ec
GROUP BY target_month
ORDER BY target_month DESC
```
ここまでで各月の各ユーザーにアクティビティセグメントを付与することができましたので，その他の属性情報をmaster_membersから，及び行動履歴情報をsales_slipから取得し付与します。
```sql
DROP TABLE IF EXISTS sample_activity_ec;
CREATE TABLE sample_activity_ec AS
WITH user_monthly_stat AS
(
  SELECT 
    TD_TIME_FORMAT(time,'yyyy-MM-01','JST') AS m, 
    member_id, COUNT(1) AS freq,
    SUM(amount) AS amount,
    SUM(price*amount) AS sales,
    SUM(IF(category='Movies and Music and Games',price*amount,0))     AS sales_movie,
    SUM(IF(category='Sports and Outdoors',price*amount,0))            AS sales_sports,
    SUM(IF(category='Toys and Kids and Baby',price*amount,0))         AS sales_toys,
    SUM(IF(category='Books and Audible',price*amount,0))              AS sales_books,
    SUM(IF(category='Home and Garden and Tools',price*amount,0))      AS sales_home,
    SUM(IF(category='Clothing and Shoes and Jewelry',price*amount,0)) AS sales_cloting,
    SUM(IF(category='Beauty and Health and Grocery',price*amount,0))  AS sales_beauty,
    SUM(IF(category='Electronics and Computers',price*amount,0))      AS sales_electronics,
    SUM(IF(category='Automotive and Industrial',price*amount,0))      AS sales_automotive
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2005-01-01','2006-04-01','JST')
  GROUP BY TD_TIME_FORMAT(time,'yyyy-MM-01','JST'), member_id
)

SELECT segment_name, stat.*
FROM activity_ec activity
LEFT OUTER JOIN user_monthly_stat stat
ON activity.target_pre_month = stat.m AND activity.member_id = stat.member_id
/* 前々月のstatが存在するセグメントは前々月のstatを採用 */
WHERE segment_name NOT IN ('休眠→復活','新規')
  
UNION ALL
SELECT segment_name, stat.*
FROM activity_ec activity
LEFT OUTER JOIN user_monthly_stat stat
ON activity.target_month = stat.m AND activity.member_id = stat.member_id
/* 前々月のstatが存在しないセグメントは前月のstatを採用 */
WHERE segment_name IN ('休眠→復活','新規')
```

## Audience Stidio でセグメントの作成
今回は（Audience Studioにおける）Master Segmentに先程作成したsample_activity_ecテーブルを指定し，Predictive Scoringのための学習対象セグメント，予測セグメント，ポジティブセグメントを作成していきます。Attribute Tableにはmaster_membersを指定し，member_idをJoin Keyとしてage,carrier,gender,prefecture,town,city,marriageを引っ張ってきます。

このMaster Segmentを「Activity Segment（EC）」と名付け，実行します。

### 学習対象セグメント
今回，学習対象となるセグメントは「新規→継続」または「新規→休眠」のいずれかになります。この2つの条件を「any」で連結します。このセグメントを「新規→（継続，休眠）」として保存します。

### 予測セグメント
予測対象となるセグメントは「新規」なのですが，最新の「新規」ユーザーのみ抽出します。なぜなら，過去で新規であったユーザーもその次の月で継続か休眠か既に判明しているからです。最も直近の月の新規のみがその先にどちらになるか判明していないのです。今回はカラムmの値が「2016-01-01」の新規ユーザー（131人）が予測対象となります。今後の条件の連結は「and」になることに注意してください。このセグメントを「新規」として保存します。

### ポジティブセグメント
ポジティブセグメントは，学習の上で「1」とみなすセグメントで，今回は「新規→休眠」となります。この時，設定はしませんが「休眠→継続」は「0」として学習され，予測対象のセグメントはそのロジスティック回帰の結果として0から1の間の確率（結果の表示はパーセントによる0-100表示）となり，1に近い予測結果は「新規→休眠」に近いユーザーであるという解釈ができます。このセグメントを「新規→休眠」として保存します。

## Predictive Scoring
次に，Predictive Scoringにて先程のセグメントの設定および，予測のための変数として使用するカラムを指定します。初めは以下を参照にしてみてください。
- Categorycal Features
  - gender, marriage, carrier, prefecture
- Quantitative Features
  - freq, amount, sales, sales_*, age
この設定で「休眠予測（EC）」という名前で保存し，Trainを実行しましょう。

### 予測結果
Trainの実行が完了しましたら，再度Master Segmentを更新（Run）する事を忘れないでください。その後，Predictive Scoring画面の「Predictive Score」タブが表示できるようになります。

この結果，スコアの高いユーザーは休眠になる可能性が高いことになるので対策が必要です。

## テストによるQuantitative Featuresの選定
ここでは予測の際に選定するQuantitative Featuresをどう選べば良いかについて一つの指針を与えることにします。
Quantitative Featuresの1つ1つは，A/Bテストによって（A：「新規→継続」，B：「新規→休眠」として）その平均値に差があるかどうかで採用するか否かを決めることができます。なぜなら予測の変数となるものは，AとBを明確に分離できる（=統計的差がある）ものであることが望ましいからです。
もちろん，予測には複数の変数の複合的要因も絡んできますので話はそこまで単純ではないのですが，Quantitative Featuresの選択には苦痛を伴う作業だと思いますので，全ての変数をテストして「REJECT（統計的に差異があると認められる）」されるものだけ選ぶというのも一つの手段です。
ここでは1例として，salesというQuantitative Featuresを採用すべきかどうかを考えていきます。
A/Bテストのテンプレートにこのsalesの平均値を当てはめます。
```sql
WITH ab_table AS
(
  SELECT 
    CASE segment_name
      WHEN '新規→継続' THEN 'A'
      WHEN '新規→休眠' THEN 'B'
    END AS ab,
    COUNT(1) AS cnt,
    AVG(sales)    AS ag,
    STDDEV(sales) AS sd
  FROM sample_activity_ec
  GROUP BY     
    CASE segment_name
      WHEN '新規→継続' THEN 'A'
      WHEN '新規→休眠' THEN 'B'
    END
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

結果は，Aの平均購買額17824円はBの12410円と差があることがありました。なのでこの変数は予測対象の変数として組み込むことにします。

## デシル分析
デシル分析は，何らかの指標を10等分割し，それぞれの分割に入る人数や平均値を見比べる手法です。
### 人数で等分割
期間を固定して（例えば直近から2年間における）ユーザーごとの購入総額が大きい順に並べ，それを人数で等分割してみましょう。それぞれの分割ごとの平均購入額がどれだけ違うかを見てみます。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, COUNT(1) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales
    FROM sales_slip
    WHERE TD_INTERVAL(time, '-730d', 'JST')
    AND member_id IS NOT NULL
    GROUP BY member_id
  )
),
sales_table AS(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
), 
tile_table AS
(
  SELECT tile, COUNT(1) AS cnt, AVG(two_years_sales) AS sales_avg, 
    1.0*SUM(two_years_sales)/MAX(total_sales) AS sales_ratio
  FROM
  (
    SELECT NTILE(10)OVER(ORDER BY two_years_sales DESC) AS tile, member_id, two_years_sales
    FROM sales_table
  ), stat
  GROUP BY tile
)

SELECT tile, cnt, sales_avg, sales_ratio,
  SUM(sales_ratio)OVER(ORDER BY tile) AS cum_sales_ratio
FROM tile_table
ORDER BY tile
```
購入額が一番多いタイル1と一番少ないタイル10を比較すれば，平均購入額92万円と2万円で大きな差が出ています。sales_ratioでは，そのタイルの人が全体の総購入額の何%を占めているかを計算しています。以下の円グラフはその内訳を表したものです。このグラフは，購入額の多いたった20%（タイル1と2）のユーザーで，全体の売上の半分以上を占めているという事実を表しています。

### 購入比率で等分割
全員の総購入額を均等に10分割し，購入額の大きい人から順に積み上げていくと，総購入額の10%までに何人が入るか，さらに次の20%までに何人が入るかを見ていきます。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, COUNT(1) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales
    FROM sales_slip
    WHERE TD_INTERVAL(time, '-730d', 'JST')
    AND member_id IS NOT NULL
    GROUP BY member_id
  )
),
sales_table AS
(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
),
tile_table AS
(
  SELECT 
    CEIL( 1.0*SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC)/total_sales*10 ) AS tile, member_id, two_years_sales
  FROM sales_table, stat
)

SELECT tile, COUNT(1) AS cnt, AVG(two_years_sales) AS sales_avg, SUM(two_years_sales) AS sales_sum
FROM tile_table
GROUP BY tile
ORDER BY tile
```
この結果からは，ロイヤル（購入額の大きい）なユーザーが112人で全体の売上の10%を占めること（平均200万円層），次のロイヤルが241人であること（平均100万円層）がわかります。最も少額購入のユーザー層の3212人が束になって，やっと最上位の112人と同じ売上総額になります。

### 売上積み上げグラフ
もっと単純に，総購入額が多いユーザーからの積み上げグラフを作ってみるとわかりやすいかもしれません。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, COUNT(1) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales
    FROM sales_slip
    WHERE TD_INTERVAL(time, '-730d', 'JST')    
    AND member_id IS NOT NULL
    GROUP BY member_id
  )
),
sales_table AS
(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')  
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT member_id, 
  RANK()OVER(ORDER BY two_years_sales DESC) AS rnk,
  two_years_sales, 
  SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC) AS cum_sales,
  1.0*SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC)/total_sales AS cum_ratio
FROM sales_table, stat
ORDER BY cum_ratio
```

このような積み上げで見れば，全体の売上の25%がたった600人程度のロイヤルユーザーによって占められていることがよくわかります。もしこのグラフが直線に近ければロイヤルなユーザーは少なく，前半が急激に膨らんでいればいるほど，とても大きな購入額の少数ユーザーが存在するということが示唆されます。

## RFM分析
### Recency
Recencyは，ユーザーを直近の「購入日」によって区分する方法です。Recencyの意味でのユーザーの良さは，直近で購入のあったユーザーほど「高く」，何年も前に購入がさかのぼるユーザーほど「低い」と見ることができます。Recencyは，いわばユーザーの「新鮮度」で，Recencyの高いユーザーほど次のイベントへの感度や新製品の購入意欲が高いものと判断できます。
Recencyでは， Rから始まるもう1つの指標であるRegistration Day（登録日）も考慮することが重要です。はるか昔に登録してから直近も購入してくれている息の長いユーザーは，最近登録したユーザーよりも貴重だと考えられます。直近登録のユーザーが，より時間が経ったときも常にRecencyが高くいられるかはわかりません。そういう意味で，同時に登録日も求めておくことが非常に有用です。
「直近」を考える際，現在進行系で蓄積されているデータに対しては今日または昨日でかまわないのですが，もっと過去のデータでは「直近」がいつかを定める必要があります。これまで何度か登場しているTD_SCHEDULED_TIME()関数を使うことで，実行ごとに「直近」の値を定めることができます。
以下のクエリでは，recent_daysはユーザーごとの（直近以前からの）最新購入日（最新を1として何日前か），registration_daysは一番最初の購入日（初購入の意味での登録日が何日前か）としています。sales_periodは購入期間を意味し，これらの差を意味します。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH recency_inner_table AS
(
  SELECT member_id, 
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MAX(time))/(60*60*24))+1 AS recent_days,
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MIN(time))/(60*60*24))+1 AS registration_days
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, NULL, TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST'), 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  member_id,
  RANK()OVER(ORDER BY recent_days ASC, registration_days DESC) AS rnk,
  recent_days,
  registration_days, 
  registration_days-recent_days+1 AS sales_period
FROM recency_inner_table
ORDER BY recent_days ASC, registration_days DESC
```

### Frequency
どれくらい頻繁に購入してくれているかをFrequencyとして，頻度が高いユーザーほど良いと考えます。Frequencyは，いわば「常連度」で，そのECサイトへの愛情を測る度数となります。
Frequencyでは，ユーザーの過去すべて（ライフタイム）における以下の値を考えます。
- 総購入回数
- それを会員期間でならした月間平均購入回数
ただ，期間を固定して例えば「直近から2年間での総購入回数」を考えれば，平均を考える必要はありません。
以下のクエリでは，期間固定（直近2年間）の総購入回数を求めています。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH frequency_inner_table AS
(
  SELECT member_id, 
    COUNT(1) AS two_years_cnt
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST')  
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  RANK()OVER(ORDER BY two_years_cnt DESC) AS rnk,
  member_id,
  two_years_cnt
FROM frequency_inner_table
ORDER BY two_years_cnt DESC
```
上記のヒストグラムでは，極端に頻度が大きい上位1人を除いています。

## Monetary
購入総額を示すMonetaryは，それが高いユーザーほど良いと考えられる，非常にダイレクトな指標です。Monetaryは，いわばECサイト運営サイドへの「貢献度」であり，これが高いユーザーは決して邪険にしてはいけません。購入総額上位の数%のユーザーだけで，全体の購入の20〜50%を占めるようなことも稀ではありません。そういったユーザーは「クジラ（ホエール）」と呼ばれます。
Monetaryでは，ユーザーの過去すべて（ライフタイム）における以下の値を考えます。
- 購入総額
- それを会員期間でならした月間平均購入額
ただ，期間を固定して例えば「直近から2年間での購入総額」を考えれば，平均を考える必要はありません。以下のクエリでは期間固定のMonetaryを求めています。
```sql
--TD_SCHEDULED_TIME()=2013-12-19
WITH monetary_table AS
(
  SELECT member_id, 
    SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST') 
  AND member_id IS NOT NULL
  GROUP BY member_id
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  RANK()OVER(ORDER BY two_years_sales DESC) AS rnk,
  member_id,
  two_years_sales
FROM monetary_table
ORDER BY two_years_sales DESC
```
上記のヒストグラムでは，極端に頻度が大きい上位1人を除いています。

## RFM 
下記は，先程まで別個で求めていたRFMをユーザーごとに同時に求めるクエリです。FrequencyとMonetaryの指標では，期間固定2年間の集計値を扱います。
また，この結果を「rfm_result」として一度テーブルに書き込んで置きましょう。
```sql
DROP TABLE IF EXISTS rfm_result;
CREATE TABLE rfm_result AS
--TD_SCHEDULED_TIME()=2013-12-19
WITH recency_inner_table AS
(
  SELECT member_id, 
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MAX(time))/(60*60*24))+1 AS recent_days,
    FLOOR(1.0*(TD_SCHEDULED_TIME()-MIN(time))/(60*60*24))+1 AS registration_days
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, NULL, TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST'), 'JST')
  AND member_id IS NOT NULL
  GROUP BY member_id
),
recency_table AS
(
  SELECT member_id,
    RANK()OVER(ORDER BY recent_days ASC, registration_days DESC) AS rnk_recency,
    recent_days,
    registration_days, 
    registration_days-recent_days+1 AS sales_period
  FROM recency_inner_table
),
frequency_inner_table AS
(
  SELECT member_id, COUNT(1) AS two_years_cnt
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST') 
  AND member_id IS NOT NULL
  GROUP BY member_id
),
frequency_table AS
(
  SELECT RANK()OVER(ORDER BY two_years_cnt DESC) AS rnk_frequency, member_id, two_years_cnt
  FROM frequency_inner_table
),
monetary_inner_table AS
(
  SELECT member_id, SUM(price*amount) AS two_years_sales
  FROM sales_slip
  WHERE TD_INTERVAL(time, '-730d', 'JST') 
  AND member_id IS NOT NULL
  GROUP BY member_id
),
monetary_table AS
(
  SELECT RANK()OVER(ORDER BY two_years_sales DESC) AS rnk_monetary, member_id, two_years_sales
  FROM monetary_inner_table
)

SELECT 
  TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, 
  r.member_id, registration_days, sales_period,
  recent_days     AS recency,
  two_years_cnt   AS frequency,
  two_years_sales AS monetary,
  rnk_recency, rnk_frequency, rnk_monetary
FROM recency_table r 
JOIN frequency_table f
ON r.member_id = f.member_id
JOIN monetary_table m
ON r.member_id = m.member_id
```

さて，各々のユーザーのRFMの値を同時に求めるだけならば，上記のクエリでおしまいです。しかしながら，RFM分析の本質は，RFMそれぞれを適切なグループに割り振ることにあります。RFM分析では，そのうち2つを縦横にしたクロス表を見ていきます。そのためのグルーピングとして最も単純な方法は，人間にとってわかりやすい区分を使うものです。

```sql
WITH recency_segment AS
(
  SELECT member_id,
    CASE 
      WHEN recency <=   7 THEN '1. 1week'
      WHEN recency <=  14 THEN '2. 2week'
      WHEN recency <=  30 THEN '3. 1month'
      WHEN recency <=  90 THEN '4. 3month'
      WHEN recency <= 365 THEN '5. 1year'
      WHEN 365 < recency  THEN '6. <1year'
      ELSE 'error'
    END AS seg_recency
  FROM rfm_result
),
frequency_segment AS
(
  SELECT member_id,
    CASE --(times*week)*(month)*(year)
      WHEN (3*4)*12*2 <= frequency THEN '1. 3/week'
      WHEN (2*4)*12*2 <= frequency THEN '2. 2/week'
      WHEN (1*4)*12*2 <= frequency THEN '3. 1/week'
      WHEN (2)  *12*2 <= frequency THEN '4. 2/month'
      WHEN (1)  *12*2 <= frequency THEN '5. 1/month'
      WHEN frequency < 12*2        THEN '6. 1/year'
      ELSE 'error'
    END AS seg_frequency
  FROM rfm_result
),
monetary_segment AS
(
  SELECT member_id,
    CASE --(money)*(month)*(year)
      WHEN ( 15000)*12*2 <= monetary THEN  '1. 15000/month'
      WHEN ( 10000)*12*2 <= monetary THEN  '2. 10000/month'
      WHEN (  5000)*12*2 <= monetary THEN  '3. 5000/month'
      WHEN (  2500)*12*2 <= monetary THEN  '4. 1000/month'
      WHEN (  1000)*12*2 <= monetary THEN  '5. 1000/month'
      WHEN monetary < (  1000)*12*2  THEN  '6. <1000/month'
      ELSE 'error'
    END AS seg_monetary
  FROM rfm_result
)
```
上記のような人数区分のグルーピングを行います。
以降では，上記のような人数区分のグルーピングを行います。
### RFマトリクス
下記のクエリはRFマトリクス（RとFのクロステーブル）を求めるものです。RMマトリクス，FM マトリクス，RFM マトリクスに必要なテーブルも掲載しています。
```sql
WITH recency_segment AS
(
  SELECT member_id,
    CASE 
      WHEN recency <=   7 THEN '1. 1week'
      WHEN recency <=  14 THEN '2. 2week'
      WHEN recency <=  30 THEN '3. 1month'
      WHEN recency <=  90 THEN '4. 3month'
      WHEN recency <= 365 THEN '5. 1year'
      WHEN 365 < recency  THEN '6. <1year'
      ELSE 'error'
    END AS seg_recency
  FROM rfm_result
),
frequency_segment AS
(
  SELECT member_id,
    CASE --(times*week)*(month)*(year)
      WHEN (3*4)*12*2 <= frequency THEN '1. 3/week'
      WHEN (2*4)*12*2 <= frequency THEN '2. 2/week'
      WHEN (1*4)*12*2 <= frequency THEN '3. 1/week'
      WHEN (2)  *12*2 <= frequency THEN '4. 2/month'
      WHEN (1)  *12*2 <= frequency THEN '5. 1/month'
      WHEN frequency < 12*2        THEN '6. 1/year'
      ELSE 'error'
    END AS seg_frequency
  FROM rfm_result
),
monetary_segment AS
(
  SELECT member_id,
    CASE --(money)*(month)*(year)
      WHEN ( 15000)*12*2 <= monetary THEN  '1. 15000/month'
      WHEN ( 10000)*12*2 <= monetary THEN  '2. 10000/month'
      WHEN (  5000)*12*2 <= monetary THEN  '3. 5000/month'
      WHEN (  2500)*12*2 <= monetary THEN  '4. 2500/month'
      WHEN (  1000)*12*2 <= monetary THEN  '5. 1000/month'
      WHEN monetary < (  1000)*12*2  THEN  '6. <1000/month'
      ELSE 'error'
    END AS seg_monetary
  FROM rfm_result
),
rf_table AS
(
  SELECT seg_recency, seg_frequency, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency
    FROM recency_segment, frequency_segment
    WHERE recency_segment.member_id = frequency_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency
),
rm_table AS
(
  SELECT seg_recency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_monetary
    FROM recency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_monetary
),
fm_table AS
(
  SELECT seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT frequency_segment.member_id, seg_frequency, seg_monetary
    FROM frequency_segment, monetary_segment
    WHERE frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_frequency, seg_monetary
),
rfm_table AS
(
  SELECT seg_recency,seg_frequency, seg_monetary, COUNT(1) AS cnt
  FROM
  (
    SELECT recency_segment.member_id, seg_recency, seg_frequency, seg_monetary
    FROM recency_segment, frequency_segment, monetary_segment
    WHERE recency_segment.member_id = monetary_segment.member_id
    AND frequency_segment.member_id = monetary_segment.member_id
  )
  GROUP BY seg_recency, seg_frequency, seg_monetary
)

SELECT seg_recency,
  IF(ELEMENT_AT(kv,'1. 3/week' ) IS NOT NULL, kv['1. 3/week'],  0) AS c1_3week,
  IF(ELEMENT_AT(kv,'2. 2/week' ) IS NOT NULL, kv['2. 2/week'],  0) AS c2_2week,
  IF(ELEMENT_AT(kv,'3. 1/week' ) IS NOT NULL, kv['3. 1/week'],  0) AS c3_1week,
  IF(ELEMENT_AT(kv,'4. 2/month') IS NOT NULL, kv['4. 2/month'], 0) AS c4_2month,
  IF(ELEMENT_AT(kv,'5. 1/month') IS NOT NULL, kv['5. 1/month'], 0) AS c5_1month,
  IF(ELEMENT_AT(kv,'6. 1/year' ) IS NOT NULL, kv['6. 1/year'],  0) AS c6_1year
FROM
(
  SELECT seg_recency, MAP_AGG(seg_frequency, cnt) AS kv
  FROM rf_table
  GROUP BY seg_recency
)
ORDER BY seg_recency
```

### RMマトリクス
```sql
...
SELECT seg_recency,
  IF(ELEMENT_AT(kv,'1. 15000/month') IS NOT NULL, kv['1. 15000/month'], 0) AS c1_15000,
  IF(ELEMENT_AT(kv,'2. 10000/month') IS NOT NULL, kv['2. 10000/month'], 0) AS c2_10000,
  IF(ELEMENT_AT(kv,'3. 5000/month')  IS NOT NULL, kv['3. 5000/month'],  0) AS c3_5000,
  IF(ELEMENT_AT(kv,'4. 2500/month')  IS NOT NULL, kv['4. 2500/month'],  0) AS c4_2500,
  IF(ELEMENT_AT(kv,'5. 1000/month')  IS NOT NULL, kv['5. 1000/month'],  0) AS c5_1000,
  IF(ELEMENT_AT(kv,'6. <1000/month') IS NOT NULL, kv['6. <1000/month'], 0) AS c6_0
FROM
(
  SELECT seg_recency, MAP_AGG(seg_monetary, cnt) AS kv
  FROM rm_table
  GROUP BY seg_recency
)
ORDER BY seg_recency
```

### FMマトリクス
```sql
...
SELECT seg_frequency,
  IF(ELEMENT_AT(kv,'1. 15000/month') IS NOT NULL, kv['1. 15000/month'], 0) AS c1_15000,
  IF(ELEMENT_AT(kv,'2. 10000/month') IS NOT NULL, kv['2. 10000/month'], 0) AS c2_10000,
  IF(ELEMENT_AT(kv,'3. 5000/month')  IS NOT NULL, kv['3. 5000/month'],  0) AS c3_5000,
  IF(ELEMENT_AT(kv,'4. 2500/month')  IS NOT NULL, kv['4. 2500/month'],  0) AS c4_2500,
  IF(ELEMENT_AT(kv,'5. 1000/month')  IS NOT NULL, kv['5. 1000/month'],  0) AS c5_1000,
  IF(ELEMENT_AT(kv,'6. <1000/month') IS NOT NULL, kv['6. <1000/month'], 0) AS c6_0
FROM
(
  SELECT seg_frequency, MAP_AGG(seg_monetary, cnt) AS kv
  FROM fm_table
  GROUP BY seg_frequency
)
ORDER BY seg_frequency
```

### RFMマトリクス（R固定）
```sql
SELECT seg_recency, seg_frequency,
  IF(ELEMENT_AT(kv,'1. 15000/month') IS NOT NULL, kv['1. 15000/month'], 0) AS c1_15000,
  IF(ELEMENT_AT(kv,'2. 10000/month') IS NOT NULL, kv['2. 10000/month'], 0) AS c2_10000,
  IF(ELEMENT_AT(kv,'3. 5000/month')  IS NOT NULL, kv['3. 5000/month'],  0) AS c3_5000,
  IF(ELEMENT_AT(kv,'4. 2500/month')  IS NOT NULL, kv['4. 2500/month'],  0) AS c4_2500,
  IF(ELEMENT_AT(kv,'5. 1000/month')  IS NOT NULL, kv['5. 1000/month'],  0) AS c5_1000,
  IF(ELEMENT_AT(kv,'6. <1000/month') IS NOT NULL, kv['6. <1000/month'], 0) AS c6_0
FROM
(
  SELECT seg_recency, seg_frequency, MAP_AGG(seg_monetary, cnt) AS kv
  FROM rfm_table
  GROUP BY seg_recency, seg_frequency
)
ORDER BY seg_recency, seg_frequency
```


### FMマトリクス（比率セグメント）
FMについても，デシル分析で見たMonetaryの売上比率を固定したセグメントと同じものを作ってみます。まず，Monetaryについては売上比率を5分割し，最上位の売上20%に入るユーザーを購入額の多い上位から入れていき，それを最もロイヤル度の高いセグメント1とします。次の売上20%に入るユーザーをセグメント2，その次を3として，5までのセグメントを作ります。
Frequencyについても同じ考え方で進め，全体の総頻度に対して頻度が高いほうから20%に入るユーザーをセグメント1とします。ただし，1人だけ圧倒的に頻度の高いユーザー（2259091）が含まれているので，このユーザーは除外して集計します。
```sql
WITH stat AS
(
  SELECT SUM(two_years_sales) AS total_sales, SUM(two_years_cnt) AS total_cnt
  FROM
  (
    SELECT member_id, SUM(price*amount) AS two_years_sales, COUNT(1) AS two_years_cnt
    FROM sales_slip
    WHERE TD_TIME_RANGE(time, TD_TIME_ADD(TD_SCHEDULED_TIME(),'-730d'), TD_SCHEDULED_TIME(), 'JST')
    AND member_id IS NOT NULL AND member_id <> '2259091'
    GROUP BY member_id
  )
),
fm_table AS
(
  SELECT ftile, mtile, COUNT(1) AS cnt, 
    AVG(two_years_cnt) AS cnt_avg, AVG(two_years_sales) AS sales_avg, 
    SUM(two_years_cnt) AS cnt_sum, SUM(two_years_sales) AS sales_sum
  FROM
  (
    SELECT member_id, two_years_cnt, two_years_sales,
      CEIL( 1.0*SUM(two_years_cnt)  OVER(ORDER BY two_years_cnt   DESC)/total_cnt*5 )   AS ftile, 
      CEIL( 1.0*SUM(two_years_sales)OVER(ORDER BY two_years_sales DESC)/total_sales*5 ) AS mtile
    FROM
    (
      SELECT member_id, SUM(price*amount) AS two_years_sales, COUNT(1) AS two_years_cnt
      FROM sales_slip
      WHERE TD_TIME_RANGE(time, TD_TIME_ADD(TD_SCHEDULED_TIME(),'-730d'), TD_SCHEDULED_TIME(), 'JST')
      AND member_id IS NOT NULL AND member_id <> '2259091'
      GROUP BY member_id
    ), stat
  )
  GROUP BY ftile, mtile
)

SELECT CAST(mtile AS INTEGER) AS mtile, mtotal,
  IF(ELEMENT_AT(kv,1) IS NOT NULL, kv[1], 0) AS f1,
  IF(ELEMENT_AT(kv,2) IS NOT NULL, kv[2], 0) AS f2,
  IF(ELEMENT_AT(kv,3) IS NOT NULL, kv[3], 0) AS f3,
  IF(ELEMENT_AT(kv,4) IS NOT NULL, kv[4], 0) AS f4,
  IF(ELEMENT_AT(kv,5) IS NOT NULL, kv[5], 0) AS f5
FROM
(
  SELECT mtile, SUM(cnt) AS mtotal, MAP_AGG(ftile, cnt) AS kv
  FROM fm_table
  GROUP BY mtile
)
ORDER BY mtile
```
上記の結果では，行にmonetaryセグメント，列にfrequencyセグメントとしています。mtotalはそれぞれのmonetaryセグメントの合計人数です。「monetaryセグメント=1，frequencyセグメント=1」の237人は，売上総額および頻度ともに全体の20%の割合を占める上位層です。その右の71人は，frequencyにおいては次の20%に入るユーザーとなります。
対角に位置するセグメントペアは，monetaryとfrequencyが同じセグメント，つまり（全体の割合の意味で）ほぼ同じ順位のユーザーで，この対角上にほとんどの数字が入ってしまう場合は，monetaryとfrequencyの関係が強いことを示しています。
最後におまけとして，monetaryとfrequency（セグメントではなく集計値そのもの）同士の相関を調べてみましょう。
```sql
SELECT CORR(two_years_cnt,two_years_sales) AS cor
FROM
(
  SELECT member_id, SUM(price*amount) AS two_years_sales, COUNT(1) AS two_years_cnt
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, TD_TIME_ADD(TD_SCHEDULED_TIME(),'-730d'), TD_SCHEDULED_TIME(), 'JST')
  AND member_id IS NOT NULL AND member_id <> '2259091'
  GROUP BY member_id
)
```
