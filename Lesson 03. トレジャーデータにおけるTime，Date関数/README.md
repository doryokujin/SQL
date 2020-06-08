# Lesson 03. トレジャーデータにおけるTime，Date関数

## TD_TIME_FORMAT
```sql
string TD_TIME_FORMAT(long unix_timestamp,
                      string format
                      [, string timezone = 'UTC'])
```

TD_TIME_FORMATは，第1引数で与えられたunixtimeを，formatで定めた日付文字列に変換するクエリです。なお，トレジャーデータのtimeカラムには常にUNIX時間が格納されます。
日付文字列のformatについては以下の一覧表を参照してください。月（Month）を表すには大文字のM，分（Minute）を表すには小文字のmを使用することに注意してください。

|Syntax    |Date or Time Component|Presentation|Examples                           |
|----------|----------------------|------------|-----------------------------------|
|G         |Era designator        |Text        |AD                                 |
|yyyy      |Year                  |Year        |1996                               |
|yy        |Year                  |Year（2 digits）|96                                 |
|MMMM      |Month in year         |Month long name|July                               |
|MMM       |Month in year         |Month short name|Jul                                |
|MM, M     |Month in year         |Month number|7                                  |
|ww，w      |Week in year          |Number      |6                                  |
|DDD, DD, D|Day in year           |Number      |189                                |
|dd, d     |Day in month          |Number      |10                                 |
|EEEE      |Day in week           |Text        |Tuesday                            |
|E，EEE     |Day in week           |Text（short form）|Tue                                |
|a         |Am/pm marker          |Text        |PM                                 |
|HH，H      |Hour in day（0-23）     |Number      |0                                  |
|kk，k      |Hour in day（1-24）     |Number      |24                                 |
|KK，K      |Hour in AM/PM（0-11）   |Number      |0                                  |
|hh，h      |Hour in AM/PM（1-12）   |Number      |12                                 |
|mm，m      |Minute in hour        |Number      |30                                 |
|ss，s      |Second in minute      |Number      |55                                 |
|SSS，SS，S  |Millisecond           |Number      |978                                |
|zzzz      |Time zone             |Zone long name|Pacific Standard Time, or GMT+01:00|
|z         |Time zone             |Zone short name|PST, or GMT+01:00                  |
|Z         |Time zone             |Zone offset |-800                               |
|u         |Day number of week（1-7）|Number      |1（for Monday）                      |

いくつか例を見てみましょう。
```sql
SELECT 
  TD_TIME_FORMAT(time,'yyyy-MM-dd','JST') AS day1,
  TD_TIME_FORMAT(time,'yyyy/MM/dd','JST') AS day2,
  TD_TIME_FORMAT(time,'yyyy/M/d'  ,'JST') AS day3,
  TD_TIME_FORMAT(time,'yyyy-MM-dd HH:mm:ss'      ,'JST') AS time1,
  TD_TIME_FORMAT(time,'yyyy-MM-dd hh:mm:ss a'    ,'JST') AS time2,
  TD_TIME_FORMAT(time,'yyyy-MM-dd HH:mm:ss:SSS'  ,'JST') AS time3,
  TD_TIME_FORMAT(time,'yyyy-MM-dd HH:mm:ss:SSS z','JST') AS time4
FROM ( VALUES 1466409507 ) AS t(time)
```

上記のクエリの結果は以下のようになります。

```sql
day1, 2016-06-20
day2, 2016/06/20
day3, 2016/6/20
time1, 2016-06-20 16:58:27
time2, 2016-06-20 04:58:27 PM
time3, 2016-06-20 16:58:27+09:00
time4, 2016-06-20 16:58:27 GMT+09:00
time5, 2016-06-20 16:58:27:000
```
day3は，月日が1桁の場合に0が省略されるので文字列の長さにばらつきが出てしまい，後々パースしにくくなります。
今度は実データを扱う例を示します。sample_access_logに入っているtimeカラムを変換してみましょう。

```sql
SELECT
  TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d1,
  TD_TIME_FORMAT(time, 'yyyy-MM-dd HH:00:00', 'JST') AS d2,
  TD_TIME_FORMAT(time, 'yyyy-MM-dd HH:mm:00', 'JST') AS d3,
  TD_TIME_FORMAT(time, 'yyyy-MM-dd HH:mm:ss', 'JST') AS d4
FROM sample_accesslog
ORDER BY time DESC
LIMIT 10
```

それぞれのtimeが何曜日に該当するかは，週の始まりである月曜日から何番目かを返す uというformatにて計算できます。以下のクエリでは，その結果を英語で曜日を表す文字列に変換し，かつユニークな日付と曜日のみ列挙しています。

```sql
SELECT DISTINCT
  TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d,
  CASE TD_TIME_FORMAT(time, 'u', 'JST')
    WHEN '1' THEN 'Monday'
    WHEN '2' THEN 'Tuesday'
    WHEN '3' THEN 'Wednesday'
    WHEN '4' THEN 'Thursday'
    WHEN '5' THEN 'Friday'
    WHEN '6' THEN 'Saturday'
    WHEN '7' THEN 'Sunday'
    ELSE 'Error'
  END AS day_of_the_week
FROM sample_accesslog
ORDER BY d
```

## TD_TIME_RANGE
```sql
boolean TD_TIME_RANGE(int/long unix_timestamp,                            
                      int/long/string start_time,
                      int/long/string end_time
                      [, string default_timezone = 'UTC'])
```

TD_TIME_RANGEは，関数の引数としてstart_time（始点）とend_time（終点）を定めることにより，この範囲内に該当するレコードのみを抽出する条件節に使用される関数です。
1番目の引数には，レコード内にunixtimeが格納されているカラムを指定します。基本的にはtimeカラムとなるでしょう。2番目の「始点」と3番目の「終点」をうまく指定できれば，この関数は大変便利に使用できます。

### A. 特定の時間軸での範囲指定
特定の時間範囲内のレコードを抽出するには，下記のようなクエリを書きます。
```sql
SELECT ... WHERE TD_TIME_RANGE(time, '2018-01-01', '2018-02-01','JST') --月
SELECT ... WHERE TD_TIME_RANGE(time, '2018-01-01', '2018-01-08','JST') --週
SELECT ... WHERE TD_TIME_RANGE(time, '2018-01-01', '2018-01-02','JST') --日
SELECT ... WHERE TD_TIME_RANGE(time, '2018-01-01 00:00:00', '2018-01-01 01:00:00','JST') --時間
SELECT ... WHERE TD_TIME_RANGE(time, '2018-01-01 00:00:00', '2018-01-01 00:01:00','JST') --分
```
いくつか注意点を述べておきます。
#### 「始点」は含まれ，「終点」は含まれない
上記のクエリ1行目の月の範囲指定を見て，「この句では2月1日も含まれるのでは？」と思う人がいるかもしれません。答えはNOです。始点は含まれますが，終点の時間は含まれません。始点（00:00:00）ちょうどから，終点ぎりぎり（23:59:59.999）までの時間までが範囲に含まれるのです。数学の表記で書くと，\[始点，終点)ということです。逆に，終点を含まない仕様に対して簡易かつ厳密に月単位の集計を行うことはできません。

#### 「始点」と「終点」にはunixtimeに加えて様式に従った日付文字列が指定可能
先程の例でも，「始点」と「終点」の明確な時間を日付文字列として記述しました。実際の定時実行される自動クエリなどでは，ここにTD_DATE_TRUNC関数などが入ってくることになり，その場合は値としてunixtimeが指定されていることになります。他の関数では日付文字列がNGとなるものもあるので注意してください。

#### TD_TIME_RANGEを使う場合と使わない場合とではパフォーマンスが圧倒的に異なる
トレジャーデータをはじめとしたビッグデータプラットフォームでは，大量のレコードが時間単位でパーティショニングして別々の場所に置かれています。TD_TIME_RANGE関数を使用すると，指定された範囲外の時間のレコードはそもそもアクセスされません。この時，「TimeIndexが効いている」と呼ばれます。特に，過去大量に蓄積されたデータについては，すべてのパーティションを見にいくか，特定のパーティションだけを見にいくかで，圧倒的にクエリ処理速度が異なってきます。

TD_TIME_RANGEを用いたWHERE節のみが，FROMで見にいくテーブルのパーティションを限定できます。
パーティションの効率化が有効でない時間範囲指定としては，例えば下記のようなものがあります。
```sql
WHERE start_time <= time AND time < end_time --NG
```
また，TD_TIME_RANGEを使っても，下記のようにルールに基づかない記述の仕方をしてしまうと，同様に効率化が無効になります。
```sql
SELECT ... WHERE TD_TIME_RANGE(time, '2013-01-01','JST') --NG
SELECT ... WHERE TD_TIME_RANGE(time, '2013-01-01', NULL, 'JST') --OK
```
なお，この関数では「終点」の記述を省くことで「始点以降」という範囲ができますが，書き方に注意が必要です。引数を省略するのではなくNULLで埋めておかないと効きません。
加えて，トレジャーデータUDF以外の計算で割り算やFLOAT型になるものを引数内で使うと効率化が無効になります。
```sql
SELECT ... WHERE TD_TIME_RANGE(time, '2013-01-01',
                               TD_TIME_ADD('2013-01-01', '1', 'JST')) --OK
SELECT ... WHERE TD_TIME_RANGE(time, TD_SCHEDULED_TIME() / 86400 * 86400)) --NG
SELECT ... WHERE TD_TIME_RANGE(time, 1356998401 / 86400 * 86400)) --NG
```
timeカラムから2016年5月分の範囲のレコードを取得してみましょう。終点に指定した日は含まれないので，その月が何日あるのかを気にする必要はありません。
```sql
SELECT
  TD_TIME_FORMAT(time, 'yyyy-MM-dd HH:mm:ss', 'JST') AS d
FROM sample_accesslog
WHERE TD_TIME_RANGE(time, '2016-05-01','2016-06-01','JST')
LIMIT 10
```
実際に取得した日付の範囲にすべてのレコードが収まっているか確認してみます。
```sql
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE TD_TIME_RANGE(time, '2016-05-01','2016-06-01','JST')
```
### B. 「始点以降」/「終点未満」
注：「以降」という言葉はその点を含み，「未満」という言葉はその点を含みません。
多くの分析では，始点と終点が定まっておらず，ある日「以降」であったり，昨日まで（今日未満）といった集計を行いたいケースがよくあります。この場合，始点か終点にNULLを入れることで目的を達成できます。
```sql
SELECT ... WHERE TD_TIME_RANGE(time, '2018-01-01', NULL, 'JST') --始点以降
SELECT ... WHERE TD_TIME_RANGE(time, NULL, '2018-01-01', 'JST') --終点未満
SELECT ... WHERE TD_TIME_RANGE(time, NULL, '2018-01-01 JST')    --始点以降
```
上記の3番目の例は間違いやすい記述方式なので，本書では一貫して1番目と2番目のNULLを含む記述方式を使用します。
2016年5月未満および2016年5月以降の日付範囲であれば，それぞれ以下のように指定します。
#### 2016年5月未満
```sql
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE TD_TIME_RANGE(time, NULL, '2016-05-01','JST')
```
#### 2016年5月以降
```sql
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE TD_TIME_RANGE(time, '2016-05-01',NULL,'JST')
```

## TD_DATE_TRUNC
```sql
long TD_DATE_TRUNC(string unit,
                   long unix_timestamp
                   [, string default_timezone = 'UTC'])
```

TD_DATE_TRUNC関数は，指定された時間をあらゆる時間単位で「切り捨て」てくれる便利な関数です。TD_DATE_TRUNCは，主にTD_TIME_RANGE内の「始点」や「終点」で設定されます。

TD_DATE_TRUNCの使い道は，TD_TIME_FORMATと同じく，timeをある程度の粒度に揃えてグルーピングすることですが，TD_TIME_FORMATが視認性の良い時間文字列を返すのでSELECT節で出力に使われるのに対し，TD_DATE_TRUNCはtimestampを返すので，TD_TIME_RANGEの引数として使っても，「TimeIndexの効用」（必要なパーティションしかアクセスしない）を保ったまま時間の計算を行うのに使われます。

2番目の引数はunixtimeであり，日付文字列ではいけません。一般的にこの2番目の引数に入る変数には下記の2種類があります。
- レコード内のtimeカラム
- TD_SCHEDULED_TIME()の値
TD_SCHEDULED_TIMEについては後述しますが，外側から任意の時間を挿入することができる便利な変数（関数）です。
また，1番目の引数には日付ユニットとして以下の種類を指定できます。
```sql
'minute'
'hour'
'day'
'week'
'month'
'quarter'
'year'
```
今，2番目の引数がtimeカラムの値で「2018-11-23 11:11:00」というunixtimeであったとします。このとき，この関数によってどんな値が返ってくるのでしょうか？ 以下の例では，それぞれのクエリ結果をコメントで示しています。結果はunixtimeであり、時間文字列ではありません！
```sql
SELECT TD_DATE_TRUNC('hour',  time, 'JST') FROM ... 
  --時間の始まり： 2018-11-23 11:00:00
SELECT TD_DATE_TRUNC('day',   time, 'JST') FROM ... 
  --日の始まり： 2018-11-23 00:00:00
SELECT TD_DATE_TRUNC('week',  time, 'JST') FROM ... 
  --週の始まり： 2018-11-19 00:00:00 （週始まりは月曜日！）
SELECT TD_DATE_TRUNC('month', time, 'JST') FROM ... 
  --月の始まり： 2018-11-01 00:00:00
SELECT TD_DATE_TRUNC('year',  time, 'JST') FROM ... 
  --年の始まり： 2018-01-01 00:00:00
```
このように，TD_DATE_TRUNCは半端な時間が格納されたtimeカラムの値を指定した日付ユニットの「始点」に合わせて切り捨ててくれる便利な関数です。

## TD_TIME_STRING
```sql
string TD_TIME_STRING(unix_timestamp,
                      '(interval string)'
                      [, time zone])
```
TD_TIME_STRING関数は，TD_DATE_TRUNCとTD_TIME_FORMATを組合せた便利な関数です。ただし，返ってくる文字列は特定の形式かつタイムゾーン付きであることに注意してください。
結果は，第2引数で指定した「時間を示す文字」で切り捨てられます。その文字の末尾に「!」を付けることで，それより詳細な日時の文字列が表示されなくなります。
```sql
SELECT 
  TD_TIME_STRING(time,'y','JST') AS time_y,
  TD_TIME_STRING(time,'M','JST') AS time_M,
  TD_TIME_STRING(time,'w','JST') AS time_w,
  TD_TIME_STRING(time,'d','JST') AS time_d,
  TD_TIME_STRING(time,'h','JST') AS time_h,
  TD_TIME_STRING(time,'m','JST') AS time_m,
  TD_TIME_STRING(time,'s','JST') AS time_s,

  TD_TIME_STRING(time,'y!','JST') AS time_y2,
  TD_TIME_STRING(time,'M!','JST') AS time_M2,
  TD_TIME_STRING(time,'w!','JST') AS time_w2,
  TD_TIME_STRING(time,'d!','JST') AS time_d2,
  TD_TIME_STRING(time,'h!','JST') AS time_h2,
  TD_TIME_STRING(time,'m!','JST') AS time_m2,
  TD_TIME_STRING(time,'s!','JST') AS time_s2
FROM ( VALUES 1576110613 ) AS t(time)
```
```sql
time_y,  2019-01-01 00:00:00+0900
time_M,  2019-12-01 00:00:00+0900
time_w,  2019-12-09 00:00:00+0900
time_d,  2019-12-12 00:00:00+0900
time_h,  2019-12-12 09:00:00+0900
time_m,  2019-12-12 09:30:00+0900
time_s,  2019-12-12 09:30:13+0900
time_y2, 2019
time_M2, 2019-12
time_w2, 2019-12-09
time_d2, 2019-12-12
time_h2, 2019-12-12 09
time_m2, 2019-12-12 09:30
time_s2, 2019-12-12 09:30:13
```

## TD_TIME_ADD
```sql
long TD_TIME_ADD(int/long/string time,
                 string duration
                 [, string default_timezone = 'UTC'])
```
TD_TIME_ADD関数は，第1引数に指定されたunixtimeまたは時間文字列に対して，第2引数に指定された時間（duration）の加減算を行ってくれる便利な関数です。timeカラムがunixtimeであれば単純に秒数（例えば1日分だと60*60*24秒）の加減算で計算できますが，以下の理由から，できる限りこの関数を利用することを強く推奨します。
- 時間文字列に対しても加減算ができる
- どの時間軸で加減算しているのかが明確で，コードの可読性が上がる
- 「1時間30分前（'-1h30m'）」といった半端な時間の加減算にも対応している

durationのバリエーションには以下のようなものがあります。ここで，「週」単位より長い「月」「年」といった単位は存在しないことに注意してください。
```sql
'Nw' ：N週後 (e.g. '1w', '4w', '48d')
'-Nw'：N週前 (e.g. '-1w', '-2w', '-48d')
'Nd' ：N日後 (e.g. '1d', '2d', '30d')
'-Nd'：N日前 (e.g. '-1d', '-2d', '-30d')
'Nh' ： N時間後 (e.g. '1h', '2h', '48h')
'-Nh'：N時間前 (e.g. '-1h', '-2h', '-48h')
'Nm' ：N分後 (e.g. '1m', '2m', '90m')
'-Nm'：N分前 (e.g. '-1m', '-2m', '-90m')
'Ns' ：N秒後 (e.g. '1s', '2s', '90s')
'-Ns'：N秒前 (e.g. '-1s', '-2s', '-90s')
'NdMhLs'：N日M時間L分後 (e.g. '1d6h30m', '-2d3h', '-1h30m')
```
```sql
SELECT
  TD_TIME_FORMAT( time, 'yyyy-MM-dd HH:mm:ss', 'JST') AS time0,
  TD_TIME_FORMAT( TD_TIME_ADD(time,  '1d','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time1,  --1日後
  TD_TIME_FORMAT( TD_TIME_ADD(time, '-1d','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time2,  --1日前
  TD_TIME_FORMAT( TD_TIME_ADD(time,  '1h','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time3,  --1時間後
  TD_TIME_FORMAT( TD_TIME_ADD(time, '-1h','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time4,  --1時間前
  TD_TIME_FORMAT( TD_TIME_ADD(time, '30m','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time5, --30分後
  TD_TIME_FORMAT( TD_TIME_ADD(time,'-30m','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time6, --30分前
  TD_TIME_FORMAT( TD_TIME_ADD(time, '10s','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time7, --10秒後
  TD_TIME_FORMAT( TD_TIME_ADD(time,'-10s','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time8, --10秒前
  TD_TIME_FORMAT( TD_TIME_ADD(time,'1d12h30m','JST'), 'yyyy-MM-dd HH:mm:ss', 'JST') AS time9 --1日12時間30分後
FROM ( VALUES 1576110613 ) AS t(time)
```

## TD_SCHEDULED_TIME
```sql
long TD_SCHEDULED_TIME()
```

TD_SCHEDULED_TIMEは，そのクエリが実行されたときのunixtimeを返す関数です。バッチクエリ（定時実行クエリ）として登録されたクエリに対しては，そのクエリの実行スケジュール時間（実際に実行された時間と異なる）が入ります。引数はありませんが，「()」を忘れるとエラーになるので注意してください。

### A. バッチクエリで使用する場合
トレジャーデータではクエリをスケジュール登録して任意の時間単位で定時実行すること（バッチクエリ）が可能です。

バッチクエリ内でTD_SCHEDULED_TIMEを使用した場合，クエリが実行された時刻が，この関数の値としてクエリ内で使用されることになります。例えば，dailyを指定すれば毎日日付が変わる時間「00:00:00」に実行されるバッチクエリになり，このときTD_SCHEDULED_TIME()の値には「2018-11-24 00:00:00」のunixtimeが入ることになります。
ここで，以下のポイントを押さえておいてください。

#### スケジュールされた時間と実際に実行された時間は異なる

ちょうど日付が変わった時点でクエリを実行するようにスケジュールしていたとしても，実際には他の処理の混み具合などによって，実際に実行される時間は遅延する場合が多くあります。例えば，dailyで「00:00:00」での実行をスケジュールしたのに，実際の処理は遅延によって「06:00:00」に実行されてしまった場合，次のどちらのunixtimeが入るのでしょうか？

- 「スケジュールされた（予定通りの）時間 = 00:00:00」
- 「実際に実行された時間 = 06:00:00」

答えはaです。処理がどんなに遅延したとしても，この値は変わりません。
また，bとなるのは NOW関数を使ったときで，この2つの関数の挙動が異なることの理解はとても重要です。

#### TD_SCHEDULED_TIME()にはいつでもスケジュール時間が入り，実際の実行時の時間が入るNOW()とは違う

すなわち，今の例ではそれぞれ次のような結果になります。
- TD_SCHEDULED_TIME()：00:00:00
- NOW()：06:00:00

### B. アドホッククエリで使用する場合
アドホッククエリとは，その場で実行して結果を待つタイプのクエリを意味します。先程と同じクエリで考えましょう。トレジャーデータでは，TD_SCHEDULED_TIMEを含んだクエリをその場で実行しようとすると，カレンダーが表示されます。

これは非常に便利な表示で，自分でTD_SCHEDULED_TIMEに入れたい日付のunixtimeを指定することができます。これがNOW関数であれば，自動的に実行した際のunixtimeが挿入されます。このアドホッククエリで用いるTD_SCHEDULED_TIMEは，過去の月次処理をやり直す場合などに大きな効力を発揮します。

## テンプレート集：「日次」「週次」「月次」（TD_TIME_RANGE編）

ここで，今まで登場した関数を使って，過去の任意の「日／週／月」での「日次／週次／月次」処理を考えてみましょう。前提として，基準日を以下とします。
基準日 = TD_SCHEDULED_TIME() = '2016-05-22 01:00:00'
そして，確認したい集計範囲は以下であるとします。（\[始点，終点)という表記は，「始点を含み，終点を含まない」を意味します。）

```sql
# 基準日 = '2016-05-22 01:00:00'
[2016-05-21 00:00:00, 2016-05-22 00:00:00）  --日次
[2016-05-09 00:00:00, 2016-05-16 00:00:00）  --週次
[2016-04-01 00:00:00, 2016-05-01 00:00:00)   --月次
```
これを今までの関数で実装する場合には，以下のように書くことになります。

### 日次テンプレート
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
TD_TIME_RANGE(
  time,  
  TD_TIME_ADD(TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST'),'-1d'),
  TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST')
 ) --日次 [2016-05-21 00:00:00, 2016-05-22 00:00:00）
```

取得した結果の日次の範囲が正しいかの確認クエリ
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE TD_TIME_RANGE(
  time,  
  TD_TIME_ADD(TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST'),'-1d'),
  TD_DATE_TRUNC('day', TD_SCHEDULED_TIME(), 'JST')
 )  --日次 [2016-05-21 00:00:00, 2016-05-22 00:00:00）
```

### 週次テンプレート
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
TD_TIME_RANGE(
  time,  
  TD_TIME_ADD(TD_DATE_TRUNC('week', TD_SCHEDULED_TIME(), 'JST'),'-1w'),
  TD_DATE_TRUNC('week', TD_SCHEDULED_TIME(), 'JST')
 ) --週次 [2016-05-09 00:00:00, 2016-05-16 00:00:00）
```
取得した結果の週次の範囲が正しいかの確認クエリ
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE TD_TIME_RANGE(
  time,  
  TD_TIME_ADD(TD_DATE_TRUNC('week', TD_SCHEDULED_TIME(), 'JST'),'-1w'),
  TD_DATE_TRUNC('week', TD_SCHEDULED_TIME(), 'JST')
 ) --週次 [2016-05-09 00:00:00, 2016-05-16 00:00:00）
```

### 月次テンプレート
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
TD_TIME_RANGE(
  time,
  TD_DATE_TRUNC(
    'month', TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(),'JST')-1, 'JST'
  ),                                                
  TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(), 'JST')
)  --月次 [2016-04-01 00:00:00, 2016-05-01 00:00:00)
```
取得した結果の月次の範囲が正しいかの確認クエリ
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE TD_TIME_RANGE(
  time,
  TD_DATE_TRUNC(
    'month', TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(),'JST')-1, 'JST'
  ),                                                
  TD_DATE_TRUNC('month', TD_SCHEDULED_TIME(), 'JST')
)  --月次 [2016-04-01 00:00:00, 2016-05-01 00:00:00)
```

## Prestoの日付関数で同様のクエリを再現する
参考までに，純粋なPrestoの日付関数を使ったテンプレートクエリも紹介します。
Prestoのdate_add関数では，なんと「month」による加減算ができるので，月次に関しては統一性の意味で便利かもしれません。

トレジャーデータに格納されているtimeカラムやTD_SCHEDULED_TIME()の値はunixtimeのため変換が必要です。また，既に説明したように，TD_SCHEDULED_TIME()の代わりに NOW()を使ってしまうと挙動が異なってしまいます。

```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
WHERE 
    date_add('day', -1, date_trunc('day', from_unixtime(TD_SCHEDULED_TIME(),'Asia/Tokyo')))
 <= from_unixtime(time,'Asia/Tokyo')
AND from_unixtime(time,'Asia/Tokyo') 
 <  date_trunc('day', from_unixtime(TD_SCHEDULED_TIME(),'Asia/Tokyo')) 
  --日次 [2016-05-21 00:00:00, 2016-05-21 00:00:00）

WHERE 
    date_add('week', -1, date_trunc('week', from_unixtime(TD_SCHEDULED_TIME(),'Asia/Tokyo')))
 <= from_unixtime(time,'Asia/Tokyo')
AND from_unixtime(time,'Asia/Tokyo')
 <  date_trunc('week', from_unixtime(TD_SCHEDULED_TIME(),'Asia/Tokyo')) 
  --週次 [2016-05-09 00:00:00, 2016-05-15 00:00:00）

WHERE
    date_add('month', -1, date_trunc('month', from_unixtime(TD_SCHEDULED_TIME(),'Asia/Tokyo')))
 <= from_unixtime(time,'Asia/Tokyo')
AND from_unixtime(time,'Asia/Tokyo')
 <  date_trunc('month', from_unixtime(TD_SCHEDULED_TIME(),'Asia/Tokyo')) 
  --月次 [2016-04-01 00:00:00, 2016-05-01 00:00:00)
```

## TD_INTERVAL

TD_INTERVALは近年彗星のごとく登場した大変便利な関数です。それまではTD_TIME_RANGEとTD_DATE_TRUNCを駆使して行っていた過去の月次処理が，この関数のみでできるようになったのです。
```sql
boolean TD_INTERVAL(int/long time,
                    string interval_string,
                    [, string default_timezone = 'UTC'])
```

基準日をTD_SCHEDULED_TIMEを定める必要がありますが，これを'2018-11-23 11:11:00'としたとき，この関数は，まずinterval_stringで指定した時間の単位でTRUNCした後，以下の時間範囲内でのレコードを返します。カレンダーを参照しながら確認してください。

```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
-- ！SELECT 句に TD_SCHEDULED_TIME() を入れることを忘れない！
TD_INTERVAL(time,  '1d', 'JST') 
  --今日 [2016-05-22 00:00:00, 2016-05-23 00:00:00)
TD_INTERVAL(time, '-1d', 'JST') 
  --昨日 [2016-05-21 00:00:00, 2016-05-22 00:00:00)
TD_INTERVAL(time, '-7d', 'JST') 
  --今週 [2016-05-16 00:00:00, 2016-05-23 00:00:00)
TD_INTERVAL(time,  '1w', 'JST') 
  --今週 [2016-05-16 00:00:00, 2016-05-23 00:00:00)
TD_INTERVAL(time, '-1w', 'JST') 
  --前週 [2016-05-09 00:00:00, 2016-05-16 00:00:00)
TD_INTERVAL(time,  '1M', 'JST') 
  --今月 [2016-05-01 00:00:00, 2016-06-01 00:00:00)
TD_INTERVAL(time, '-1M', 'JST') 
  --前月 [2016-04-01 00:00:00, 2016-05-01 00:00:00)
TD_INTERVAL(time, '-2M', 'JST')
  --前月+前々月 [2016-04-01 00:00:00, 2016-06-01 00:00:00)
```

週の始まりは月曜日であることに注意してください。

### 今日（TRUNCした日から+1d）
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time,  '1d', 'JST') 
  --今日 [2016-05-22 00:00:00, 2016-05-23 00:00:00)
```

### 昨日（TRUNCした日から-1d）
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time, '-1d', 'JST') 
  --昨日 [2016-05-21 00:00:00, 2016-05-22 00:00:00)
```

### 今週（TRUNCした週から+1w）
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time,  '1w', 'JST') 
  --今週 [2016-05-16 00:00:00, 2016-05-23 00:00:00)
```

### 先週（TRUNCした週から-1w）
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time, '-1w', 'JST') 
  --前週 [2016-05-09 00:00:00, 2016-05-16 00:00:00)
```

### 今月（TRUNCした月から+1M）
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time,  '1M', 'JST') 
  --今月 [2016-05-01 00:00:00, 2016-06-01 00:00:00)
```

### 前月（TRUNCした月から-1M）（データが2016-04-23までであることに注意）
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time, '-1M', 'JST') 
  --前月 [2016-04-01 00:00:00, 2016-05-01 00:00:00)
```

### 前月+前々月（TRUNCした月から-2M）（2016-07-22の日付に変更していることに注意）
```sql
--TD_SCHEDULED_TIME() = '2016-07-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time, '-2M', 'JST')
  --前月+前々月 [2016-05-01 00:00:00, 2016-07-01 00:00:00)
```

TD_INTERVALは，TD_SCHEDULED_TIMEで指定した基準日をまずTRUNCして日（週，月）の始まりの時間に整えたうえで，そこを起点にinterval_stringで指定した数字分（マイナスなら過去方向，プラスなら未来方向）だけ範囲を定めます。

### 重要な注意点
TD_INTERVALの第1引数にはTD_SCHEDULED_TIMEを入れることができません（エラーとなります）。また，SELECT節の中にTD_SCHEDULED_TIMEを入れておかないと，クエリの実行時に基準日指定ができず，実行時の時間（NOW関数の値）が入ってしまいます。

### 強力なオフセット機能
この関数の強力な点は，さらにオフセット（'±Ld/±Nd'，'±Lw/±Nw'，'±LM/±NM'）を使用できることです（L，Nは任意の整数）。これによって，以下を指定できます。
- TRUNCした日から±N日を起点として±L日間
- TRUNCした週から±N週を起点として±L週間
- TRUNCした月から±N月を起点として±L月間
異なるinterval_stringの併用も可能ですが，複雑になるのでここでは説明を割愛します。

```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
--！!SELECT 句に TD_SCHEDULED_TIME() を入れることを忘れない！
TD_INTERVAL(time,  '-1d/-1d', 'JST') 
  --TRUNC後1日前から-1日間 [2016-05-20 00:00:00, 2016-05-21 00:00:00)
TD_INTERVAL(time,  '-7d/-2d', 'JST') 
  --TRUNC後2日前から-7日間 [2016-05-13 00:00:00, 2016-05-20 00:00:00)
TD_INTERVAL(time,  '-7d/-2d', 'JST') 
  --TRUNC後7日前から-7日間 [2016-05-08 00:00:00, 2016-05-15 00:00:00)
TD_INTERVAL(time, '-1w/-1w', 'JST')  
  --TRUNC後1週間前から-1週間 [2016-05-02 00:00:00, 2016-05-09 00:00:00)
TD_INTERVAL(time,  '-1M/-1M', 'JST') 
  --TRUNC後1ヶ月前から-1ヶ月間 [2016-03-01 00:00:00, 2016-04-01 00:00:00)
TD_INTERVAL(time, '-2M/-1M', 'JST')  
  --TRUNC後1ヶ月前から-2ヶ月間 [2016-02-01 00:00:00, 2016-04-01 00:00:00)
```

### 'd'でTRUNC後1日前から-1日間
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time,  '-1d/-1d', 'JST') 
  --TRUNC後1日前から-1日間 [2016-05-20 00:00:00, 2016-05-21 00:00:00)
```

### 'd'でTRUNC後2日前から-7日間
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time,  '-7d/-2d', 'JST') 
  --TRUNC後2日前から-7日間 [2016-05-13 00:00:00, 2016-05-20 00:00:00)
```

### 'd'でTRUNC後7日前から-7日間
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time,  '-7d/-7d', 'JST') 
  --TRUNC後2日前から-7日間 [2016-05-08 00:00:00, 2016-05-15 00:00:00)
```

先程の-7d/-7dは-1w/-1wと変えるだけで結果が異なります。なぜなら後者はTRUNCの単位が週になるからです。前者と同じ結果になる書き方は-1w/-7dとなるのでご注意ください。

### 'w'でTRUNC後1週間前から-1週間
```sql
--TD_SCHEDULED_TIME() = '2016-05-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time, '-1w/-1w', 'JST')  
  --TRUNC後1週間前から-1週間 [2016-05-02 00:00:00, 2016-05-09 00:00:00)
```

### 'M'でTRUNC後1ヶ月前から-1ヶ月間（2016-07-22の日付に変更していることに注意）
```sql
--TD_SCHEDULED_TIME() = '2016-07-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time,  '-1M/-1M', 'JST') 
  --TRUNC後1ヶ月前から-1ヶ月間 [2016-05-01 00:00:00, 2016-06-01 00:00:00)
```

### 'M'でTRUNC後1ヶ月前から-2ヶ月間（2016-07-22の日付に変更していることに注意）
```sql
--TD_SCHEDULED_TIME() = '2016-07-22 01:00 AM' の unixtime
SELECT
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS min_d,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd HH:mm:ss', 'JST') AS max_d
FROM sample_accesslog
WHERE
  TD_INTERVAL(time, '-2M/-1M', 'JST')  
  --TRUNC後1ヶ月前から-2ヶ月間 [2016-04-01 00:00:00, 2016-06-01 00:00:00)
```

### テンプレート集：「日次」「週次」「月次」（TD_INTERVAL編）

先程，TD_TIME_RANGEやTD_DATE_TRUNCなどを駆使して過去の任意の日，週，月での日次，週次，月次処理を実行するテンプレートを紹介しましたが，TD_INTERVALを使えば同様のことが以下のように簡単に書けます。

```sql
# 基準日 = '2018-11-23 11:11:00'
[2018-11-22 00:00:00, 2018-11-23 00:00:00) --日次
[2018-11-12 00:00:00, 2018-11-19 00:00:00) --週次
[2018-10-01 00:00:00, 2018-11-01 00:00:00) --月次
```

```sql
--TD_SCHEDULED_TIME() = '2018-11-23 11:11:00'
--！SELECT 句に TD_SCHEDULED_TIME() を入れることを忘れない！
TD_INTERVAL(time, '-1d', 'JST')  
  --日次 [2018-11-22 00:00:00, 2018-11-23 00:00:00)
TD_INTERVAL(time, '-1w', 'JST')  
  --週次 [2018-11-12 00:00:00, 2018-11-19 00:00:00)
TD_INTERVAL(time, '-1M', 'JST')  
  --月次 [2018-10-01 00:00:00, 2018-11-01 00:00:00)
```

最後に，TD_TIME_RANGEとTD_TINTERVALのそれぞれの長所と短所をまとめます。

### TD_TIME_RANGE型とTD_INTERVAL型に優劣はない
２種類のテンプレートを比べるとTD_INTERVAL型を選択しない理由はないと思われるかもしれませんが，必ずしもそうは言い切れないと筆者は考えています。以降の各章では様々なアクセスログ集計クエリを紹介していきますが，その中でも両者にはケースバイケースで使い分けの場面があることを確認しています。

- TD_TIME_RANGE型では，「始点」と「終点」が明示されているので，クエリ作成者がどの範囲を取りたいのかを汲み取りやすい
- TD_TIME_RANGE型では，「始点」と「終点」のどちらかをNULLとすることで，「始点以上」や「終点未満」という片側が無限大の範囲を取れる
- TD_TIME_RANGE型を真面目に理解しようとすると，様々なDATE関数の知識が深まる
- TD_INTERVAL型では書式の意味やそれがどの範囲を取るかについての正しい理解が必要
- TD_INTERVAL型は「始点」や「終点」に絶対的な時間を入れられない
- TD_INTERVAL型にはオフセット機能があり，TD_TIME_RANGE以上の活用可能性を秘めている

## このLessonで注意したい点まとめ
- TIME，DATE関数を利用する際には，日本時間で扱うために「JST」の指定を忘れずに
- TD_TIME_RANGEによる時間範囲の指定は，始点を含み，終点を含まないことに注意
- TD_TIME_RANGEによる「以降」「未満」の指定では，始点か終点にNULLを明記する
- TD_TIME_ADDで月，年での加減算はできない。週単位まで
- TD_TIME_RANGEおよびTD_INTERVALを必ず使いこなすこと

