# Lesson 19. Web ログ分析クエリ

## IPアドレスを分析するトレジャーデータUDF

```sql
SELECT  
  TD_IP_TO_COUNTRY_NAME(td_ip)                   AS country_name,
  TD_IP_TO_MOST_SPECIFIC_SUBDIVISION_NAME(td_ip) AS subdivision_name,
  TD_IP_TO_CITY_NAME(td_ip)                      AS city_name,
  TD_IP_TO_POSTAL_CODE(td_ip)                    AS postal_code,
  TD_IP_TO_LATITUDE(td_ip) AS lat, TD_IP_TO_LONGITUDE(td_ip) AS lon,
  TD_IP_TO_CONNECTION_TYPE(td_ip) AS conn_type,
  TD_IP_TO_DOMAIN(td_ip)          AS domain
FROM sample_accesslog
ORDER BY postal_code
LIMIT 10
```
|country_name|subdivision_name|city_name|postal_code|lat    |lon     |conn_type|domain|
|------------|----------------|---------|-----------|-------|--------|---------|------|
|Japan       |Hokkaido        |Obihiro  |001-0011   |42.8863|143.4148|Cable/DSL|      |
|Japan       |Hokkaido        |Obihiro  |001-0011   |42.8863|143.4148|Cable/DSL|      |
|Japan       |Hokkaido        |Obihiro  |001-0011   |42.8863|143.4148|Cable/DSL|      |


### 国，県，都市名を特定する
```sql
SELECT  
  TD_IP_TO_COUNTRY_NAME(td_ip) AS country_name,
  TD_IP_TO_MOST_SPECIFIC_SUBDIVISION_NAME(td_ip) AS subdivision_name,
  TD_IP_TO_CITY_NAME(td_ip) AS city_name,
  COUNT(1) AS cnt
FROM sample_accesslog
GROUP BY TD_IP_TO_COUNTRY_NAME(td_ip),
  TD_IP_TO_MOST_SPECIFIC_SUBDIVISION_NAME(td_ip),
  TD_IP_TO_CITY_NAME(td_ip)
ORDER BY cnt DESC
LIMIT 10
```
|country_name|subdivision_name|city_name|cnt|
|------------|----------------|---------|---|
|Japan       |NULL            |NULL     |8976|
|Japan       |Tokyo           |Tokyo    |7376|
|Japan       |Tokyo           |Shibakoen|5831|


### 郵便番号を特定し，市町村名にマッピングする
郵便番号に対応する市町村データを以下のURLからダウンロードし，ken_all_romeテーブルとして格納します。
https://www.post.japanpost.jp/zipcode/dl/roman-zip.html

ダウンロードしたファイルにはヘッダーがありませんので，以下を先頭行に挿入してください。
```sql
postal_code,pref_name,city_name,town_name,pref_name_rome,city_name_rome,town_name_rome
```

下記のクエリでは，IPアドレスより変換した郵便番号をken_all_romeと結合して詳細な市町村名を取得し，集計しています。ユニークなユーザーの市町村集計を行うために，DISTINCTを用いてIPアドレスを取ってきています。

```sql
SELECT s1.postal_code, pref_name_rome, city_name_rome, town_name_rome, COUNT(1) AS cnt
FROM
(
  SELECT DISTINCT td_client_id,
    REGEXP_REPLACE(TD_IP_TO_POSTAL_CODE(td_ip),'-','') AS postal_code
  FROM sample_accesslog
) s1
JOIN
(
  SELECT postal_code, pref_name_rome, city_name_rome, town_name_rome
  FROM ken_all_rome
) s2
ON s1.postal_code = s2.postal_code
GROUP BY s1.postal_code, pref_name_rome, city_name_rome, town_name_rome
ORDER BY cnt DESC
LIMIT 10
```
|postal_code|pref_name_rome|city_name_rome|town_name_rome|cnt |
|-----------|--------------|--------------|--------------|----|
|1020082    |TOKYO TO      |CHIYODA KU    |ICHIBANCHO    |1803|
|1540017    |TOKYO TO      |SETAGAYA KU   |SETAGAYA      |647 |
|1600021    |TOKYO TO      |SHINJUKU KU   |KABUKICHO     |484 |


### 接続タイプ，ドメインを特定する
IPアドレスからわかる興味深い情報として，接続タイプとドメインを特定することができます。
```sql
SELECT 
  TD_IP_TO_CONNECTION_TYPE(td_ip) AS conn_type,
  TD_IP_TO_DOMAIN(td_ip)          AS domain,
  COUNT(1) AS cnt
FROM sample_accesslog
WHERE TD_IP_TO_CONNECTION_TYPE(td_ip) IS NOT NULL AND TD_IP_TO_DOMAIN(td_ip) IS NOT NULL
GROUP BY TD_IP_TO_CONNECTION_TYPE(td_ip), TD_IP_TO_DOMAIN(td_ip)
ORDER BY cnt DESC
LIMIT 100
```
|conn_type |domain|cnt  |
|----------|------|-----|
|Cable/DSL |ucom.ne.jp|13633|
|Cable/DSL |ocn.ne.jp|6527 |
|Cable/DSL |dion.ne.jp|2862 |
|Cellular  |spmode.ne.jp|2494 |
|Cable/DSL |nuro.jp|2478 |


## アクティビティの推移

#### 使用するデータ：sample_accesslog_fluentd

今回は「ユーザーがアクセスしたか否か」に着目していきます。ここから導き出されるユーザーのアクティビティに基づく「セグメント」は，サイト全体のユーザーの出入りをダイナミックに表現してくれる有益なものになります。
サイトに訪問してくれるユーザーには，毎月継続的に来てくれる人もいれば，今月初めて来てくれた人もいます。また，既に訪問がなくなって確認できないユーザーの方がたくさんいるかもしれません。
今，ある時点においてその前月と前々月のユーザーごとのアクティビティ（アクセスがあったかなかったか）を調べ，以下のようなアクティビティセグメントに分類し，集計することを目的とします。
|activity_past_1 （1ヶ月前のアクティビティ）|activity_past_2 （2ヶ月前のアクティビティ）|セグメント名|
|------------------------------|------------------------------|------|
|active（継続）                    |active（継続）                    |継続    |
|active（継続）                    |non_active（休眠）                |休眠→復活 |
|active（継続）                    |welcome（新規）                   |新規→継続 |
|non_active（休眠）                |active（継続）                    |継続→休眠 |
|non_active（休眠）                |non_active（休眠）                |休眠    |
|non_active（休眠）                |welcome（新規）                   |新規→休眠 |
|welcome（新規）                   |NULL                          |新規    |

このセグメントによる集計が行えると，以下のような「2部グラフ」と呼ばれる可視化手法によってアクティビティの推移を確認することができます。

### 前月のアクティビティ

まず（ある時点における）前月において，ユーザーを「active（継続）」「non_active（休眠）」「welcome（新規）」のセグメントに分類します。このセグメントの定義は，
- 「active」は1回でもアクセスのあったユーザー
- 「non_active」は前月はアクセスのなかったが，過去にアクセスがあったユーザー（新規ユーザーと区別するため）
- 「welcome」は，は前月はアクセスのなく，過去に1回もアクセスがない

となり，後者2つを識別するためには前月1ヶ月間のデータだけではなく，それより過去のデータも参照しないといけないことがわかります。今，
- 1ヶ月前より過去のデータ：before_1month テーブル
- 前月1ヶ月間：past_1month テーブル
とすると，これらは以下のクエリで作成できます。今回はTD_INTERVALを使いますので冒頭のコメントアウトでTD_SCHEDULED_TIME関数を記述しておきます。また，実行時には日付を「2014-12-25」としてください。
```sql
/* ログの範囲 [2014-09-05, 2015-01-07] */
/* TD_SCHEDULED_TIME() = '2014-12-25 00:00:00' */
WITH before_1month AS
(
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-1M', 'JST') /* 1ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_1month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M', 'JST') /* 前月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
)
```

「1ヶ月前よりも過去」という表現はTD_INTERVALでは記述できませんので，初めにログの範囲を確認し，それよりも長い期間を指定することで代替します。今回は'-10y'としています。

さて，この2つのテーブルのユーザーリストにおいて，
- 双方に存在するユーザー：active
- before_1monthのみに存在するユーザー：non_active
- past_1monthのみに存在するユーザー：welcome

というセグメントになりますので，FULL OUTER JOINの上，JOINキー双方のNULLを確認することになります。
またactiveだったユーザーは，存在してる最大と最小の日付を併せて求めることで，その期間が前月（今回は11月）の範囲になっているかでクエリが正しいかを確認しています。
```sql
/* ログの範囲 [2014-09-05, 2015-01-07]
/* TD_SCHEDULED_TIME() = '2014-12-25 00:00:00' */
WITH before_1month AS
  (
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-1M', 'JST') /* 1ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_1month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M', 'JST') /* 前月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
)

SELECT
  IF(b1.td_client_id IS NOT NULL, b1.td_client_id, p1.td_client_id) AS td_client_id,
  CASE 
    WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS NOT NULL THEN 'active'
    WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS     NULL THEN 'non_active'
    WHEN b1.td_client_id IS     NULL AND p1.td_client_id IS NOT NULL THEN 'welcome'
  END AS segment,
  TD_TIME_FORMAT(p1.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
  TD_TIME_FORMAT(p1.max_time, 'yyyy-MM-dd', 'JST') AS max_time
FROM before_1month b1
FULL OUTER JOIN past_1month p1
ON b1.td_client_id = p1.td_client_id
```
|td_client_id|segment|min_time|max_time  |
|------------|-------|--------|----------|
|2788574a-5be7-43c8-a73a-8bd47706a34f|non_active|NULL    |NULL      |
|0495933c-9608-40d1-ea1c-e81b71ae15d3|active |2014-11-02|2014-11-27|
|1b4b27c5-923c-4a18-859e-4cb7aca2799c|non_active|NULL    |NULL      |


一度，セグメントごとの人数を確認してみましょう。
```sql
/* ログの範囲 [2014-09-05, 2015-01-07]
/* TD_SCHEDULED_TIME() = '2014-12-25 00:00:00' */
WITH before_1month AS
  (
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-1M', 'JST') /* 1ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_1month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M', 'JST') /* 前月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
activity_past_1month AS
(
  SELECT
    IF(b1.td_client_id IS NOT NULL, b1.td_client_id, p1.td_client_id) AS td_client_id,
    CASE 
      WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS NOT NULL THEN 'active'
      WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS     NULL THEN 'non_active'
      WHEN b1.td_client_id IS     NULL AND p1.td_client_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p1.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p1.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_1month b1
  FULL OUTER JOIN past_1month p1
  ON b1.td_client_id = p1.td_client_id
)

SELECT segment, COUNT(1) AS cnt
FROM activity_past_1month
GROUP BY segment
```
|segment   |cnt   |
|----------|------|
|active    |1262  |
|welcome   |10793 |
|non_active|19442 |


さてこの結果でも有用な情報ですが，この結果からはnon_activeだった人が，前から non_activeだったのか，直近まではactiveだったのか，はたまたwelcomeからすぐにnon_activeになったのか，といった背景が読み取れません。そこで，前々月のアクティビティを同様に求めることでこれらの背景を明確にしていきます。

### 前々月のアクティビティ
前月のアクティビティにおけるアクティビティとほぼ同じクエリで，テーブル名とTD_INTERVALのハイライト部分だけ変更されています。
```sql
/* ログの範囲 [2014-09-05, 2015-01-07]
/* TD_SCHEDULED_TIME() = '2014-12-25 00:00:00' */
WITH before_2month AS
(
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-2M', 'JST') /* 2ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_2month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M/-1M', 'JST') /* 前々月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
activity_past_2month AS
(
  SELECT
    IF(b2.td_client_id IS NOT NULL, b2.td_client_id, p2.td_client_id) AS td_client_id,
    CASE 
      WHEN b2.td_client_id IS NOT NULL AND p2.td_client_id IS NOT NULL THEN 'active'
      WHEN b2.td_client_id IS NOT NULL AND p2.td_client_id IS     NULL THEN 'non_active'
      WHEN b2.td_client_id IS     NULL AND p2.td_client_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p2.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p2.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_2month b2
  FULL OUTER JOIN past_2month p2
  ON b2.td_client_id = p2.td_client_id
)

SELECT segment, COUNT(1) AS cnt
FROM activity_past_2month
GROUP BY segment
```
|segment   |cnt   |
|----------|------|
|active    |1003  |
|welcome   |11244 |
|non_active|8457  |


### アクティビティの推移
前月と前々月のユーザーごとのアクティビティセグメントをLEFT OUTER JOINすることで同ユーザーのセグメントの推移が確認できます。（下のクエリは主要部分だけ抜粋しています。）
```sql
SELECT 
  TD_TIME_FORMAT(TD_DATE_TRUNC('month', TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(),'JST')-1, 'JST'),'yyyy-MM-dd','JST') AS target_month,
  activity_past_1month.td_client_id, 
  activity_past_1month.segment AS activity_past_1,
  activity_past_2month.segment AS activity_past_2,
  activity_past_1month.min_time AS min_time_past_1,
  activity_past_1month.max_time AS max_time_past_1,
  activity_past_2month.min_time AS min_time_past_2, 
  activity_past_2month.max_time AS max_time_past_2
FROM activity_past_1month
LEFT OUTER JOIN activity_past_2month
ON activity_past_1month.td_client_id = activity_past_2month.td_client_id
/* 集合 activity_past_1month は activity_past_2month を内包する */
```

さらにセグメントの組合せで集計したクエリを，実行可能なクエリとして記述します。
```sql
/* ログの範囲 [2014-09-05, 2015-01-07]
/* TD_SCHEDULED_TIME() = '2014-12-25 00:00:00' */
WITH before_1month AS
  (
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-1M', 'JST') /* 1ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
before_2month AS
(
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-2M', 'JST') /* 2ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_1month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M', 'JST') /* 前月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_2month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M/-1M', 'JST') /* 前々月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
activity_past_1month AS
(
  SELECT
    IF(b1.td_client_id IS NOT NULL, b1.td_client_id, p1.td_client_id) AS td_client_id,
    CASE 
      WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS NOT NULL THEN 'active'
      WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS     NULL THEN 'non_active'
      WHEN b1.td_client_id IS     NULL AND p1.td_client_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p1.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p1.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_1month b1
  FULL OUTER JOIN past_1month p1
  ON b1.td_client_id = p1.td_client_id
),
activity_past_2month AS
(
  SELECT
    IF(b2.td_client_id IS NOT NULL, b2.td_client_id, p2.td_client_id) AS td_client_id,
    CASE 
      WHEN b2.td_client_id IS NOT NULL AND p2.td_client_id IS NOT NULL THEN 'active'
      WHEN b2.td_client_id IS NOT NULL AND p2.td_client_id IS     NULL THEN 'non_active'
      WHEN b2.td_client_id IS     NULL AND p2.td_client_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p2.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p2.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_2month b2
  FULL OUTER JOIN past_2month p2
  ON b2.td_client_id = p2.td_client_id
),
activity_users AS
(
  SELECT 
    TD_TIME_FORMAT(TD_DATE_TRUNC('month', TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(),'JST')-1, 'JST'),'yyyy-MM-dd','JST') AS target_month,
    activity_past_1month.td_client_id, 
    activity_past_1month.segment AS activity_past_1,
    activity_past_2month.segment AS activity_past_2,
    activity_past_1month.min_time AS min_time_past_1, activity_past_1month.max_time AS max_time_past_1,
    activity_past_2month.min_time AS min_time_past_2, activity_past_2month.max_time AS max_time_past_2
  FROM activity_past_1month
  LEFT OUTER JOIN activity_past_2month /* 集合 activity_past_1month は activity_past_2month を内包する */
  ON activity_past_1month.td_client_id = activity_past_2month.td_client_id
)

SELECT target_month, 
  a.activity_past_1, 
  a.activity_past_2, 
  COUNT(1) AS cnt
FROM activity_users a
GROUP BY target_month, a.activity_past_1, a.activity_past_2
ORDER BY activity_past_1, activity_past_2
```
|target_month|activity_past_1|activity_past_2|cnt  |
|------------|---------------|---------------|-----|
|2014-11-01  |active         |active         |245  |
|2014-11-01  |active         |non_active     |151  |
|2014-11-01  |active         |welcome        |866  |
|2014-11-01  |non_active     |active         |758  |
|2014-11-01  |non_active     |non_active     |8306 |
|2014-11-01  |non_active     |welcome        |10378|
|2014-11-01  |welcome        |NULL           |10793|


### セグメント名の割り振り
最後に，冒頭で紹介したセグメント名を割り振った集計結果を出してみます。
|activity_past_1 （1ヶ月前のアクティビティ）|activity_past_2 （2ヶ月前のアクティビティ）|セグメント名|
|------------------------------|------------------------------|------|
|active（継続）                    |active（継続）                    |継続    |
|active（継続）                    |non_active（休眠）                |休眠→復活 |
|active（継続）                    |welcome（新規）                   |新規→継続 |
|non_active（休眠）                |active（継続）                    |継続→休眠 |
|non_active（休眠）                |non_active（休眠）                |休眠    |
|non_active（休眠）                |welcome（新規）                   |新規→休眠 |
|welcome（新規）                   |NULL                          |新規    |

```sql
/* ログの範囲 [2014-09-05, 2015-01-07]
/* TD_SCHEDULED_TIME() = '2014-12-25 00:00:00' */
WITH before_1month AS
  (
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-1M', 'JST') /* 1ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
before_2month AS
(
  SELECT td_client_id
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-10y/-2M', 'JST') /* 2ヶ月前より過去 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_1month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M', 'JST') /* 前月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
past_2month AS
(
  SELECT td_client_id, MIN(time) AS min_time, MAX(time) AS max_time
  FROM sample_accesslog_fluentd
  WHERE TD_INTERVAL(time, '-1M/-1M', 'JST') /* 前々月1ヶ月間 */
  AND td_client_id IS NOT NULL
  GROUP BY td_client_id
),
activity_past_1month AS
(
  SELECT
    IF(b1.td_client_id IS NOT NULL, b1.td_client_id, p1.td_client_id) AS td_client_id,
    CASE 
      WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS NOT NULL THEN 'active'
      WHEN b1.td_client_id IS NOT NULL AND p1.td_client_id IS     NULL THEN 'non_active'
      WHEN b1.td_client_id IS     NULL AND p1.td_client_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p1.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p1.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_1month b1
  FULL OUTER JOIN past_1month p1
  ON b1.td_client_id = p1.td_client_id
),
activity_past_2month AS
(
  SELECT
    IF(b2.td_client_id IS NOT NULL, b2.td_client_id, p2.td_client_id) AS td_client_id,
    CASE 
      WHEN b2.td_client_id IS NOT NULL AND p2.td_client_id IS NOT NULL THEN 'active'
      WHEN b2.td_client_id IS NOT NULL AND p2.td_client_id IS     NULL THEN 'non_active'
      WHEN b2.td_client_id IS     NULL AND p2.td_client_id IS NOT NULL THEN 'welcome'
    END AS segment,
    TD_TIME_FORMAT(p2.min_time, 'yyyy-MM-dd', 'JST') AS min_time,
    TD_TIME_FORMAT(p2.max_time, 'yyyy-MM-dd', 'JST') AS max_time
  FROM before_2month b2
  FULL OUTER JOIN past_2month p2
  ON b2.td_client_id = p2.td_client_id
),
activity_users AS
(
  SELECT 
    TD_TIME_FORMAT(TD_DATE_TRUNC('month', TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(),'JST')-1, 'JST'),'yyyy-MM-dd','JST') AS target_month,
    activity_past_1month.td_client_id, 
    activity_past_1month.segment AS activity_past_1,
    activity_past_2month.segment AS activity_past_2,
    activity_past_1month.min_time AS min_time_past_1, activity_past_1month.max_time AS max_time_past_1,
    activity_past_2month.min_time AS min_time_past_2, activity_past_2month.max_time AS max_time_past_2
  FROM activity_past_1month
  LEFT OUTER JOIN activity_past_2month /* 集合 activity_past_1month は activity_past_2month を内包する */
  ON activity_past_1month.td_client_id = activity_past_2month.td_client_id
),
activity_summary AS
(
  SELECT target_month, a.activity_past_1, a.activity_past_2, COUNT(1) AS cnt,
    MIN(min_time_past_1) AS min_time_past_1, MAX(max_time_past_1) AS max_time_past_1,
    MIN(min_time_past_2) AS min_time_past_2, MAX(max_time_past_2) AS max_time_past_2
  FROM activity_users a
  GROUP BY target_month, a.activity_past_1, a.activity_past_2
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

SELECT target_month, segment_name, cnt
FROM activity_summary a
JOIN master_activity m
ON a.activity_past_1 = m.activity_past_1 AND a.activity_past_2 IS NOT DISTINCT FROM m.activity_past_2
ORDER BY segment_name
```
|target_month|segment_name|cnt       |
|------------|------------|----------|
|2014-11-01  |休眠          |8306      |
|2014-11-01  |休眠→復活       |151       |
|2014-11-01  |新規          |10793     |
|2014-11-01  |新規→休眠       |10378     |
|2014-11-01  |新規→継続       |866       |
|2014-11-01  |継続          |245       |
|2014-11-01  |継続→休眠       |758       |


このクエリはmaster_activityでセグメント名に変換する際に注意が必要で，「新規」のセグメント名を割り振る場合には，双方のactivity_past_2をNULLで一致させることになるのでON節において「IS NOT DISTINCT FROM」を使っている点です。これが単純なイコールですと「新規」のセグメントは結合後に登場しません。

この1回のクエリでは1つのtarget_monthにおけるその月と前月での推移をみています。target_monthを1ヶ月ずつずらしていき，実行結果を積み重ねれば，各セグメントごとの人数の推移を見ることができます。

また，さらなるセグメントの細分化としてはactiveの中でも特定のコンバージョンページを訪問してくれたユーザーとそうでないユーザーで分類することで，コンバージョンも踏まえたアクティビティの推移を見ることができます。ここではそのマスタだけ紹介するに止めておきます。
|activity_past_1|activity_past_2|segment_name|importance|
|---------------|---------------|------------|----------|
|active         |active         |継続          |4         |
|active         |active_cv      |CV→継続       |4         |
|active         |non_active     |休眠→復活       |4         |
|active         |welcome        |新規→継続       |4         |
|active         |welcome_cv     |新規CV→継続     |3         |
|active_cv      |active         |継続→継続CV     |4         |
|active_cv      |active_cv      |継続CV        |5         |
|active_cv      |non_active     |休眠→復活CV     |5         |
|active_cv      |welcome        |新規→継続CV     |5         |
|active_cv      |welcome_cv     |新規CV→継続CV   |5         |
|non_active     |active         |継続→休眠       |2         |
|non_active     |active_cv      |継続CV→休眠     |2         |
|non_active     |non_active     |休眠          |1         |
|non_active     |welcome        |新規→休眠       |1         |
|non_active     |welcome_cv     |新規CV→休眠     |1         |
|welcome        |               |新規          |3         |
|welcome_cv     |               |新規CV        |5         |

## 1セッション内におけるユーザーの閲覧タイトル集計
1セッションの中でユーザーが閲覧したタイトル群を集計します。1セッション内で閲覧されたタイトルはリストとして格納されるので，リストの要素と順番が同じものを集計します。これは，閲覧したタイトルの順番と閲覧したページ数が考慮され区別されていることを意味します。
なお，td_titleの多くは末尾に「 - Treasure Data」と付いているので，これを除外しておきます。1セッションの設定は60分です。sizeが1のリストは直帰か同ページのループにほぼ等しいと考えられます。
```sql
WITH session_table AS
(
  SELECT
    td_client_id, REGEXP_REPLACE(td_title,' - Treasure Data', '') AS td_title, time,
    TD_SESSIONIZE_WINDOW(time,60*60)OVER(PARTITION BY td_client_id ORDER BY time) AS session_id
  FROM sample_accesslog
)

SELECT td_title_list, CARDINALITY(td_title_list) AS size, COUNT(1) AS cnt
FROM
(
  SELECT td_client_id, session_id, ARRAY_AGG(td_title) AS td_title_list
  FROM session_table
  GROUP BY td_client_id, session_id
)
GROUP BY td_title_list
ORDER BY cnt DESC
```
|td_title_list|size  |cnt       |
|-------------|------|----------|
|["Treasure Data - データ分析をクラウドで、シンプルに。"]|1     |14080     |
|["Treasure Data - データ分析をクラウドで、シンプルに。", "Treasure Data - データ分析をクラウドで、シンプルに。"]|2     |3305      |
|["Treasure Data - データ分析をクラウドで、シンプルに。", "企業情報"]|2     |858       |
|["プライベートDMPソリューション”TREASURE(トレジャー) DMP(ディーエムピー)”を 4月より提供開始 - プレスリリース"]|1     |659       |
|["Treasure Data - データ分析をクラウドで、シンプルに。", "サービス概要"]|2     |504       |


次に，順番とリスト内のタイトルの重複を除外したリストによる集計を行います。これは，1セッション内で閲覧されたユニークなタイトルの集合を意味します。
```sql
WITH session_table AS
(
  SELECT
    td_client_id, REGEXP_REPLACE(td_title,' - Treasure Data', '') AS td_title, time,
    TD_SESSIONIZE_WINDOW(time,60*60)OVER(PARTITION BY td_client_id ORDER BY time) AS session_id
  FROM sample_accesslog
)

SELECT td_title_list, CARDINALITY(td_title_list) AS size, COUNT(1) AS cnt
FROM
(
  SELECT td_client_id, session_id, ARRAY_SORT(ARRAY_DISTINCT(ARRAY_AGG(td_title))) AS td_title_list
  FROM session_table
  GROUP BY td_client_id, session_id
)
GROUP BY td_title_list
ORDER BY cnt DESC
```
|td_title_list|size  |cnt       |
|-------------|------|----------|
|["Treasure Data - データ分析をクラウドで、シンプルに。"]|1     |17920     |
|["Treasure Data - データ分析をクラウドで、シンプルに。", "企業情報"]|2     |1187      |
|["Treasure Data - データ分析をクラウドで、シンプルに。", "サービス概要"]|2     |763       |
|["プライベートDMPソリューション”TREASURE(トレジャー) DMP(ディーエムピー)”を 4月より提供開始 - プレスリリース"]|1     |704       |
|["TD Private Seminar"]|1     |556       |


## パス分析

パス分析は，従来のコンバージョン分析（単純にコンバージョンページのPVを数えるなど）に対して，ランディングからコンバージョンに至るまでのパス（経路）を集計することでユーザーがコンバージョンに至ったモチベーションがどこにあるのかを知る手がかりとなる分析です。
従来のコンバージョン分析には，下図のように，直前のページだけしか考慮しないという欠点があります。

これに対しパス分析では，下図のように，個々のユーザーについてコンバージョンごとの経路全体に着目します。

パス分析によって得られる知見としては，「コンバージョンから遠いが必ず媒介ページとなるものの発見」，「パスの長さとページ内訳の比較によるコンバージョン効率化への道筋」などが挙げられます。

この分析で作られる「コンバージョンパス」と呼ばれるテーブルの例を下記に示します（本データとは異なるものです）。アクセスログをユーザーごとに時系列で並べ，コンバージョンが起きるごとに区切りを入れることで，ユーザーごとかつコンバージョンごとのレコードグループを作ります。このグループごとにコンバージョンIDを割り振れば，様々な分析のインプットテーブルの完成です。

トレジャーデータにおけるパス分析には，td_global_idと連携することで自社サイトに限らず外部サイトの回遊を含めたパスを作れるという利点があります。
ただし，ユーザーごとに異なるパスの長さ（コンバージョンに至るまでに遷移したページ数）のバリエーションすべてで集計するのは効率的ではありませんし，計算機的に無理が生じるかもしれません（それでも大規模サイトでは有効です）。ここでは，パス分析の入門として，コンバージョンに至るまでに通ったユニークなページの集合を取り，その集合で集計することにします。

### Step1. コンバージョンマスタテーブル
はじめに，どのページをコンバージョンとみなすかのマスタを作成します。このマスタの作り方はある程度柔軟で，以下の例ではtd_titleにおいて特定のタイトル文字列を含むものをコンバージョンとみなしています。今回は活用しませんが，コンバージョンごとの重要度をcv_levelとして区別しておき，ケースによってどの重要度のコンバージョンまで含んで分析するかを決めることができます。
```sql
SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
```
|cv_level  |cv_title|
|----------|--------|
|5         |資料ダウンロード|
|4         |お問い合わせ確認|
|3         |お申し込みを受け付けました|


### Step2. コンバージョン履歴テーブル
下記のクエリにより，コンバージョンのあったユーザーのアクセスログで該当するコンバージョンレコードを抽出したコンバージョン履歴テーブルを作ります。
```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
)

SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level
  )
  ORDER BY td_client_id, cv_id
```
|td_client_id|cv_title|td_title                |cv_id|time      |
|------------|--------|------------------------|-----|----------|
|003e1280-7640-4cba-8ab8-b9a55fc12923|お問い合わせ確認|お問い合わせ確認 - Treasure Data|1    |1462931828|
|0075c928-2661-4d44-c5da-e244faa85e77|資料ダウンロード|資料ダウンロードを受け付けました        |1    |1462516530|
|009fb5f1-b726-4cce-f6cd-ea0eb8c85068|資料ダウンロード|資料ダウンロードを受け付けました        |1    |1466864498|

クエリ中の下記の部分ではコンバージョンレコードを抽出しています。master_cvの定義を変えてURLで比較したり，cv_levelを引き上げたりする場合には，ここを変えることになります。
```sql
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level
```

### Step3. コンバージョンパス
下記のクエリによりコンバージョンパスを作成します。同じユーザーでコンバージョン履歴とアクセスログとを結合し，各レコードがどのコンバージョンIDに該当するかを判断しています。
```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
),
cv_history AS
(
  SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level 
  )
),
cv_raw_join_table AS
(
  SELECT
    log.td_client_id, cv_id, log.td_title, cv_title, log.time AS time, cv_history.time AS cv_time
  FROM sample_accesslog log
  JOIN cv_history
  ON log.td_client_id = cv_history.td_client_id
  AND log.time <= cv_history.time       
),
cv_organized_join_table AS
(
  SELECT
    td_client_id, td_title, time,
    MIN(cv_id) AS cv_id,
    MIN(cv_time) AS cv_time,
    IF((MIN(cv_time)-time)=0,1,0) AS cv_flag,
    MIN(cv_title) AS cv_title
  FROM cv_raw_join_table
  GROUP BY td_client_id, td_title, time
)

SELECT
  td_client_id, 
  ROW_NUMBER()OVER(PARTITION BY td_client_id, cv_id ORDER BY time) AS node_id,
  cv_id, 
  cv_flag, 
  td_title, 
  cv_title, 
  time,
  cv_time
FROM cv_organized_join_table
ORDER BY td_client_id, cv_id, node_id
```
|td_client_id|node_id|cv_id                   |cv_flag             |td_title|cv_title|time      |cv_time   |
|------------|-------|------------------------|--------------------|--------|--------|----------|----------|
|003e1280-7640-4cba-8ab8-b9a55fc12923|1      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|お問い合わせ確認|1462930626|1462931828|
|003e1280-7640-4cba-8ab8-b9a55fc12923|2      |1                       |0                   |お問い合わせ - Treasure Data|お問い合わせ確認|1462930688|1462931828|
|003e1280-7640-4cba-8ab8-b9a55fc12923|3      |1                       |1                   |お問い合わせ確認 - Treasure Data|お問い合わせ確認|1462931828|1462931828|
|0075c928-2661-4d44-c5da-e244faa85e77|1      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1462516299|1462516530|
|0075c928-2661-4d44-c5da-e244faa85e77|2      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1462516300|1462516530|
|0075c928-2661-4d44-c5da-e244faa85e77|3      |1                       |1                   |資料ダウンロードを受け付けました|資料ダウンロード|1462516530|1462516530|
|009fb5f1-b726-4cce-f6cd-ea0eb8c85068|1      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1466864410|1466864498|
|009fb5f1-b726-4cce-f6cd-ea0eb8c85068|2      |1                       |1                   |資料ダウンロードを受け付けました|資料ダウンロード|1466864498|1466864498|
|014a3104-0d75-4e91-b898-17188255175a|1      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1463449460|1466750101|
|014a3104-0d75-4e91-b898-17188255175a|2      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1466747106|1466750101|
|014a3104-0d75-4e91-b898-17188255175a|3      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1466747108|1466750101|
|014a3104-0d75-4e91-b898-17188255175a|4      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1466750063|1466750101|
|014a3104-0d75-4e91-b898-17188255175a|5      |1                       |0                   |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|資料ダウンロード|1466750064|1466750101|
|014a3104-0d75-4e91-b898-17188255175a|6      |1                       |1                   |資料ダウンロードを受け付けました|資料ダウンロード|1466750101|1466750101|
|014a3104-0d75-4e91-b898-17188255175a|1      |2                       |1                   |資料ダウンロードを受け付けました|資料ダウンロード|1466750122|1466750122|

### Step3’. 非コンバージョンパス
パスの数（絶対数）を求めるだけならばコンバージョンパスだけで可能ですが，コンバージョンしなかった同じパスの数と比較すれば比率を求めることができます。絶対数ではトップページを含むパスの回数が多く現れていたものが，比率を見ることでより内部の重要なページにフォーカスが当てられることもあります。
非コンバージョンパスは，コンバージョン履歴にないユーザーを抽出し，ユーザーごとのアクセスレコードでコンバージョンIDを1と割り振っただけの単純なものです。
```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
),
cv_history AS
(
  SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level 
  )
)

  SELECT
    log.td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY log.td_client_id ORDER BY time) AS node_id,
    1 AS cv_id, log.td_title, log.time AS time
  FROM sample_accesslog log
  LEFT OUTER JOIN ( SELECT td_client_id FROM cv_history ) cv_history
  ON cv_history.td_client_id IS NULL
  ORDER BY td_client_id, node_id
```
|td_client_id|node_id|cv_id                   |td_title|time      |
|------------|-------|------------------------|--------|----------|
|000077fb-2c93-4cd7-d9d0-293866aaec31|1      |1                       |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|1461454040|
|000077fb-2c93-4cd7-d9d0-293866aaec31|2      |1                       |企業情報 - Treasure Data|1461454057|
|000077fb-2c93-4cd7-d9d0-293866aaec31|3      |1                       |採用情報 - Treasure Data|1461454142|
|000077fb-2c93-4cd7-d9d0-293866aaec31|4      |1                       |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|1461454166|
|0005348a-a71f-4d65-e90c-599c9590312d|1      |1                       |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|1463487235|
|000696ff-bf86-4f0f-8ede-b74825fa3a83|1      |1                       |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|1464672788|
|000696ff-bf86-4f0f-8ede-b74825fa3a83|2      |1                       |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|1464672788|
|0012bd82-a511-49d6-a4b0-baf9d4949c5f|1      |1                       |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|1466830259|
|0012bd82-a511-49d6-a4b0-baf9d4949c5f|2      |1                       |サービス概要 - Treasure Data|1466830321|

### Step4. コンバージョンパスにおける（1ステップ）遷移回数分析
コンバージョンパスを求められたら，そのパス内のページからページへの遷移回数を求めて集計します。例えば， {A→B→C→A→G} というパスがあるとすると，「A→B」「B→C」「C→A」「A→G」が1回ずつ数えられます。この集計により，せっかくのパスを分解して点で見ているように思えるかもしれませんが，この遷移の全体を可視化（後ほど紹介）すると大きな流れを見ることができ有益です。
```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
),
cv_history AS
(
  SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level 
  )
),
cv_raw_join_table AS
(
  SELECT
    log.td_client_id, cv_id, log.td_title, cv_title, log.time AS time, cv_history.time AS cv_time
  FROM sample_accesslog log
  JOIN cv_history
  ON log.td_client_id = cv_history.td_client_id
  AND log.time <= cv_history.time       
),
cv_organized_join_table AS
(
  SELECT
    td_client_id, td_title, time,
    MIN(cv_id) AS cv_id,
    MIN(cv_time) AS cv_time,
    IF((MIN(cv_time)-time)=0,1,0) AS cv_flag,
    MIN(cv_title) AS cv_title
  FROM cv_raw_join_table
  GROUP BY td_client_id, td_title, time
),
cv_path AS
(
  SELECT
    td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY td_client_id, cv_id ORDER BY time) AS node_id,
    cv_id, cv_flag, td_title, cv_title, time, cv_time
  FROM cv_organized_join_table
  ORDER BY td_client_id, cv_id, node_id
)

SELECT cv_title, td_title, lead_td_title, COUNT(1) AS cnt
FROM
(
  SELECT td_client_id, td_title, LEAD(td_title)OVER(PARTITION BY td_client_id, cv_id ORDER BY node_id) AS lead_td_title, cv_title
  FROM cv_path
)
WHERE lead_td_title IS NOT NULL
GROUP BY cv_title, td_title, lead_td_title
ORDER BY cv_title, cnt DESC
```
|cv_title  |td_title|lead_td_title           |cnt|
|----------|--------|------------------------|---|
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|111|
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|サービス概要 - Treasure Data  |49 |
|お問い合わせ確認  |リソース - Treasure Data|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|37 |
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|リソース - Treasure Data    |34 |


### Step4’. 非コンバージョンパスにおける（1ステップ）遷移回数分析
同様に非コンバージョンパスからも遷移回数を集計しておきます。

```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
),
cv_history AS
(
  SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level 
  )
),
non_cv_path AS
(
  SELECT
    log.td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY log.td_client_id ORDER BY time) AS node_id,
    1 AS cv_id, log.td_title, log.time AS time
  FROM sample_accesslog log
  LEFT OUTER JOIN ( SELECT td_client_id FROM cv_history ) cv_history
  ON cv_history.td_client_id IS NULL
)

SELECT td_title, lead_td_title, COUNT(1) AS cnt
FROM
(
  SELECT td_client_id, td_title, LEAD(td_title)OVER(PARTITION BY td_client_id, cv_id ORDER BY node_id) AS lead_td_title
  FROM non_cv_path
)
WHERE lead_td_title IS NOT NULL
GROUP BY td_title, lead_td_title
ORDER BY cnt DESC 
```
|td_title  |lead_td_title|cnt                     |
|----------|-------------|------------------------|
|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|10358                   |
|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|サービス概要 - Treasure Data|2235                    |
|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|企業情報 - Treasure Data|2166                    |
|サービス概要 - Treasure Data|サービス概要 - Treasure Data|1562                    |


### Step5. コンバージョン率の高い遷移導出
同じ遷移のコンバージョンと非コンバージョン回数があれば，各々の遷移について以下のようにCV率を求めることができます。
- CV率 = CV回数 / (CV回数+非CV回数)
もしCV率の高い遷移がうまく辿れてコンバージョンにつながっていれば，その一連の遷移はコンバージョンへ至る王道（ゴールデンパス）と呼べるでしょう。

```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
),
cv_history AS
(
  SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level 
  )
),
cv_raw_join_table AS
(
  SELECT
    log.td_client_id, cv_id, log.td_title, cv_title, log.time AS time, cv_history.time AS cv_time
  FROM sample_accesslog log
  JOIN cv_history
  ON log.td_client_id = cv_history.td_client_id
  AND log.time <= cv_history.time       
),
cv_organized_join_table AS
(
  SELECT
    td_client_id, td_title, time,
    MIN(cv_id) AS cv_id,
    MIN(cv_time) AS cv_time,
    IF((MIN(cv_time)-time)=0,1,0) AS cv_flag,
    MIN(cv_title) AS cv_title
  FROM cv_raw_join_table
  GROUP BY td_client_id, td_title, time
),
cv_path AS
(
  SELECT
    td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY td_client_id, cv_id ORDER BY time) AS node_id,
    cv_id, cv_flag, td_title, cv_title, time, cv_time
  FROM cv_organized_join_table
  ORDER BY td_client_id, cv_id, node_id
),
cv_trans AS
(
  SELECT cv_title, td_title, lead_td_title, COUNT(1) AS cnt
  FROM
  (
    SELECT td_client_id, td_title, LEAD(td_title)OVER(PARTITION BY td_client_id, cv_id ORDER BY node_id) AS lead_td_title, cv_title
    FROM cv_path
  )
  WHERE lead_td_title IS NOT NULL
  GROUP BY cv_title, td_title, lead_td_title
),
non_cv_path AS
(
  SELECT
    log.td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY log.td_client_id ORDER BY time) AS node_id,
    1 AS cv_id, log.td_title, log.time AS time
  FROM sample_accesslog log
  LEFT OUTER JOIN ( SELECT td_client_id FROM cv_history GROUP BY td_client_id ) cv_history
  ON cv_history.td_client_id IS NULL
),
non_cv_trans AS
(
  SELECT td_title, lead_td_title, COUNT(1) AS cnt
  FROM
  (
    SELECT td_client_id, td_title, LEAD(td_title)OVER(PARTITION BY td_client_id, cv_id ORDER BY node_id) AS lead_td_title
    FROM non_cv_path
  )
  WHERE lead_td_title IS NOT NULL
  GROUP BY td_title, lead_td_title
)

SELECT cv_title, t1.td_title, t1.lead_td_title, 1.0*t1.cnt/(t1.cnt+t2.cnt) AS cv_ratio, t1.cnt AS cv_cnt, t2.cnt AS non_cv_cnt
FROM cv_trans t1
JOIN
( SELECT td_title, lead_td_title, cnt FROM non_cv_trans ) t2
ON t1.td_title = t2.td_title AND t1.lead_td_title = t2.lead_td_title
ORDER BY cv_title, cv_cnt DESC
```
|cv_title  |td_title|lead_td_title           |cv_ratio            |cv_cnt|non_cv_cnt|
|----------|--------|------------------------|--------------------|------|----------|
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|0.010601719197707736|111   |10359     |
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|サービス概要 - Treasure Data  |0.021453590192644482|49    |2235      |
|お問い合わせ確認  |リソース - Treasure Data|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|0.11455108359133127 |37    |286       |
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|リソース - Treasure Data    |0.08695652173913043 |34    |357       |
|お問い合わせ確認  |お問い合わせ - Treasure Data|お問い合わせ確認 - Treasure Data|0.5                 |30    |30        |
|お問い合わせ確認  |サービス概要 - Treasure Data|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|0.030335861321776816|28    |895       |
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|お問い合わせ - Treasure Data  |0.1891891891891892  |21    |90        |
|お問い合わせ確認  |企業情報 - Treasure Data|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|0.019853709508881923|19    |938       |
|お問い合わせ確認  |連携するシステム一覧 - Treasure Data|パートナー企業 - Treasure Data |0.06666666666666667 |16    |224       |
|お問い合わせ確認  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|ニュースルーム - Treasure Data |0.09803921568627451 |15    |138       |

### Step6. コンバージョンパス集計（集合版）
個々のコンバージョンパスをそのまま集計する理想的な方法は，そのバリエーションの多さゆえに今回のサンプルデータでは疎な集計となってしまうため，重複するページを排除して順番を統一した「集合」を集計することにします。
```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
),
cv_history AS
(
  SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level 
  )
),
cv_raw_join_table AS
(
  SELECT
    log.td_client_id, cv_id, log.td_title, cv_title, log.time AS time, cv_history.time AS cv_time
  FROM sample_accesslog log
  JOIN cv_history
  ON log.td_client_id = cv_history.td_client_id
  AND log.time <= cv_history.time       
),
cv_organized_join_table AS
(
  SELECT
    td_client_id, td_title, time,
    MIN(cv_id) AS cv_id,
    MIN(cv_time) AS cv_time,
    IF((MIN(cv_time)-time)=0,1,0) AS cv_flag,
    MIN(cv_title) AS cv_title
  FROM cv_raw_join_table
  GROUP BY td_client_id, td_title, time
),
cv_path AS
(
  SELECT
    td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY td_client_id, cv_id ORDER BY time) AS node_id,
    cv_id, cv_flag, td_title, cv_title, time, cv_time
  FROM cv_organized_join_table
  ORDER BY td_client_id, cv_id, node_id
)

SELECT cv_title, td_title_trans, MAX(path_length) AS path_length, COUNT(1) AS cnt
FROM
(
  SELECT cv_title, td_client_id, cv_id, 
    ARRAY_JOIN(ARRAY_DISTINCT(ARRAY_AGG(td_title)),' - ') AS td_title_trans, 
    CARDINALITY(ARRAY_DISTINCT(ARRAY_AGG(td_title))) AS path_length
--    ARRAY_JOIN(ARRAY_AGG(td_title),' -> ') AS td_title_trans, 
--    CARDINALITY(ARRAY_AGG(td_title)) AS path_length
  FROM cv_path
  GROUP BY cv_title, td_client_id, cv_id
)
GROUP BY cv_title, td_title_trans
ORDER BY cv_title DESC, cnt DESC
```
|cv_title  |td_title_trans|path_length             |cnt                 |
|----------|--------------|------------------------|--------------------|
|資料ダウンロード  |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data - 資料ダウンロードを受け付けました|3                       |485                 |
|資料ダウンロード  |資料ダウンロードを受け付けました|1                       |101                 |
|資料ダウンロード  |TD Private Seminar - Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data - 資料ダウンロードを受け付けました|3                       |18                  |
|資料ダウンロード  |トレジャーデータ会社概要資料ダウンロードを受け付けました|1                       |13                  |
|資料ダウンロード  |TD Private Seminar - 資料ダウンロードを受け付けました|3                       |8                   |


純粋なコンバージョンパスを集計したい場合は，上記のクエリでコメントアウトした部分を使ってください。

### Step6’. コンバージョンパスごとのCV率（集合版）
先程のコンバージョンリストを集合化したもののCV率を求めたものです。

```sql
WITH master_cv AS
(
  SELECT cv_level, cv_title
  FROM 
  (  VALUES
    (5,'資料ダウンロード'),
    (4,'お問い合わせ確認'),
    (3,'お申し込みを受け付けました')
  ) AS t(cv_level,cv_title)
),
cv_history AS
(
  SELECT td_client_id, cv_title, td_title, ROW_NUMBER() OVER( PARTITION BY td_client_id ORDER BY time) AS cv_id, time
  FROM
  (
    SELECT td_client_id, td_title, cv_title, cv_level, sample_accesslog.time
    FROM sample_accesslog, master_cv
    WHERE REGEXP_LIKE(td_title, cv_title)
    AND 3 <= cv_level 
  )
),
cv_raw_join_table AS
(
  SELECT
    log.td_client_id, cv_id, log.td_title, cv_title, log.time AS time, cv_history.time AS cv_time
  FROM sample_accesslog log
  JOIN cv_history
  ON log.td_client_id = cv_history.td_client_id
  AND log.time <= cv_history.time       
),
cv_organized_join_table AS
(
  SELECT
    td_client_id, td_title, time,
    MIN(cv_id) AS cv_id,
    MIN(cv_time) AS cv_time,
    IF((MIN(cv_time)-time)=0,1,0) AS cv_flag,
    MIN(cv_title) AS cv_title
  FROM cv_raw_join_table
  GROUP BY td_client_id, td_title, time
),
cv_path AS
(
  SELECT
    td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY td_client_id, cv_id ORDER BY time) AS node_id,
    cv_id, cv_flag, td_title, cv_title, time, cv_time
  FROM cv_organized_join_table
  ORDER BY td_client_id, cv_id, node_id
),
cv_list AS
(
  SELECT cv_title, td_title_set, MAX(path_length) AS path_length, COUNT(1) AS cnt
  FROM
  (
    SELECT cv_title, td_client_id, cv_id, 
      ARRAY_SORT(ARRAY_DISTINCT(ARRAY_AGG(td_title))) AS td_title_set,
      CARDINALITY(ARRAY_DISTINCT(ARRAY_AGG(td_title))) AS path_length
    FROM cv_path
    GROUP BY cv_title, td_client_id, cv_id
  )
  GROUP BY cv_title, td_title_set
),
non_cv_path AS
(
  SELECT
    log.td_client_id, 
    ROW_NUMBER()OVER(PARTITION BY log.td_client_id ORDER BY time) AS node_id,
    1 AS cv_id, log.td_title, log.time AS time
  FROM sample_accesslog log
  LEFT OUTER JOIN ( SELECT td_client_id FROM cv_history GROUP BY td_client_id ) cv_history
  ON cv_history.td_client_id IS NULL
),
non_cv_list AS
(
  SELECT td_title_set, MAX(path_length) AS path_length, COUNT(1) AS cnt
  FROM
  (
    SELECT td_client_id,
      ARRAY_SORT(ARRAY_DISTINCT(ARRAY_AGG(td_title))) AS td_title_set,
      CARDINALITY(ARRAY_DISTINCT(ARRAY_AGG(td_title))) AS path_length
    FROM non_cv_path
    GROUP BY td_client_id
  )
  GROUP BY td_title_set
)

SELECT cv_title, t1.td_title_set, path_length, 1.0*MAX(t1.cnt)/(MAX(t1.cnt)+SUM(t2.cnt)) AS cv_ratio, MAX(t1.cnt) AS cv_cnt, SUM(t2.cnt) AS non_cv_cnt
FROM cv_list t1
JOIN ( SELECT td_title_set, cnt FROM non_cv_list ) t2
ON CARDINALITY(ARRAY_EXCEPT(t1.td_title_set,t2.td_title_set)) = 1
AND 1 < t1.path_length 
GROUP BY cv_title, t1.td_title_set, path_length 
ORDER BY cv_title DESC, cv_ratio DESC
```
|cv_title  |td_title_set|path_length             |cv_ratio            |cv_cnt|non_cv_cnt|
|----------|------------|------------------------|--------------------|------|----------|
|資料ダウンロード  |["TD Private Seminar", "Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data", "ダウンロードを受け付けました", "トレジャーデータ会社概要ダウンロード", "資料ダウンロードを受け付けました", nil]|6                       |0.5                 |1     |1         |
|資料ダウンロード  |["TD Private Seminar", "Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data", "ダウンロードを受け付けました", "トレジャーデータ会社概要ダウンロード", "トレジャーデータ概要ビデオ", "資料ダウンロードを受け付けました"]|6                       |0.3333333333333333  |1     |2         |
|資料ダウンロード  |["Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data", "ダウンロードを受け付けました", "トレジャーデータ会社概要ダウンロード", "登録完了:", "資料ダウンロードを受け付けました", nil]|6                       |0.3333333333333333  |1     |2         |
|資料ダウンロード  |["TD Private Seminar", "Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data", "ダウンロードを受け付けました", "資料ダウンロードを受け付けました", nil]|5                       |0.14285714285714285 |1     |6         |
|資料ダウンロード  |["TD Private Seminar", "ダウンロードを受け付けました", "トレジャーデータ会社概要ダウンロード", "資料ダウンロードを受け付けました"]|4                       |0.14285714285714285 |1     |6         |
|資料ダウンロード  |["Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data", "ダウンロードを受け付けました", "トレジャーデータ概要ビデオ", "資料ダウンロードを受け付けました"]|4                       |0.05555555555555555 |1     |17        |


non_cv_title_setとcv_title_setを比較するには，前者にcvタイトル以外のタイトルをすべて含むset（その他も含んで可）の合計を後者と比較します。以下のように結果がcv_titleの1要素のみとなるものがこの条件です。
```sql
CARDINALITY(ARRAY_EXCEPT(t1.td_title_list,t2.td_title_list)) = 1
```
