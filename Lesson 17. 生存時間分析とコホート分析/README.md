# Lesson 17. 生存時間分析とコホート分析

## コホート分析
ユーザーの人数を加入月ごとに集計し，さらにそれぞれの加入月のユーザーのうち1ヶ月後，2ヶ月後，…にアクセスしたユーザーのユニークな人数を求めます。基本的には月を追うごとにユニーク数は減っていく傾向にありますが，その減っていく割合を加入月別に比較できるようなマトリクスを描いたのがコホートテーブルです。
```sql
WITH stat AS
(
  SELECT 
    uid, TD_TIME_FORMAT(MIN(time), 'yyyy-MM', 'JST') AS first_login
  FROM login
  GROUP BY uid
)

SELECT m, first_login, COUNT(1) AS cnt
FROM
(
  SELECT
    uid, TD_TIME_FORMAT(time, 'yyyy-MM', 'JST') AS m
  FROM login
  GROUP BY uid, TD_TIME_FORMAT(time, 'yyyy-MM', 'JST')
) login
JOIN
(
  SELECT uid, first_login FROM stat
) stat
ON login.uid = stat.uid
GROUP BY m, first_login
ORDER BY m, first_login
```
この結果をコホートテーブルとして表示すると，対角と左半分のみに値が存在するマトリクスになります。カラム（加入月）ごとに下に追っていけば，翌月以降のユニークな（生存）人数が見えます。行（該当月）ごとに見れば，その月にアクセスのあったユーザーの構成（何月に登録したか）がわかります。

上記のコホートテーブルは，Googleスプレッドシートのピボットテーブルによって作成しました。SQLのみで Pivot を行っても同じものが作れます。
```sql
WITH stat AS
(
  SELECT 
    uid, TD_TIME_FORMAT(MIN(time), 'yyyy-MM', 'JST') AS first_login
  FROM login
  GROUP BY uid
),
login_table AS
(
  SELECT
    uid, TD_TIME_FORMAT(time, 'yyyy-MM', 'JST') AS m
  FROM login
  GROUP BY uid, TD_TIME_FORMAT(time, 'yyyy-MM', 'JST')
),
cohort_table AS
(
  SELECT m, first_login, COUNT(1) AS cnt
  FROM login_table login JOIN stat
  ON login.uid = stat.uid
  GROUP BY m, first_login
)

SELECT m,
  IF(ELEMENT_AT(kv,'2011-11') IS NOT NULL, kv['2011-11'], 0) AS m2011_11,
  IF(ELEMENT_AT(kv,'2011-12') IS NOT NULL, kv['2011-12'], 0) AS m2011_12,
  IF(ELEMENT_AT(kv,'2012-01') IS NOT NULL, kv['2012-01'], 0) AS m2012_01,
  IF(ELEMENT_AT(kv,'2012-02') IS NOT NULL, kv['2012-02'], 0) AS m2012_02,
  IF(ELEMENT_AT(kv,'2012-03') IS NOT NULL, kv['2012-03'], 0) AS m2012_03,
  IF(ELEMENT_AT(kv,'2012-04') IS NOT NULL, kv['2012-04'], 0) AS m2012_04,
  IF(ELEMENT_AT(kv,'2012-05') IS NOT NULL, kv['2012-05'], 0) AS m2012_05,
  IF(ELEMENT_AT(kv,'2012-06') IS NOT NULL, kv['2012-06'], 0) AS m2012_06,
  IF(ELEMENT_AT(kv,'2012-07') IS NOT NULL, kv['2012-07'], 0) AS m2012_07,
  IF(ELEMENT_AT(kv,'2012-08') IS NOT NULL, kv['2012-08'], 0) AS m2012_08,
  IF(ELEMENT_AT(kv,'2012-09') IS NOT NULL, kv['2012-09'], 0) AS m2012_09,
  IF(ELEMENT_AT(kv,'2012-10') IS NOT NULL, kv['2012-10'], 0) AS m2012_10,
  IF(ELEMENT_AT(kv,'2012-11') IS NOT NULL, kv['2012-11'], 0) AS m2012_11,
  IF(ELEMENT_AT(kv,'2012-12') IS NOT NULL, kv['2012-12'], 0) AS m2012_12  
FROM
(
  SELECT m, MAP_AGG(first_login, cnt) AS kv
  FROM cohort_table
  GROUP BY m
)
ORDER BY m ASC
```
加入月ごとにその後のアクセス人数を追って色ごとにプロットすると，下記のようなコホートチャートが得られます。

## 継続期間
下記のクエリは，ユーザーのインストール日（初回アクセス日）から最終アクセス日までのプレイ日数（継続期間）を求めます。
```sql
WITH histo AS
(
  SELECT HISTOGRAM(term) AS res_map
  FROM
  (
    SELECT uid, 
     FLOOR( 1.0*(MAX(time)-MIN(time))/(60*60*24) )+1 AS term
    FROM login
    GROUP BY uid
  )
)

SELECT key, value
FROM histo
CROSS JOIN UNNEST (
  res_map
) t2 (key, value)
ORDER BY key
```
この結果はなかなか興味深いですね。インストール初日でドロップしまうユーザーがたくさんいて，続いて日が浅い順に人数が多くなっています。また，継続日数が極端に長い人がいることもわかります。これは，Webに公開された日から現時点まで続いているような熱狂的継続ユーザーです。

## 生存時間集計（単純集計）
継続期間を見るには，先程のような期間の分布ではなく，時間を追うごとにどれくらいの人数が減っていくかを階段チャートで見るほうが効果的です。先程のクエリを少し書き換えてみましょう。

```sql
WITH term_table AS
(
  SELECT term, COUNT(1) AS cnt
  FROM
  (
    SELECT uid, FLOOR( 1.0*(MAX(time)-MIN(time))/(60*60*24) )+1 AS term
    FROM login  
    GROUP BY uid
  )
  GROUP BY term
  UNION ALL
  SELECT term, cnt
  FROM ( VALUES (0,0) ) AS t(term, cnt)
),
stats AS
(
  SELECT SUM(cnt) AS cnt_total
  FROM term_table
)

SELECT 
  term,
    cnt_total-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS survival_cnt,
  1.0*(cnt_total-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))/cnt_total  AS survival_ratio
FROM term_table, stats
ORDER BY term
```

初期状態を見るために，term_tableのUNION ALLによって継続週0のレコードを接続しています。また，statsテーブルで対象の全体人数を求めています。
継続週0では全人数cnt_totalがいて，1週間後には継続期間が1週間だった人数だけがいなくなるので，前の値からその継続期間の人数を減算していけば，その週における生存人数を求めることができます。さらに，全体人数からの比を求めれば，その時点での生存率（らしき値）が求められます。

上記のグラフからは，1日目で生存ユーザーが半分近くにドロップし，さらに日が浅いうちは日を追うごとに徐々にユーザーが減っていき，次第にそれが緩やかになっていっていることが一目でわかります。なお，継続日が最大に近いところでも大きくドロップしているように見えますが，これは現時点での観測できる最大の日数であり，より観測日数が伸びればここも急激なドロップではなく緩やかな減少になっているはずです。
ところで，この例は観測期間が長いので，集計を週単位でやり直してみましょう（1日で辞めるユーザーは特別なので週単位で丸めるのは不本意ではありますが）。クエリの変更点は，termlカラムの分母を1週間の秒数に変えるだけです。

```sql
WITH term_table AS
(
  SELECT term, COUNT(1) AS cnt
  FROM
  (
    SELECT uid, FLOOR( 1.0*(MAX(time)-MIN(time))/(60*60*24*7) )+1 AS term
    FROM login  
    GROUP BY uid
  )
  GROUP BY term
  UNION ALL
  SELECT term, cnt
  FROM ( VALUES (0,0) ) AS t(term, cnt)
),
stats AS
(
  SELECT SUM(cnt) AS cnt_total
  FROM term_table
)

SELECT 
  term,
    cnt_total-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS survival_cnt,
  1.0*(cnt_total-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))/cnt_total  AS survival_ratio
FROM term_table, stats
ORDER BY term
```

## 生存時間分析
先程までの例で見た生存期間で対象となるユーザーの中には，観測時点（集計実行時点）で既に退会してしまっているユーザー（これを「打ち切り」ユーザーと呼びます）もいれば，継続中のユーザーもいます。打ち切りユーザーの継続期間「3」は時間が経っても変わりませんが，継続ユーザーは，同じ継続期間「3」でも時間が経てば継続が伸びて「4」以上になる可能性があります。打ち切りと継続を区別，加味したうえで生存期間を出すことができれば，はるかに信頼度の高い分析ができるでしょう。生存時間分析は，その要望を叶える高度な統計手法です。

退会ユーザーを調べるため，今回は現時点（今の例ではログに2012/12/27までしか入っていないので，厳密にはこれが「現時点」です）までの2週間以内にアクセスしていないユーザーを退会とみなし，is_cancelled = 1とします。そして，各継続期間（週，play_term_from_install）で退会ユーザーの人数cancel_cntと現役ユーザーの人数censored_cnt（censoredは打ち切りという意味で，観測時点ではまだ退会していないので将来の退会を確認することなく打ち切られたという意味）を求めます。ここまでがtermsテーブルの内容になります。

次に，「リスクを負っている人数」を求めます。これは，前の期間での生存人数として算出されるので，LAG関数で前の値を取得してきます。
```sql
WITH term_table AS
(
  SELECT uid, FLOOR( 1.0*(MAX(time)-MIN(time))/(60*60*24*7) )+1 AS term,
    IF( MAX(time)<TD_TIME_ADD(TD_SCHEDULED_TIME(),'-2w','JST'),1,0 ) AS is_cancelled --TD_SCHEDULED_TIME()=2012-12-27
  FROM login  
  GROUP BY uid
),
term_cnt_table AS
(
  SELECT term,
    COUNT(1) AS cnt,
    COUNT(IF(is_cancelled=1,1,NULL)) AS cancel_cnt,
    COUNT(IF(is_cancelled=0,1,NULL)) AS censored_cnt
  FROM term_table
  GROUP BY term
  UNION ALL
  SELECT term, cnt, cancel_cnt, censored_cnt
  FROM ( VALUES (0,0,0,0) ) AS t(term,cnt,cancel_cnt,censored_cnt)
),
stats AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM term_cnt_table
)

SELECT term, cancel_cnt, censored_cnt, 
  LAG(survival_cnt,1,survival_cnt)OVER(ORDER BY term) AS including_risk_cnt,
  1.0*survival_cnt/total_cnt  AS survival_ratio
FROM
(
  SELECT *, total_cnt-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS survival_cnt
  FROM term_cnt_table, stats
)
```
この結果テーブルをinput_tableとします。この結果から，インデックスi（ここでは行番号+1）に対するカプランマイヤー推定量とネルソンアーレン推定量という推定量Stを計算します。（詳細はテキストをご参照ください。）これらの推定量が，時間tにおける生存率を表しています。

```sql
WITH term_table AS
(
  SELECT uid, FLOOR( 1.0*(MAX(time)-MIN(time))/(60*60*24*7) )+1 AS term,
    IF( MAX(time)<TD_TIME_ADD(TD_SCHEDULED_TIME(),'-2w','JST'),1,0 ) AS is_cancelled --TD_SCHEDULED_TIME()=2012-12-27
  FROM login  
  GROUP BY uid
),
term_cnt_table AS
(
  SELECT term,
    COUNT(1) AS cnt,
    COUNT(IF(is_cancelled=1,1,NULL)) AS cancel_cnt,
    COUNT(IF(is_cancelled=0,1,NULL)) AS censored_cnt
  FROM term_table
  GROUP BY term
  UNION ALL
  SELECT term, cnt, cancel_cnt, censored_cnt
  FROM ( VALUES (0,0,0,0) ) AS t(term,cnt,cancel_cnt,censored_cnt)
),
stats AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM term_cnt_table
),
input_table AS
(
  SELECT term, cancel_cnt, censored_cnt, 
    LAG(survival_cnt,1,survival_cnt)OVER(ORDER BY term) AS including_risk_cnt,
    1.0*survival_cnt/total_cnt  AS survival_ratio
  FROM
  (
    SELECT *, total_cnt-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS survival_cnt
    FROM term_cnt_table, stats
  )
),
survival_stat AS
(
  SELECT *, 1-(1.0*cancel_cnt/including_risk_cnt) AS km_stat_seed, 1.0*cancel_cnt/including_risk_cnt AS nelson_aalen_stat_seed
  FROM input_table
)

SELECT *,
  EXP(SUM(LN(km_stat_seed))OVER(ORDER BY term ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)) AS km_stat,
  EXP(-SUM(nelson_aalen_stat_seed)OVER(ORDER BY term ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)) AS nelson_aalen_stat
FROM survival_stat
```

上図では，カプランマイヤー（積-極限）推定量km_statおよびネルソンアーレン推定量nelson_aalen_statと，先程の単純集計による生存率（survival_ratio）とを重ねて描いています。それぞれで結果が少し異なっていることがわかります。一般に，ネルソンアーレン推定量はカプランマイヤー推定量よりも小さな標本サイズでより良い実効性を持つ推定量として提案されています。

生存時間分析のクエリはやや複雑ですが，クエリテンプレートと割り切れば，変更すべき箇所は以下の5箇所のみです。
- term_tableのFROM節（読み込むデータ）
- ユーザー識別子「uid」の変更
- いつまでアクセスなければ退会とみなすかの「2w」の部分
- そして観測日をいつとみなすかの「TD_SCHEDULED_TIME()」の値
- termの単位（今回は「週単位」）の部分

### sample_access_log

sample_access_logの生存時間分析は以下のクエリで実行できます。
- term_tableのFROM節：「sample_access_log」
- ユーザー識別子：「td_client_id」
- いつまでアクセスなければ退会とみなすか：「1w」
- そして観測日をいつとみなすか：「TD_SCHEDULED_TIME() = 2016-07-05」
- termの単位の部分：「日単位」

```sql
WITH term_table AS
(
    SELECT td_client_id, FLOOR( 1.0*(MAX(time)-MIN(time))/(60*60*24) )+1 AS term,
      IF( MAX(time)<TD_TIME_ADD(TD_SCHEDULED_TIME(),'-1w','JST'),1,0 ) AS is_cancelled --TD_SCHEDULED_TIME()=2016-07-05
    FROM sample_accesslog
    GROUP BY td_client_id
),
term_cnt_table AS
(
  SELECT term,
    COUNT(1) AS cnt,
    COUNT(IF(is_cancelled=1,1,NULL)) AS cancel_cnt,
    COUNT(IF(is_cancelled=0,1,NULL)) AS censored_cnt
  FROM term_table
  GROUP BY term
  UNION ALL
  SELECT term, cnt, cancel_cnt, censored_cnt
  FROM ( VALUES (0,0,0,0) ) AS t(term,cnt,cancel_cnt,censored_cnt)
),
stats AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM term_cnt_table
),
input_table AS
(
  SELECT term, cancel_cnt, censored_cnt, LAG(survival_cnt,1,survival_cnt)OVER(ORDER BY term) AS including_risk_cnt,
    1.0*survival_cnt/total_cnt  AS survival_ratio
  FROM
  (
    SELECT *, total_cnt-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS survival_cnt
    FROM term_cnt_table, stats
  )
),
survival_stat AS
(
  SELECT *, 1-(1.0*cancel_cnt/including_risk_cnt) AS km_stat_seed, 1.0*cancel_cnt/including_risk_cnt AS nelson_aalen_stat_seed
  FROM input_table
)

SELECT *,
  EXP(SUM(LN(km_stat_seed))OVER(ORDER BY term ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)) AS km_stat,
  EXP(-SUM(nelson_aalen_stat_seed)OVER(ORDER BY term ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)) AS nelson_aalen_stat
FROM survival_stat
```
コーポレートサイトのような，継続的にはアクセスされないサイトのログに対しては，1日より長く継続してくれるユーザーは10%以下になってしまっています。

### sales_slip
sales_slipの生存時間分析は以下のクエリで実行できます。
- termsテーブルのFROM節：「sales_slip」
- ユーザー識別子：「member_id」
- いつまでアクセスなければ退会とみなすか：「1m」
- そして観測日をいつとみなすか：「TD_SCHEDULED_TIME() = 2013-12-19」
- play_term_from_installの単位の部分：「月単位」

```sql
WITH term_table AS
(
    SELECT member_id, FLOOR( 1.0*(MAX(time)-MIN(time))/(60*60*24*7*30) )+1 AS term,
      IF( MAX(time)<TD_TIME_ADD(TD_SCHEDULED_TIME(),'-1m','JST'),1,0 ) AS is_cancelled --TD_SCHEDULED_TIME()=2013-12-19
    FROM sales_slip
    GROUP BY member_id
),
term_cnt_table AS
(
  SELECT term,
    COUNT(1) AS cnt,
    COUNT(IF(is_cancelled=1,1,NULL)) AS cancel_cnt,
    COUNT(IF(is_cancelled=0,1,NULL)) AS censored_cnt
  FROM term_table
  GROUP BY term
  UNION ALL
  SELECT term, cnt, cancel_cnt, censored_cnt
  FROM ( VALUES (0,0,0,0) ) AS t(term,cnt,cancel_cnt,censored_cnt)
),
stats AS
(
  SELECT SUM(cnt) AS total_cnt
  FROM term_cnt_table
),
input_table AS
(
  SELECT term, cancel_cnt, censored_cnt, LAG(survival_cnt,1,survival_cnt)OVER(ORDER BY term) AS including_risk_cnt,
    1.0*survival_cnt/total_cnt  AS survival_ratio
  FROM
  (
    SELECT *, total_cnt-SUM(cnt)OVER(ORDER BY term RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS survival_cnt
    FROM term_cnt_table, stats
  )
),
survival_stat AS
(
  SELECT *, 1-(1.0*cancel_cnt/including_risk_cnt) AS km_stat_seed, 1.0*cancel_cnt/including_risk_cnt AS nelson_aalen_stat_seed
  FROM input_table
)

SELECT *,
  EXP(SUM(LN(km_stat_seed))OVER(ORDER BY term ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)) AS km_stat,
  EXP(-SUM(nelson_aalen_stat_seed)OVER(ORDER BY term ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)) AS nelson_aalen_stat
FROM survival_stat
```

ECサイトでは，はじめの数ヶ月で辞めてしまうユーザーこそ少ないですが，8ヶ月ほどで3/4になり，1年ほどで半分になり，14ヶ月ほどで1/4になります。後半にかけて急に生存率が下がっていくことが見てとれますね。
