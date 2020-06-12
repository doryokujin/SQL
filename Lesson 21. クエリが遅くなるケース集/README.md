# Lesson 21. クエリが遅くなるケース集

今までの章で紹介したクエリでは機能の紹介や分析の目的を果たすことを最優先し，パフォーマンスは二の次にしていました。本章ではクエリのパフォーマンスについて徹底して言及していきます。他の章とは矛盾する（書いていたクエリをパフォーマンスの意味で否定する）こともありうるので，本章は独立したものとして読み進めてください。

## 10GB以上の結果が返ってくるクエリ

10GB以上の結果をダウンロードしたりS3へアップロードしたりする場合には，クエリ結果の出力待ちがボトルネックになり，リソースをうまく使いきれません。この問題の対処として，クエリ冒頭に以下のマジックコメントを記述することで性能が改善します（ORDER BY節がない場合に限り）。
```sql
-- set session result_output_redirect='true'
```

## アカウントで何らかのリソース上限に達している
アカウントにはいくつかのリソース上限が設けられています。
- 同時実行クエリ数
- メモリ消費量
- 分割上限
これらいずれかが上限に達している場合には，トレジャーデータが全体のシステム保護のためにパフォーマンスを低下させることになります。

## データパーティショニングに問題がある
テーブルは基本的にtimeカラム（他のカラムでも設定可）によってパーティショニングされていますが，特定の時間にだけ大量のレコードが格納されてしまっているなど「Data Skew（データの偏り）」と呼ばれる事象が起きている場合，ワーカーノードがその大きなパーティションからのデータスキャンに時間を取られるためにクエリの実行が遅くなってしまいます。

逆に，「Fragmented（断片化された）」パーティション，つまり「非常にたくさんのパーティションに分割されている状態」かつ「個々のパーティションは非常に小さなデータ」の場合には，S3からのデータ取得（トレジャーデータではデータをS3に保存しGETリクエストによって取得します）にレイテンシーが生じて遅くなります。例えば，100万パーティションあるとGETリクエストに20分ほどの時間がかかってしまいます。

以下のクエリにより，個々のパーティションのレコード数の上位を確認して偏りや断片化がないか調べることができます。
```sql
SELECT TD_TIME_FORMAT(time, 'yyyy-MM-dd HH:00:00', 'JST') AS hour, COUNT(1) AS cnt
FROM sample_accesslog
GROUP BY 1
ORDER BY cnt DESC
LIMIT 100
```
|hour      |cnt|
|----------|---|
|2016-06-03 14:00:00|385|
|2016-06-21 15:00:00|300|
|2016-06-21 17:00:00|214|
|2016-06-16 14:00:00|213|
|2016-06-23 16:00:00|202|


## AWSやトレジャーデータプラットフォームに障害が起きている
クエリのパフォーマンスはAWS障害（S3の遅延など）に大きく影響されます。もちろん，トレジャーデータのプラットフォーム（TD API，PlazmaDB）の障害にも影響されます。障害が疑われる場合には https://status.treasuredata.com/ を参照してください。

### 単一ノード処理をする記述がある
1. DISTINCT
2. COUNT(DISTINCT x)
3. ORDER BY
4. UNION（UNION ALLではない）

これらの処理は，単一ノード上でしか実行できない（分散できない）ため，非常に時間がかかってしまう場合があります。また，「Exceeded max (local) memory xxGB error」というエラーで処理が中断してしまう可能性があります。可能な対処方法としては以下のようなものがあります。

1. DISTINCT → GROUP BYで置き換える
2. COUNT(DISTINCT x) → APPROX_DISTINCT(x) に書き換える
3. ORDER BY → どうしても使いたい場合にはHiveも併用する
4. UNION → UNION ALLで置き換える

### DISTINCTの書き換え
sales_slipにおけるカテゴリ，サブカテゴリ，グッズのユニークな組合せを列挙するクエリを考えます。DISTINCTの本来の目的は「重複を排除する」ことなので，以下のクエリがまず考えら
れます。
```sql
SELECT DISTINCT category, sub_category, goods_id
FROM sales_slip
```

一方，組合せの列挙はGROUP BYでも以下のように書けます。
```sql
SELECT category, sub_category, goods_id
FROM sales_slip
GROUP BY category, sub_category, goods_id
```
GROUP BYは，本来は以下を行うものです。
1. 指定されたカラムの値で行をグループ化
2. 各グループから1行取り出し
3. 取り出した行を集約してテーブルを再構成する

2.の過程は，ユニークな組合せを取り出すことを意味します。さらに3.の過程で，そのグループに対するCOUNTなどの集約情報を添えることができます。
上の例ではパフォーマンスの差異が見られませんが，集約などの処理も行えるGROUP BYでDISTINCTを代替しておくことをお勧めします。

### COUNT(DISTINCT x)の書き換え
カラムxのユニークな値を数えたい場合に用います。以下の例を見てみます。
```sql
SELECT COUNT(DISTINCT goods_id)
FROM sales_slip
```
|_col0     |
|----------|
|385768    |

このクエリは，GROUP BYを用いて以下のように書き換えることができます。
```sql
SELECT COUNT(goods_id)
FROM
(
  SELECT goods_id
  FROM sales_slip
  GROUP BY goods_id
)
```
|_col0     |
|----------|
|385768    |

DISTINCT自身はNULLもユニークな値として列挙してくれますが，COUNTされる際に無視されてしまいます。よってCOUNT(1)での等価な以下のクエリではNULLを除外してCOUNT(1)する必要があります。
```sql
SELECT COUNT(1)
FROM
(
  SELECT goods_id
  FROM sales_slip
  WHERE goods_id IS NOT NULL
  GROUP BY goods_id
)
```
|_col0     |
|----------|
|385768    |

GROUP BYでもパフォーマンスの改善にはなりません。そこで，厳密な数値を得る必要がなければ，APPROX_DISTINCT(x)関数を使用します。
```sql
SELECT APPROX_DISTINCT(goods_id)
FROM sales_slip
```
|_col0     |
|----------|
|397118    |


### ORDER BYの対処

ORDER BYは単一ノード処理となるので重い処理です。通常，単一ノードのメモリ上限は5GB程度なので，GB級の結果をORDER BYするのはかなり大変です。結果をテーブルに書き出したり外部出力したりする場合，レコードの並びはそれほど重要でない場合がほとんどでしょう。それでもORDER BYが必要であれば，以下のようにするのが賢明です。
1. 並び替えしない結果を「CREATE TABLE AS」または「INSERT INTO」でいったん書き込み
2. リソース競合しないHiveでORDER BYする

### UNIONをUNION ALLへ置き換え
UNIONとUNION ALLの違いを知らずに何気なく前者を使っていたら，考え方を改めてください。UNIONは重複レコードの除去を単一ノード処理で行うので思い処理となります。一方，UNION ALLは単純に重複などを見ずに結合するので圧倒的にパフォーマンスが優れています。

## パフォーマンスの悪いUDFを使っている
いくつかのUDFは，ローカルまたはリモート上のデータベースを参照しにいきます。例えば，IPアドレスを分析するトレジャーデータUDFや，通貨を変換するTD_CURRENCY_CONV関数がそれに該当します。これらの関数にたくさんのレコードを食わせるようにクエリが書かれていると，パフォーマンスは下がります。レコードをいくつか集約した後に初めてその関数が使われるように書けないか検討してみてください。

## GROUP BYに問題がある

### GROUP BYのキーが多数ある
GROUP BYのキーがたくさんあるようなクエリはメモリーの消費が激しく，パフォーマンスが確実に悪くなります。この事象に遭遇するのは，生データの重複を除外したいような場合です。この場合には存在するすべてのカラムがGROUP BYに並びますから，カラムが多いデータの重複処理には大変な時間がかかります。

### Cardinalityの高いカラムからGROUP BYのキーが並んでいない
ここで言う「Cardinalityが高い」とは，カラムの値のバリエーションの数が相対的に多いことを指します。例えば，sales_slipにおけるmember_idはgenderに比べて「Cardinalityが高い」といえます。GROUP BYのキーの並び順は，この「Cardinalityの高い」順に並べます。つまり，下記の2つでは前者のほうが適切です。
```sql
SELECT GROUP BY member_id, gender --good

SELECT GROUP BY gender, member_id --bad
```

## JOINに問題がある
### 「Simple Equi-Joins」を心がけていない
Equi-Joinsとは，ON節における左右テーブルの結合条件が「=」だけのJOINを指します。さらにSimpleとは，計算式を伴わない極々シンプルな記述を指します。以下で例を見てみましょう。
```sql
SELECT a.date, b.name FROM
left_table a
JOIN right_table b
ON a.date = CAST((b.year * 10000 + b.month * 100 + b.day) as VARCHAR)
```
上のクエリでは右側に計算式が入っており，VARCAHRと条件式のEqui-Joinとなるので，シンプルではありません。計算式では日付「2015-11-14」を「20151114」という文字列に変換しています。左側のテーブルがそのような形で格納されているからです。

上記のクエリは，VARCHARとVARCHARの「Simple Equi-Joins」として次のように書き換えることでパフォーマンスが改善されます。
```sql
SELECT a.date, b.name 
FROM left_table a
JOIN (
  SELECT
    CAST((b.year * 10000 + b.month * 100 + b.day) as VARCHAR) AS date, name
  FROM right_table
) b
ON a.date = b.date  --Simple Equi-Join
```

### JOINの数が多い
JOIN自体が重い処理なので，たくさんのJOINが行われるようなクエリは遅くなりがちです。

### JOINの右側のテーブルサイズが大きい（Broadcast Joinを理解していない）
JOINでは，基本的に右側にくるテーブルがマスタデータのようなスマートなテーブルであると想定されています。そして，右側のテーブルがすべてのワーカーに配信され，そのメモリ上に載ることになります（これはBroadcast Joinと呼ばれます）。よって，右側のテーブルがメモリに載らないほど巨大な場合には処理が遅くなります。これはLEFT OUTER JOINやRIGHT OUTER JOINのような向きのあるJOINだけでなく，INNER JOINやCROSS JOINについても当てはまります。

例としてCROSS JOINを挙げます。
```sql
SELECT * FROM small_table, large_table
WHERE small_table.id = large_table.id
```
上記のクエリより下記のクエリのほうが効率が良くなります。
```sql
SELECT * FROM large_table, small_table
WHERE large_table.id = small_table.id
```

### CROSS JOINを使っている
CROSS JOINは，その性質上（最大でレコード数nの2乗オーダーのスキャンが発生する），CPU消費が激しい処理です。CPU timeの情報を参考にしてください。

### JOINキーのサイズが大きい
値のサイズが大きいURLや長い文字列などをJOINのキーにする場合は，負荷を軽減するために，SMART_DIGEST関数でハッシュ値（6〜10の文字数かつユニーク性を保っている）に変換したものをJOINのキーとします。

以下の例では，member_id（サイズが大きい値ではありませんが）を参考として変換しています。
```sql
SELECT goods_id, slip.member_id, age
FROM sales_slip slip
JOIN
( 
  SELECT member_id, gender, age FROM master_members
) members
ON SMART_DIGEST(slip.member_id) = SMART_DIGEST(members.member_id)
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
```
同様にしてGROUP BYのキーのサイズの節約もできます。ただし，SMART_DIGEST関数の結果は逆変換できないので，変換前の元のカラム値がわからなくなってしまう点に注意が必要です。

### Distributed Hash Joinを試していない
「Distributed Hash Join」とは，双方のテーブルのハッシュ値をJOINキーに使うことです。右側のテーブルが大きくてもメモリの消費量を抑えてくれる代わりに，ネットワーク消費量が犠牲になります。これを使用するにはマジックコメントとして以下のように書きます。
```sql
-- set session join_distribution_type = 'PARTITIONED'
SELECT ... FROM large_table l, small_table s WHERE l.id = s.id
```

## TD_TIME_RANGEでTime Indexの効用が効いていない
TD_TIME_RANGEは，テーブルパーティションの参照範囲を最小限にとどめてテーブル読み込み速度を劇的に改善してくれます。しかし，TD_TIME_RANGEによるTime Indexの効率化が効かない記述になっていることがしばしばあります。ここではそのようなケースを丁寧に見ていきます。

### 終点（end_time）が省略されている
start_time（始点）とend_time（終点）にNULLを置くと，それぞれ「終点より前」，「始点以降」という意味合いで使用できます。後者のend_timeは省略できますが，その場合は効率化が効かないので注意が必要です。
```sql
SELECT ... WHERE TD_TIME_RANGE(time, '2013-01-01','JST') --NG
SELECT ... WHERE TD_TIME_RANGE(time, '2013-01-01', NULL, 'JST') --OK
```

### 引数内での計算で割り算をしている，結果がFLOAT型となる
始点と終点に計算式を入れたいケースがしばしばあります。しかし，そこで割り算や整数で返ってこないUDFを使ってしまうと効率化が効きません。TD_TIME系およびTD_DATE系の関数を使った場合には効率化が有効になります。
```sql
SELECT ... WHERE TD_TIME_RANGE(time, '2013-01-01',
                               TD_TIME_ADD('2013-01-01', '1', 'JST')) --OK
SELECT ... WHERE TD_TIME_RANGE(time, TD_SCHEDULED_TIME() / 86400 * 86400)) --NG
SELECT ... WHERE TD_TIME_RANGE(time, 1356998401 / 86400 * 86400)) --NG
```

### （知らず知らずのうちに）OR節の中で使われている
よく考えれば当然ですが，OR節の一方だけにTD_TIME_RANGEがあっても，他方の条件が考慮されて全件スキャンとなっていることがあります。AND節だけのクエリであれば，どんどん条件で絞り込まれていくので効率化が有効ですが，問題はORとANDが混在する場合です。

まず，「NOT，AND，OR」が「( )」なしで混在している場合，適用順序は「NOT > AND > OR」になります。つまり，先にANDがどんどん実行されて最後にORが適用されるので，「( )」なしのクエリにTD_TIME_RANGEがあってもだいたいは効率化が効いていません。意図的にORを「( )」で括って先に適用させることで効率化の希望が見えてきます。
```sql
SELECT * FROM table1
WHERE
  col1 < 100
  OR col2 is TRUE
  AND TD_TIME_RANGE(time, '2015-11-01')
```
上記のクエリは，下記のクエリと同等であり，ORの片方で使われるTD_TIME_RANGEによる効率化は効きません。
```sql
( col1 < 100 ) OR ( col2 is TRUE AND TD_TIME_RANGE(time, '2015-11-01') ) --NG
```
クエリの意図が先にORを適用させる場合であれば，以下のように書くことで効率化をはかれます。
```sql
( col1 < 100 OR col2 is TRUE ) AND TD_TIME_RANGE(time, '2015-11-01') --OK
```

## SELECT * を使っている
「SELECT *」は誰もが一番はじめに習う記述ですが，すべてのカラムを読み込むこの記述は実際には推奨されません。すべてのカラムを読み込む必要があるのは稀ですから，常に必要なカラムのみを選択するようにしてください。
また，一見するとSELECT *を使っていない下記のようなJOINにおいても，bのすべてのカラムが読み込まれます。
```sql
FROM a JOIN b
```
つまり，上記は以下と同等です。
```sql
FROM a JOIN ( SELECT * FROM b )
```
右側のテーブルが大きい場合には，クエリとしては長くなりますが，以下のように書きましょう。
```sql
FROM a JOIN ( SELECT col1, col2,... FROM b )
```

## 複数のLIKE表現が並んでいる
LIKEによる文字列抽出は，正規表現よりも表現能力が乏しいため，ORでたくさんのLIKEを連結することになって重い処理になりがちです。正規表現を工夫（正規表現の中でORを代替する）して少数のREGEXP_LIKEに置き換えることをお勧めします。
```sql
SELECT ...
FROM access_log
WHERE
  method LIKE '%GET%' OR
  method LIKE '%POST%' OR
  method LIKE '%PUT%' OR
  method LIKE '%DELETE%'
 ```
上記のクエリよりも，同結果が得られる下記のほうがパフォーマンスに優れています。
```sql
SELECT ...
FROM access_log
WHERE regexp_like(method, 'GET|POST|PUT|DELETE')
```

## Result Outputを使わず，クエリ内でCREATEやINSETを使う
Result Outputは，異なるアカウント間での結果の書き出しには必要ですが，同じアカウント内のテーブルへ書き出す場合には，クエリ内で結果を書き出すための記述をして実行するほうが並列処理ができて圧倒的に早くなります。なぜなら，Result OutputではS3へ結果のレコードを書き出す（アップロードする）必要があるからです。

### 既存のテーブルを上書き（Overwrite）する場合
既存のテーブルを上書きする場合には，必ずはじめにそれをDROPしておきます。テンプレートは以下のようになります。CREATE TABLE table AS以降は，全体を「( )」で囲まず，いつものようにSELECT文を書いてください。DROP TABLE節の最後にセミコロンを忘れないでください。
```sql
DROP TABLE IF EXISTS my_result;
CREATE TABLE my_result AS
SELECT * FROM my_table
```

### 既存のテーブルに追記（Append）する場合
この場合は，もしテーブルが存在しなかった場合にエラーとならないように，timeカラムだけを持ったテーブルを予め作成しておきます。なお，INSERTするSELECT節の中にtimeカラムがなければ，実行時の時刻がtimeカラム値として挿入されることになります。多くの場合はレコード作成時の時間を使うはずなので，timeカラムを書き忘れないようにしてください。また，CREATE節の最後にセミコロンを忘れないでください。
```sql
CREATE TABLE IF NOT EXISTS my_result(time bigint);
INSERT INTO my_result 
SELECT * FROM my_table
```
