# Lesson 18. ゲームログ分析クエリ

## Login KPI

### PV，UU（デイリー）
```sql
SELECT
  TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d,
  COUNT(1) AS pv,
  COUNT(DISTINCT uid) AS uu
FROM login
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
ORDER BY d ASC
```

### PV，UU（月次）
```sql
SELECT
  TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS m,
  COUNT(1) AS pv,
  COUNT(DISTINCT uid) AS uu
FROM login
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
ORDER BY m ASC
```

### PV，UU（月次×デイリーのピボット）
月と日でのPV，UUのピボットテーブルを，表計算ソフトのお世話になることなくSQLで作ってみましょう。月をカラムにするほうがカラムの数は少なくて済みますが，どう可視化したいかによるので，ここでは両方のケースを紹介します。
注意点を以下にまとめます。
- まずは月次と日次でGROUP BYした縦持ちテーブルを作る
- この例では年は固定とする。年を固定しておけば，月（’MM’）のみ，日（’dd’）のみ保持していればよい（「yyyy-MM」などと持たなくてよい）
- ピボットする際，キーが存在しない（ある月日の数値がない場合もある）場合に対処するため，ELEMENT_ATで要素チェックする
- 特にUUは，横方向を通したUUを別に計算しておいて一緒に見られると効果的

#### 1. 行：日，列：月，値：UU
```sql
WITH 
res_activity AS
(
  SELECT
    TD_TIME_FORMAT(time, 'MM', 'JST') AS m,
    TD_TIME_FORMAT(time, 'dd', 'JST') AS d,
    COUNT(1) AS pv,
    COUNT(DISTINCT uid) AS uu
  FROM login
  WHERE TD_TIME_RANGE(time, '2012-01-01','2013-01-01', 'JST')
  GROUP BY TD_TIME_FORMAT(time, 'MM', 'JST'), TD_TIME_FORMAT(time, 'dd', 'JST')
),
stat AS
(
  SELECT
    TD_TIME_FORMAT(time, 'dd', 'JST') AS d,
    COUNT(1) AS pv_total,
    COUNT(DISTINCT uid) AS uu_total
  FROM login
  WHERE TD_TIME_RANGE(time, '2012-01-01','2013-01-01', 'JST')
  GROUP BY TD_TIME_FORMAT(time, 'dd', 'JST')
)

SELECT d, uu_total,
  IF(ELEMENT_AT(kv,'01') IS NOT NULL, kv['01'], 0) AS m01,
  IF(ELEMENT_AT(kv,'02') IS NOT NULL, kv['02'], 0) AS m02,
  IF(ELEMENT_AT(kv,'03') IS NOT NULL, kv['03'], 0) AS m03,
  IF(ELEMENT_AT(kv,'04') IS NOT NULL, kv['04'], 0) AS m04,
  IF(ELEMENT_AT(kv,'05') IS NOT NULL, kv['05'], 0) AS m05,
  IF(ELEMENT_AT(kv,'06') IS NOT NULL, kv['06'], 0) AS m06,
  IF(ELEMENT_AT(kv,'07') IS NOT NULL, kv['07'], 0) AS m07,
  IF(ELEMENT_AT(kv,'08') IS NOT NULL, kv['08'], 0) AS m08,
  IF(ELEMENT_AT(kv,'09') IS NOT NULL, kv['09'], 0) AS m09,
  IF(ELEMENT_AT(kv,'10') IS NOT NULL, kv['10'], 0) AS m10,
  IF(ELEMENT_AT(kv,'11') IS NOT NULL, kv['11'], 0) AS m11,
  IF(ELEMENT_AT(kv,'12') IS NOT NULL, kv['12'], 0) AS m12
FROM
(
  SELECT t1.d, MAP_AGG(t1.m,uu) AS kv, MAX(uu_total) AS uu_total
  FROM res_activity t1
  JOIN
  ( SELECT * FROM stat ) t2
  ON t1.d = t2.d
  GROUP BY t1.d
)
ORDER BY d
```

#### 2. 行：月，列：日，値：PV
```sql
WITH 
res_activity AS
(
  SELECT
    TD_TIME_FORMAT(time, 'MM', 'JST') AS m,
    TD_TIME_FORMAT(time, 'dd', 'JST') AS d,
    COUNT(1) AS pv,
    COUNT(DISTINCT uid) AS uu
  FROM login
  WHERE TD_TIME_RANGE(time, '2012-01-01','2013-01-01', 'JST')
  GROUP BY TD_TIME_FORMAT(time, 'MM', 'JST'), TD_TIME_FORMAT(time, 'dd', 'JST')
),
stat AS
(
  SELECT
    TD_TIME_FORMAT(time, 'MM', 'JST') AS m,
    COUNT(1) AS pv_total,
    COUNT(DISTINCT uid) AS uu_total
  FROM login
  WHERE TD_TIME_RANGE(time, '2012-01-01','2013-01-01', 'JST')
  GROUP BY TD_TIME_FORMAT(time, 'MM', 'JST')
)

SELECT m, pv_total,
  IF(ELEMENT_AT(kv,'01') IS NOT NULL, kv['01'], 0) AS d01,
  IF(ELEMENT_AT(kv,'02') IS NOT NULL, kv['02'], 0) AS d02,
  IF(ELEMENT_AT(kv,'03') IS NOT NULL, kv['03'], 0) AS d03,
  IF(ELEMENT_AT(kv,'04') IS NOT NULL, kv['04'], 0) AS d04,
  IF(ELEMENT_AT(kv,'05') IS NOT NULL, kv['05'], 0) AS d05,
  IF(ELEMENT_AT(kv,'06') IS NOT NULL, kv['06'], 0) AS d06,
  IF(ELEMENT_AT(kv,'07') IS NOT NULL, kv['07'], 0) AS d07,
  IF(ELEMENT_AT(kv,'08') IS NOT NULL, kv['08'], 0) AS d08,
  IF(ELEMENT_AT(kv,'09') IS NOT NULL, kv['09'], 0) AS d09,
  IF(ELEMENT_AT(kv,'10') IS NOT NULL, kv['10'], 0) AS d10,
  IF(ELEMENT_AT(kv,'11') IS NOT NULL, kv['11'], 0) AS d11,
  IF(ELEMENT_AT(kv,'12') IS NOT NULL, kv['12'], 0) AS d12,
  IF(ELEMENT_AT(kv,'13') IS NOT NULL, kv['13'], 0) AS d13,
  IF(ELEMENT_AT(kv,'14') IS NOT NULL, kv['14'], 0) AS d14,
  IF(ELEMENT_AT(kv,'15') IS NOT NULL, kv['15'], 0) AS d15,
  IF(ELEMENT_AT(kv,'16') IS NOT NULL, kv['16'], 0) AS d16,
  IF(ELEMENT_AT(kv,'17') IS NOT NULL, kv['17'], 0) AS d17,
  IF(ELEMENT_AT(kv,'18') IS NOT NULL, kv['18'], 0) AS d18,
  IF(ELEMENT_AT(kv,'19') IS NOT NULL, kv['19'], 0) AS d19,
  IF(ELEMENT_AT(kv,'20') IS NOT NULL, kv['20'], 0) AS d20,
  IF(ELEMENT_AT(kv,'21') IS NOT NULL, kv['21'], 0) AS d21,
  IF(ELEMENT_AT(kv,'22') IS NOT NULL, kv['22'], 0) AS d22,
  IF(ELEMENT_AT(kv,'23') IS NOT NULL, kv['23'], 0) AS d23,
  IF(ELEMENT_AT(kv,'24') IS NOT NULL, kv['24'], 0) AS d24,
  IF(ELEMENT_AT(kv,'25') IS NOT NULL, kv['25'], 0) AS d25,
  IF(ELEMENT_AT(kv,'26') IS NOT NULL, kv['26'], 0) AS d26,
  IF(ELEMENT_AT(kv,'27') IS NOT NULL, kv['27'], 0) AS d27,
  IF(ELEMENT_AT(kv,'28') IS NOT NULL, kv['28'], 0) AS d28,
  IF(ELEMENT_AT(kv,'29') IS NOT NULL, kv['29'], 0) AS d29,
  IF(ELEMENT_AT(kv,'30') IS NOT NULL, kv['30'], 0) AS d30,
  IF(ELEMENT_AT(kv,'31') IS NOT NULL, kv['31'], 0) AS d31
FROM
(
  SELECT t1.m, MAP_AGG(t1.d,uu) AS kv, MAX(pv_total) AS pv_total
  FROM res_activity t1
  JOIN
  ( SELECT * FROM stat ) t2
  ON t1.m= t2.m
  GROUP BY t1.m
)
ORDER BY m
```

### 新規ユーザー数（月次）

新規ユーザー数（月次）は，ユーザーuidの最初のログインのタイムスタンプを月で丸めて集計します。

```sql
SELECT TD_TIME_FORMAT(first_login_time, 'yyyy-MM-01', 'JST') AS m, COUNT(1) AS new_user
FROM
(
  SELECT uid, MIN(time) AS first_login_time
  FROM login
  GROUP BY uid
) t1
GROUP BY TD_TIME_FORMAT(first_login_time, 'yyyy-MM-01', 'JST')
ORDER BY m ASC
```

### ログイン回数（デイリー）
ユーザーの1日当たりの平均ログイン回数を求めます。
```sql
SELECT
  TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d,
  COUNT(1) * 1.0 / COUNT(DISTINCT(uid)) AS play_times
FROM login
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
ORDER BY d ASC
```

### セッション回数（デイリー）
ユーザーの1日当たりの平均セッション回数（プレイ回数と解釈できる）を求めます。
レコードをセッション（ある時間までをひとまとまりとして扱い，idを割り振る）で割り振るには，Window関数を内包するTD_SESSIONIZE_WINDOW関数を使います。
uid，日付ごとのパーティションで30分を区切りとしてセッションIDを振っていきます。
uidごとの1日のセッション回数を求めるには，uidと日付をGROUP BYキーにしてCOUNT(DISTINCT session_id)を使います。
```sql
SELECT d, AVG(session_cnt) AS avg_session_cnt
FROM
(
  SELECT uid, d, COUNT(DISTINCT session_id) AS session_cnt
  FROM
  (
    SELECT
      uid, time,
      TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d,
      TD_SESSIONIZE_WINDOW(time,60*30)OVER(PARTITION BY uid, TD_TIME_FORMAT(time,'yyyy-MM-dd','JST') ORDER BY time) AS session_id
    FROM login
  )
  GROUP BY uid, d
)
GROUP BY d
ORDER BY d ASC
```

## Monetary KPI

### ARPU，ARPPU，DAU（デイリー）
```sql
WITH stat_login AS
(
  SELECT
    TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d,
    COUNT(distinct uid) AS uu,
    COUNT(1) AS pv
  FROM login
  GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
),
stat_pay AS
(
  SELECT
    TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d,
    COUNT(distinct uid) AS uu,
    SUM(price*amount) AS sales
  FROM pay
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
)

SELECT
  t1.d AS d, t1.uu AS uu_login, t2.uu AS uu_pay,
  1.0*t2.sales/t1.uu AS arpu, 1.0*t2.sales/t2.uu AS arppu
FROM stat_login t1
LEFT OUTER JOIN ( SELECT * FROM stat_pay ) t2
ON (t1.d=t2.d)
ORDER BY d asc, arpu, arppu
```

### ARPU，ARPPU，MAU（月次）
```sql
WITH stat_login AS
(
  SELECT
    TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS m,
    COUNT(distinct uid) AS uu,
    COUNT(1) AS pv
  FROM login
  GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
),
stat_pay AS
(
  SELECT
    TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS m,
    COUNT(distinct uid) AS uu,
    SUM(price*amount) AS sales
  FROM pay
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST')
)

SELECT
  t1.m AS m, t1.uu AS uu_login, t2.uu AS uu_pay,
  1.0*t2.sales/t1.uu AS arpu, 1.0*t2.sales/t2.uu AS arppu
FROM stat_login t1
LEFT OUTER JOIN ( SELECT * FROM stat_pay ) t2
ON (t1.m=t2.m)
ORDER BY m asc, arpu, arppu
```

### 新規課金ユーザー（月次）
```sql
SELECT TD_TIME_FORMAT(first_pay_time, 'yyyy-MM-01', 'JST') AS m, COUNT(1) AS new_pay_user
FROM
(
  SELECT uid, MIN(time) AS first_pay_time
  FROM pay
  GROUP BY uid
) t1
GROUP BY TD_TIME_FORMAT(first_pay_time, 'yyyy-MM-01', 'JST')
ORDER BY m ASC
```

### クロステーブル：インストール × 最初の課金

課金ユーザーを対象に，各インストール月から何ヶ月後に課金してくれたのか，月ごとの人数を見ていくコホートテーブルに近いクエリです。

```sql
WITH stat AS
(
  SELECT 
    uid, TD_TIME_FORMAT(MIN(time), 'yyyy-MM', 'JST') AS first_pay
  FROM pay
  GROUP BY uid
),
cohort_table AS
(
  SELECT first_login, first_pay, COUNT(1) AS cnt
  FROM
  (
    SELECT
      uid, TD_TIME_FORMAT(MIN(time), 'yyyy-MM', 'JST') AS first_login
    FROM login
    GROUP BY uid
  ) login
  LEFT OUTER JOIN
  (
    SELECT uid, first_pay FROM stat
  ) stat
  ON login.uid = stat.uid
  GROUP BY first_login, first_pay
)

SELECT first_pay,
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
  SELECT first_pay, MAP_AGG(first_login, cnt) AS kv
  FROM cohort_table
  GROUP BY first_pay
)
ORDER BY first_pay ASC
```

このテーブルでは，行方向に「初課金月」，列方向に「インストール（初回アクセス）月」が並んでいます。例えば2行目をまず横方向に見ると，2011-12における初課金ユーザーの内訳は11月登録者の4654人と12月登録者の620人であることがわかります。
また，2列目の「m2011_11」を縦方向に見ることで，2011-11に登録したユーザーのうち11月に2168人，12月に4654人，翌年1月に1144人，2月に969人，...が初課金していることがわかります。どの列を見ても，（2011-11を除いて）登録月に初課金したユーザーが最も多く，登録月から離れるほど課金する可能性が減っていく様子がわかります。縦方向を1つの色として分布をチャートにしたものを下記に示します。

### 課金ユーザーのコホート分析
コホート分析を課金ユーザーを対象に実施してみます。初課金月の人数をそれぞれ求め，それ以降の月で課金する人数が何人になっていくかをマトリクスで見ていきます。
```sql
WITH stat AS
(
  SELECT 
    uid, TD_TIME_FORMAT(MIN(time), 'yyyy-MM', 'JST') AS first_pay
  FROM pay
  GROUP BY uid
),
cohort_table AS
(
  SELECT m, first_pay, COUNT(1) AS cnt
  FROM
  (
    SELECT
      uid, TD_TIME_FORMAT(time, 'yyyy-MM', 'JST') AS m
    FROM pay
    GROUP BY uid, TD_TIME_FORMAT(time, 'yyyy-MM', 'JST')
  ) pay
  JOIN
  (
    SELECT uid, first_pay FROM stat
  ) stat
  ON pay.uid = stat.uid
  GROUP BY m, first_pay
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
  SELECT m, MAP_AGG(first_pay, cnt) AS kv
  FROM cohort_table
  GROUP BY m
)
ORDER BY m ASC
```

## Retention KPI
### 直帰率（デイリー）
デイリーでの直帰率を求めます。
```sql
WITH stat_bounce AS
(
  SELECT d, COUNT(IF(cnt=1,1,NULL)) AS cnt_bounce
  FROM
  (
    SELECT uid, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d, COUNT(1) AS cnt
    FROM login
    GROUP BY uid, TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
  )
  GROUP BY d
),
stat_login AS
(
  SELECT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d, COUNT(DISTINCT uid) AS uu
  FROM login
  GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
)


SELECT stat_bounce.d AS d, 1.0*cnt_bounce/uu AS bounce_ratio
FROM stat_bounce,stat_login
WHERE stat_bounce.d = stat_login.d
ORDER BY stat_bounce.d
```

### 3ヶ月後の復帰ユーザー
90日ほど音沙汰がなかったのに，直近7日以内にアクセスがあったユーザーを「復帰」とみなして歓迎することにし，その復帰ユーザーの数を数えます。もちろん，「直近7日」と「90日」は任意に指定可能です。
```sql
--TD_SCHEDULED_TIME()=2012-12-27
WITH recent_users AS
(
  SELECT uid
  FROM login
  WHERE TD_TIME_RANGE( time, TD_TIME_ADD(TD_SCHEDULED_TIME(), '-7d'), TD_SCHEDULED_TIME(), 'JST')
  GROUP BY uid
), 
first_access_users AS
(
  SELECT uid, MIN(time)
  FROM login
  GROUP BY uid
  HAVING MIN(time) < TD_TIME_ADD(TD_SCHEDULED_TIME(),'-97d','JST')
),
no_dormant_users AS
(
  SELECT uid
  FROM login
  WHERE TD_TIME_RANGE( time, TD_TIME_ADD(TD_SCHEDULED_TIME(), '-7d'), TD_TIME_ADD(TD_SCHEDULED_TIME(), '-97d'), 'JST')
  GROUP BY uid
),
dormant_users AS
(
  SELECT first_access_users.uid AS uid
  FROM first_access_users
  LEFT OUTER JOIN no_dormant_users
  ON first_access_users.uid = no_dormant_users.uid
  WHERE no_dormant_users.uid IS NULL
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, COUNT(1) AS cnt
FROM recent_users,dormant_users
WHERE recent_users.uid=dormant_users.uid
```
はじめのrecent_usersテーブルでは，基準日（このログは2012-12-27までなのでそれまでの日）から7日以内にアクセスのあったユーザーのリストを作っています。

次のdormant_usersテーブルでは，基準日から (7日前, 97日前] の90日間にアクセスがなかったユーザー（かつ登録日が97日よりも前）を求めています。「アクセスのあった」ユーザーを求めるのは簡単ですが，「アクセスがなかったユーザー」を求めるのは結構大変です。

はじめに，TD_TIME_ADD関数は「year」および「month」の加減算ができないことに注意してください。年や月は一定の時間ではないので，1ヶ月前を使うならば「-4w」および「-30d」を使います。

まず，97日より前が登録日のユーザーのリストを作ります（first_access_users）。次に， (7日前, 97日前] の期間にアクセスのあったユーザーのリストを作ります（no_dormant_users）。これらの一時テーブルをLEFT OUTER JOINしてno_dormant_usersのuidがNULLとなるユーザーが， (7日前, 97日前] の期間にアクティブでなかった，かつ登録が97日より前のユーザーです。そこで，LEFT OUTER JOIN後にWHERE節で「no_dormant_users.uid IS NULL」のuidだけを抽出します。このユーザー群は，直近7日間か，97日より前にアクティブだったユーザーです。このユーザーリストをdormant_usersテーブルとします。

最後に，recent_usersテーブルとdormant_usersテーブルをJOINすれば，所望のユーザーリストが得られ，その数を数えることで目的を果たすことができます。

### 7日連続ログイン
基準日までに7日連続でログインしてくれているユーザーを数えます。まず以下のクエリを見てみましょう。
```sql
--TD_SCHEDULED_TIME()=2012-12-27
WITH login_cnt_7d_table AS
(
  SELECT uid, COUNT( DISTINCT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')) AS login_cnt
  FROM login
  WHERE TD_INTERVAL(time, '-7d', 'JST') 
  GROUP BY uid
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, login_cnt, COUNT(1) AS cnt
FROM login_cnt_7d_table
GROUP BY login_cnt
ORDER BY login_cnt
```

これは基準日から7日前までの7日間の間に何日アクセスしてくれているかを日にちごとに集計したものです。ここで，連続アクセスを意味できるのは「login_cnt=7」のところだけです。よって

```sql
--TD_SCHEDULED_TIME()=2012-12-27
WITH login_cnt_7d_table AS
(
  SELECT uid, COUNT( DISTINCT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')) AS login_cnt
  FROM login
  WHERE TD_INTERVAL(time, '-7d', 'JST') 
  GROUP BY uid
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, login_cnt, COUNT(1) AS cnt
FROM login_cnt_7d_table
GROUP BY login_cnt
HAVING login_cnt=7
```

login_cntが7以下のユーザーは，「連続」ではなく，7日間にn日ログインしたという意味になります。他の連続日を取りたい場合は値を変えてUNION ALLします。
```sql
--TD_SCHEDULED_TIME()=2012-12-27
WITH login_cnt_7d_table AS
(
  SELECT uid, COUNT( DISTINCT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')) AS login_cnt
  FROM login
  WHERE TD_INTERVAL(time, '-7d', 'JST') 
  GROUP BY uid
), login_cnt_6d_table AS
(
  SELECT uid, COUNT( DISTINCT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')) AS login_cnt
  FROM login
  WHERE TD_INTERVAL(time, '-6d', 'JST') 
  GROUP BY uid
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, login_cnt, COUNT(1) AS cnt
FROM login_cnt_7d_table
GROUP BY login_cnt
HAVING login_cnt=7
UNION ALL
SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, login_cnt, COUNT(1) AS cnt
FROM login_cnt_6d_table
GROUP BY login_cnt
HAVING login_cnt=6
```

### 頻繁なログインユーザー
基準日までの直近1週間で5日以上ログインしているユーザーを求めます。
```sql
--TD_SCHEDULED_TIME()=2012-12-27
WITH login_cnt_7d_table AS
(
  SELECT uid, COUNT( DISTINCT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')) AS login_cnt
  FROM login
  WHERE TD_INTERVAL(time, '-7d', 'JST') 
  GROUP BY uid
)

SELECT TD_TIME_FORMAT(TD_SCHEDULED_TIME(),'yyyy-MM-dd','JST') AS d, login_cnt, COUNT(1) AS cnt
FROM login_cnt_7d_table
GROUP BY login_cnt
HAVING 5 <= login_cnt
ORDER BY login_cnt DESC
```
