# Lesson 04. 集約関数

## 集約関数一覧
|FIELD1    |FIELD2       |FIELD3     |
|----------|-------------|-----------|
|関数名       |概要           |Return Type|
|COUNT(*), COUNT(col), COUNT(expr),  COUNT(DISTINCT col)|(*)：NULLを含む行の合計数を返します  (1)：NULLを含む行の合計数を返します  (col)：colの値がNULLでない行の合計数を返します  (expr)：指定されている式の結果がNULLではない行の数を返します  (DISTINCT col)：NULLでないユニークな行の合計数を数えます|BIGINT     |
|SUM(col), SUM(expr), SUM(DISTINCT col)|(col)：グループ内の列の値の合計を返します  (expr)：指定されている式の結果がNULLか0ならば0が加算され，それ以外の数値ならばその値が加算されます  (DISTINCT col)：グループ内の列のユニークな値の合計を返します|DOUBLE     |
|AVG(col),  AVG(expr), AVG(DISTINCT col)|(col)：グループ内の列の値の平均を返します。値がNULLの場合はスキップされ，行数（分母）にも加えられません。  (expr)：指定されている式の結果がNULLの場合はスキップされ，行数（分母）にも加えられません。それ以外の数値ならその値が加算され，行数（分母）も1でカウントされます  (DISTINCT col)：グループ内の列のユニークな値の平均を返します|DOUBLE     |
|MIN(col)  |グループ内の列の最小値を返します|DOUBLE     |
|MAX(col)  |グループ内の列の最大値を返します|DOUBLE     |
|VARIANCE(col), VAR_POP(col)|グループ内の数値列の分散を返します|DOUBLE     |
|VAR_SAMP(col)|グループ内の数値列のサンプル分散（不偏分散）を返します|DOUBLE     |
|STDDEV_POP(col)|グループ内の数値列の標準偏差を返します|DOUBLE     |
|STDDEV_SAMP(col)|グループ内の数値列のサンプル標準偏差を返します|DOUBLE     |
|COVAR_POP(col1, col2)|グループ内の数値列のペアの母集団の共分散を返します|DOUBLE     |
|COVAR_SAMP(col1, col2)|グループ内の数値列のペアのサンプル共分散を返します|DOUBLE     |
|CORR(col1, col2|グループ内の数値列のペアの相関係数（Pearson係数）を返します|DOUBLE     |

## COUNTの挙動を理解する

（様々な条件で）行数を数えるCOUNT関数は，最もよく使われる集約関数の1つです。この最も基礎的な関数でさえ，NULLの扱い（有効として数えるか否か）が明確でない場合に適当な引数が使われて間違った結果をもたらしてしまう事例が絶えません。ここではCOUNTの正しい挙動を理解することで集約関数について理解を深めることにします。例として，次のクエリで生成したNULLを含む6件の数値からなるテーブルを使います。

```sql
SELECT a
FROM ( VALUES 0, 0, 1, 2, NULL, NULL ) AS t(a)
```

上記のテーブルに対し、次のように様々な引数を持ったCOUNTを実行します。

```sql
SELECT 
  COUNT(*)     AS cnt_aster, 
  COUNT(0)     AS cnt_0, 
  COUNT(1)     AS cnt_1, 
  COUNT(FALSE) AS cnt_false,
  COUNT(NULL)  AS cnt_null, 
  COUNT(a)     AS cnt_col,
  COUNT(IF(a=0,1,0))     AS cnt_if_0,
  COUNT(IF(a=0,1,NULL))  AS cnt_if_null,
  COUNT(IF(NULL,1,0))    AS cnt_if_null_0,
  COUNT(IF(NULL,1,NULL)) AS cnt_if_null_null
FROM ( VALUES 0, 0, 1, 2, NULL, NULL ) AS t(a)
```
|cnt_aster |cnt_0        |cnt_1      |cnt_false|cnt_null|cnt_col|cnt_if_0|cnt_if_null|cnt_if_null_0|cnt_if_null_null|
|----------|-------------|-----------|---------|--------|-------|--------|-----------|-------------|----------------|
|6         |6            |6          |6        |0       |4      |6       |2          |6            |0               |


上記の結果をまとめてみましょう。

- 引数がNULL以外の定数値（0，1，FALSE）および「*」の場合は，NULLを含む全レコード件数6を返す
- 引数が常にNULLの場合は，すべてのレコードがスキップされ，レコード件数は0となる
- 引数が特定のカラムを指定する場合には，そのカラム値がNULLでないレコード件数4を返す
- 引数がIF式の場合，
  - IF(a=0, 1, 0)は，aがNULLの場合は無条件でNULLを返し，この場合の式の返り値は偽値の0を返す
  - aが0の場合の式の返り値は真値の1を返す
  - aが0でなくNULLでない場合の返り値は偽値の0を返す
  - IF(NULL, 1, 0)の条件式は無条件でNULLを返し，この場合の式の返り値は偽値の0を返す
  - IF(NULL, 1, NULL)の条件式は無条件でNULLを返し，この場合の式の返り値は偽値のNULLを返す
- このIFの返り値を受けて，
  - COUNT(NULL)の行は参照されない（スキップ）
  - COUNT(1)およびCOUNT(0)の行は数えられる
- COUNT(IF(a=0, 1, 0))は，a=0の条件結果にかかわらずNULLを含む全件数6を返す
- COUNT(IF(a=0, 1, NULL))は，aが0である場合のみCOUNT(1)で，他はCOUNT(NULL)でスキップされるため，件数2を返す
- COUNT(IF(NULL, 1, 0))は，すべてのレコードでCOUNT(0)となるため，全件数6を返す
- COUNT(IF(NULL, 1, NULL))は，すべてのレコードでCOUNT(NULL)となるため，レコード件数は0となる

本テキスト全体では，次の3種のCOUNT引数を用途別に採用するものとします。

- NULLを含むレコードの件数：COUNT(1)
- 特定のカラムでNULLを含まないレコードの件数：COUNT(a)（※カラムaがNULLを含む場合）
- 条件を満たす場合はカウント，満たさない場合はカウントしない：COUNT( IF(expr, 1, NULL) ) （※exprは条件式）

## 単純なユーザー数のカウント [ COUNT * ]
以降ではサンプルデータをもとにしてCOUNTの例を見ていきます。
SELECT COUNT(1) AS cnt
FROM sales_slip

上記のクエリは，sales_slipテーブルに存在する全レコードをカウントしていることになります。
「抽出」から「集計」へ
上記クエリの実行結果について注目してほしいのは以下の点です。
sales_slipでは，5,892,348件のレコードに対してクエリの結果は1件
sales_slipには含まれない結果テーブルが返ってくる
これは，今回のクエリタイプが「抽出」ではなく，「集計（集約）」であることによります。「集計」とは，あるテーブルに対して以下の結果を求め，そのテーブルの持つ情報の「サマリ」を得ることです。
テーブル内のあるカラムに収められた数値に対して合計や平均を求める
ある条件を満たすレコードの件数を数える
あるカラムの最大／最小値を求める
サマリには情報が集約されるものなので，結果のテーブルは元のテーブルとは異なる圧縮された情報（=少ないレコード件数）となります。
集約関数と算術関数（演算子）の大きな違いは，算術関数や演算子は「横の計算」の役割を担い，集約関数は「縦の計算」の役割を担うことです。
算術演算子は，同レコード内のカラムaの数値とカラムbの数値に対して四則演算を行うものであり，いわば横方向の演算です。
数学関数は，あるカラムの数値に対してSINやLOGなどの関数を適用するものです。また，小数点以下の値を四捨五入するROUNDや，ランダム値を出すRANDなどの関数があります。これらもまたレコード内のある数値カラムに適用されるものです。
集約関数は，上記の2点とは決定的に異なります。それは，集約関数はある数値カラムに対してレコードを貫通して縦方向に演算を行うことです。

次に，レコードの中で「member_idがNULLでない」レコード件数を取得してみましょう。NULLでないものを除外することを考えるときは，「どのカラム」について（つまり主語が必要）NULLを除外するのかを意識してください。例えば，今回のデータではmember_id内にはNULLが含まれますが，それ以外のカラムにはNULLは含まれません。
SELECT COUNT(member_id) AS cnt
FROM sales_slip

先程より少ない件数が結果として得られました。これは，member_idがNULLであるレコードがカウントされていないからです。
「member_idがNULLのものを除外してカウントしたい」という意図を明確に伝えるうえでは，同じ結果でも以下のクエリのほうが優しいといえます。
SELECT COUNT(1) AS cnt
FROM sales_slip
WHERE member_id IS NOT NULL
カラムの値によるグループごとのCOUNT [ GROUP BY ]
sales_slipには，同じユーザーでも購入する時間が異なれば別レコードとして記録されます。そのため，ユーザーが重複して購入回数の数だけレコードが存在します。そうすると，ユーザーごとの購入回数というものを調べたくなってきます。そこで登場するのがGROUP BY節です。
まず，SELECT節とGROUP BY節にmember_idだけを指定した以下のクエリを考えます。
SELECT member_id
FROM sales_slip
GROUP BY member_id
ORDER BY member_id

この結果には，以下のクエリと同じく，ユニークなmember_idがリストアップされています。
SELECT DISTINCT member_id
FROM sales_slip
ORDER BY member_id
GROUP BYの本領は，member_idごとに「集計」をさせられることです。
SELECT member_id, COUNT(1) AS cnt
FROM sales_slip
GROUP BY member_id
ORDER BY cnt DESC

上記のクエリでは，結果を購入回数の多い順に並び替えています。GROUP BYにおいてはNULLを1つの値として扱っていることに注意してください。言い換えると，NULLの件数1,520,178件を他のmember_idと一緒に数えています。NULLを除外したい場合には「WHERE member_id IS NOT NULL」を加えます。WHERE節におけるIS NULLの追加の仕方については後述します。
このように，GROUP BYにより指定したカラムの値について集計を行うことが可能となります。しかし，GROUP BYで集計を行う際には，エラーを起こさないためにいくつか注意が必要です。
SELECT節でGROUP BY節にないカラムを登場させるためには，集約関数を与えてやらないといけない
GROUP BY節には，member_idやcategory，sub_category，goods_idのような「数」でないカラムが入る。こうした集計の「内訳項目」となるようなカラムは「ディメンジョン」と呼ばれることがある。GROUP BY節に指定した複数のディメンジョンは，必ず（裸のままで）SELECT節に登場しないといけない
集約の対象となる（集約関数の引数となる）カラムは，amountやprice，またはその積（amount × price）といった数値計算可能なカラムであり，こうしたカラムは「メジャー」と呼ばれることがある
ディメンジョンもメジャーも1つに限定されない


これらのルールとともに，このクエリが実行される順序が理解できると，以下のクエリはエラーになることがわかるようになります。
SELECT member_id, category, COUNT(1) AS cnt
FROM sales_slip
GROUP BY member_id
ORDER BY cnt DESC
LIMIT 10
上記クエリを実行すると，以下のように，categoryカラムがGROUP BY節にないことが指摘されます。
'category' must be an aggregate expression or appear in GROUP BY clause
このエラーの原因をもう少し説明しましょう。はじめに，「いくつのディメンジョンで集計するか」がGROUP BYの記述によって決まります。実行されたクエリはmember_idごとに集計されるものと認識され，同じ値のmember_idのレコードが同じところに集められてきます。
一方，SELECT節では，そのmember_idに加えてcategoryが指定されています。このクエリはmember_idごとにのみ集めてくるものであり，categoryなど他のディメンジョンのことはまったく考慮されていないので，ここでストップしてしまいます。
次に正しいクエリを見てみましょう。
SELECT member_id, category, COUNT(1) AS cnt
FROM sales_slip
GROUP BY member_id, category
ORDER BY cnt DESC
LIMIT 10
こちらのクエリでは，GROUP BYでmember_idごと，さらにcategoryごとに集計が行われると認識され，この2つの組合せにより同じ値のレコードが同じところに集められてきます。SELECT節では，きちんとその2つのディメンジョンが選択されているので，次の作業のCOUNTという集計命令が問題なく実行されて結果が出力されます。

WHERE節で集計対象を予め絞り込む [ WHERE GROUP BY ]
WHERE節によって予め集計の対象レコードを絞り込むことができます。その際には以下の点に注意してください。
WHERE節はGROUP BY節の前に書くこと
GROUP BYよりも先にWHEREが実行されること
SELECT member_id, category, COUNT(1) AS cnt
FROM sales_slip
WHERE member_id = '10000'
GROUP BY member_id, category
ORDER BY cnt DESC

sales_slipのようなレコード数が多いテーブルでは，時間範囲指定のWHERE節（詳細は後述の章で説明）なしに全レコードを対象とする集計をかけると，とても時間がかかります。また，多くの場合，集計では1日や1ヶ月といった限定的な範囲のレコードしか用いません。クエリの実行時間が長くなる場合には，以下のように時間範囲指定のWHERE節を入れることがあります。
SELECT member_id, category, COUNT(1) AS cnt
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id, category
ORDER BY cnt DESC

冒頭で示した2つの注意点のうち，「GROUP BYよりも先にWHEREが実行されること」は特に重要です。集計値に対して条件による絞り込みを行いたい場合がよくあるでしょう。例えば，「categoryでの購入回数が150を超えるmember_idとそのcategoryを取ってくる」という例を考えてみてください。以下のようなクエリが考えられますが，これではエラーになります。
SELECT member_id, category, COUNT(1) AS cnt
FROM sales_slip
WHERE 150 <= COUNT(1)  --エラー箇所！
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id, category
ORDER BY cnt DESC
LIMIT 10
上記クエリがエラーになるのは，条件を指定しようとしたCOUNT(1)という集計が，WHEREが実行された時点ではまだ行われていないことに起因します。WHEREを使って目的を果たすためには，時間範囲指定のWHEREを早いうちに書くことが重要なので，以下のように内側のSELECT文に記述する必要があります。
SELECT member_id, category, cnt
FROM
(
  SELECT member_id, category, COUNT(1) AS cnt
  FROM sales_slip
  WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, category
)
WHERE 150 <= cnt
ORDER BY cnt DESC
LIMIT 10

結果は出力されましたが，NULLが上位になっているのが気になります。特定できるmember_idで抽出し直しましょう。NULLを除外するためのWHEREの位置に注目してください。
SELECT member_id, category, cnt
FROM
(
  SELECT member_id, category, COUNT(1) AS cnt
  FROM sales_slip
  WHERE member_id IS NOT NULL
  AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, category
)
WHERE 150 <= cnt
ORDER BY cnt DESC
LIMIT 10

このクエリはだいぶ長くなってしまいました。実は，集計においてはHAVINGという絞り込みが可能で，こちらを使うとよりシンプルな記述で目的を果たせます。
HAVING節による集計値の条件でさらに絞り込む [ GROUP BY HAVING ]
HAVING節は，WHEREと同じ条件絞り込みでも，GROUP BYの後に実行される挙動を示します。HAVING節の注意点は以下の2つです。
HAVING節はGROUP BY節の後に書くこと
GROUP BYよりも後にHAVINGが実行されること
先程と同じ結果をHAVING節でシンプルに記述すると以下のようになります。
SELECT member_id, category, COUNT(1) AS cnt
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id, category
HAVING 150 <= COUNT(1)
ORDER BY cnt DESC
LIMIT 10

ここで少し頭を整理しましょう。今のクエリの実行順は次のように表せます。

まず，WHERE節の中でもTD_TIME_RANGE関数を用いた絞り込みが一番先に行われます。
次に，絞り込まれた時間範囲のテーブルパーティションのみがFROM節でスキャンしされます。
続いて，一般のWHERE節による絞り込みが行われます。まだGROUP BYは実行されていないので，集約関数による絞り込みはできません。
GROUP BYによって，節に指定されたディメンジョンごとにデータが集約されます。
SELECT節によって，集約されたデータに何らかの集計演算が実行されます。
集約された後にHAVINGが実行されるので，集約関数を条件に使えます。
ORDER BYで並び替えられます。
最後に，LIMITによる絞り込みが行われます。ORDER BYが前置されている場合，全レコードの読み取りおよびSORTアルゴリズムの実行後に絞り込まれることになるので，パフォーマンスの改善は微々たるものです。
GROUP BYの挙動とルール，そしてWHEREとHAVINGの実行順序が理解できれば，集計については1つの山を越えたことになります。
集計クエリの実行順序まとめ
（左）記述順序（右）実行順序
SUMによる値の合計
数値の入った，あるカラムの値について合計を取りたいときには，SUM関数を使います。
SELECT 
  COUNT(1) AS cnt, 
  COUNT(member_id) AS cnt_omit_null, 
  SUM(amount) AS total_amount
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

上記のクエリでは，SUM(amount)で全レコードの購入点数合計を求めていますが，member_idがNULLのレコードの値も加算されています。member_idが特定されているレコードに限って合計を求めたい場合には，下記のようにしてNULLを除外します。
SELECT 
  COUNT(1) AS cnt,
  COUNT(member_id) AS cnt_omit_null, 
  SUM(amount) AS total_amount
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

各レコードについて（購入点数）×（価格）=（売上総額）となるので，NULLを除く全レコードの総売上額を求めるには以下のように書きます。SUM(amount * price)では，各レコードごとのamount * price値が足し合わされていきます。
SELECT COUNT(1) AS cnt, SUM(amount * price) AS total_sales
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

次に，member_idごとの購入回数（cnt），合計購入点数（total_amount），売上総額（total_sales）を一度に見てみましょう。売上の多い上位10件を取得します。ここではNULLが集計されている様子をあえて見てみます。
SELECT member_id, COUNT(1) AS cnt, 
  SUM(amount) AS total_amount,
  SUM(amount * price) AS total_sales
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id
ORDER BY total_sales DESC
LIMIT 10

ここで，売上総額を購入回数で割れば「1回の購入当たりの平均金額」，合計購入点数を購入回数で割れば「1点当たりの平均金額」を求めることができます。また，今度は NULLを除外します。さらに，SUM(amount)の値が0のmember_idがあれば0による除算が起こるので，そのmember_idに遭遇した時点でこのクエリはエラーを返し，結果を返してくれません。そこで，HAVING節でそのようなmember_idを除外しておきます。COUNT(1)については0を返すことがないので0除算を考慮する必要はありません。
SELECT member_id, 
  COUNT(1)                          AS cnt, 
  SUM(amount)                       AS total_amount,
  SUM(amount * price)               AS total_sales,
  SUM(amount * price) / COUNT(1)    AS avg_sales_per_cnt,
  SUM(amount * price) / SUM(amount) AS avg_sales_per_amount
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id
HAVING 0 < SUM(amount)
ORDER BY total_sales DESC
LIMIT 10

上記クエリでは結果を，total_salesで並び替えていますが，avg_sales_per_cntやavg_sales_per_amountで並び替えると，また違った世界が見えそうです。ここで上記クエリの次の部分に注目してください。
  SUM(amount * price) / COUNT(1)    AS avg_sales_per_cnt,
  SUM(amount * price) / SUM(amount) AS avg_sales_per_amount
これを以下のようにショートカットで記述できないでしょうか？
  total_sales / cnt          AS avg_sales_per_cnt,
  total_sales / total_amount AS avg_sales_per_amount
実は，同じSELECT節で新しく別名が割り振られたカラムの名前は，その外側のSELECT節で初めて利用できるようになります。そのため，上記のように書くと，そのような名前のカラムがないとエラーが出ます。同じ結果を返すためには次のようなクエリを書きます。
SELECT member_id, cnt, total_amount, total_sales, 
  total_sales / cnt          AS avg_sales_per_cnt,
  total_sales / total_amount AS avg_sales_per_amount
FROM
(
  SELECT member_id,
    COUNT(1)            AS cnt, 
    SUM(amount)         AS total_amount,
    SUM(amount * price) AS total_sales
  FROM sales_slip
  WHERE member_id IS NOT NULL
  AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id
  HAVING 0 < SUM(amount)
)
ORDER BY total_sales DESC
LIMIT 10
AVGによる値の平均は注意深く
平均を求めるAVGはSUM関数と同じくらい有名ですが，以下の場合には注意が必要です。
AVG集計対象の値にNULLが含まれるとき
その他のディメンジョンでNULLが含まれるとき
前者については簡単な例で挙動を確認できます。まずはNULLがない場合を見てみます。
SELECT AVG(a) AS avg1
FROM ( VALUES 1, 1, 1, 1, 0, 0 ) AS t(a)

結果は，4個の1と2個の0を足した4をレコード数の6で割った4/6の値です。ここで0の代わりにNULLが含まれるとどうなるでしょうか？
SELECT AVG(a) AS avg2
FROM ( VALUES 1, 1, 1, 1, NULL, NULL ) AS t(a)

NULLの値はAVGの集計対象とならず，4個の1を足してレコード数の4で割った結果になりました。
場合によってはNULLを0と同一視して扱いたいことがあります。SUM関数では気にしなくてよかったのですが，AVG関数ではNULLを気にする必要があります。同一視する場合には，予めNULLを0とみなす記述が必要です。
そこで登場するのがCOALESCE関数です。これは1番目の引数で指定したカラム内のすべてのNULLを2番目の引数の値で置き換えるとても便利な関数です。
SELECT AVG( COALESCE(a, 0) ) AS avg3
FROM ( VALUES 1, 1, 1, 1, NULL, NULL ) AS t(a)

AVGに関してもう1つ注意が必要なのは，「その他のディメンジョンでNULLが含まれる」ときです。sales_slipの例では，ディメンジョンmemebr_idにNULLが含まれており，集計対象のamountやpriceにはNULLが含まれていない場合がそれに該当します。ここでは，SUM関数の説明の後半で登場したavg_sales_per_cnt（1回の購入当たりの平均金額）をAVGにより簡潔に記述することでAVG関数の仕組みを理解しましょう。（ちなみに，1点当たりの平均金額を計算するavg_sales_per_amountのほうはAVGでは記述できません。）
SELECT
  COUNT(1)                             AS cnt, 
  1.0* SUM(amount) / COUNT(1)          AS avg_amount1,
  AVG(amount)                          AS avg_amount2,
  1.0 * SUM(amount * price) / COUNT(1) AS avg_sales_per_cnt1,
  AVG(amount * price)                  AS avg_sales_per_cnt2
FROM sales_slip
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

上記の結果から類推できますように，AVG関数はその引数に指定されたカラムの値について，その合計をレコード件数で割った「1件当たり」の平均を求めます。AVG関数で暗黙にCOUNT関数も使われるということは，全体の平均を取るとき，あるカラム（今の例ではmember_id）に含まれるNULLを含むか否かで結果が変わることに注意してください。（NULLを含めると結果が間違っているという意味ではありません。NULLの扱い方の方針によってどちらが正しいかが決まります。）
以下のクエリはmember_idのNULLを除外した集計結果になります。
SELECT
  COUNT(1)            AS cnt, 
  AVG(amount)         AS avg_amount3,
  AVG(amount * price) AS avg_sales_per_cnt3
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

member_idがGROUP BYでディメンジョンに指定される場合，NULLはそれを1つの値として集約するので，集計値に影響が出ることはありません。NULLが必要ない場合には常に除外するという習慣をつけましょう。
SELECT member_id, 
  COUNT(1)                             AS cnt, 
  1.0 * SUM(amount) / COUNT(1)         AS avg_amount1,
  AVG(amount)                          AS avg_amount2,
  1.0 * SUM(amount * price) / COUNT(1) AS avg_sales_per_cnt1,
  AVG(amount * price)                  AS avg_sales_per_cnt2
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id
LIMIT 10

時間をGROUP BYで扱う
集計においては，1日や月間といった特定の時間範囲を指定するのが一般的です。時間範囲のWHERE節による絞り込みについては既に紹介しました。
一方，日付ごと（デイリー，日次）や月ごと（マンスリー，月次）に集計をかけることもよく行います。この場合には，GROUP BY節に time値そのものでなく日付や月といった丸めた時間を置いて集計します。
日付ごとの売上を見る例で説明します。下記のクエリは，2011年における売上をデイリーで見るものです。
SELECT 
  TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS d,
  SUM(amount * price) AS total_sales
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
ORDER BY d DESC
LIMIT 10

下記は，カテゴリごとに月次での売上を見るクエリです。
SELECT 
  TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST') AS d,
  category,
  SUM(amount * price) AS total_sales
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-01', 'JST'), category
ORDER BY d DESC, category
LIMIT 10


MAX，MINを扱う集計
MAXおよびMIN関数は，集約関数の1つとして扱われますが，これらが返す値は「集計」された値ではなく，レコード内の実在する値の中で最大と最小に該当する値です。それらが「そのまま提示される」ことに注意が必要です。
MAXおよびMINに指定したカラムのレコード内にNULLがあると，そのレコードは無視されます。すべてのレコードがNULLの場合はNULLを返します。
MAX_BYおよびMIN_BYという，とても有用な関数もあります。MAX_BY(b, a)は，あるカラムaの値が最大となるレコードのカラムbの値を拾ってきます。MAX_BY(b, a, n)は，カラムaの値上位n件を（LISTとして）返します。
下記のクエリは，salse_slip中のpriceの最大および最小値と，最大および最小となるgoods_idが何かを特定する例です。
SELECT 
  MAX(price) AS max_price, 
  MIN(price) AS min_price, 
  MAX_BY(goods_id, price) AS goods_id_max_price, 
  MIN_BY(goods_id, price) AS goods_id_min_price
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

price = 37,9048のgoods_id = 514206が最も高く，price = 1のgoods_id = 508703が最も安いことがわかりました。念のため，goods_idとpriceの関係が正しいか確認しておきます。
SELECT DISTINCT category, sub_category, goods_id, price
FROM sales_slip
WHERE goods_id IN ('514206','508703')
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

次に，member_idごとに，「1回の買い物額」が最も高いものと安いものを見てみましょう。下記のクエリは買い物額が最も高いmember_idの上位10件をピックアップするものです。
SELECT member_id,
  MAX(price) AS max_price, 
  MIN(price) AS min_price, 
  MAX_BY(goods_id, price) AS goods_id_max_price,
  MIN_BY(goods_id, price) AS goods_id_min_price
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id
ORDER BY max_price DESC
LIMIT 10

下記のクエリは，member_idごとの2011年の購入履歴の最初と最新を，日付と値段を含めて調べる例です。
SELECT member_id,
  TD_TIME_FORMAT(MAX(time), 'yyyy-MM-dd', 'JST') AS last_access, 
  TD_TIME_FORMAT(MIN(time), 'yyyy-MM-dd', 'JST') AS first_access, 
  MAX_BY(goods_id, time) AS goods_id_last_access, 
  MIN_BY(goods_id, time) AS goods_id_first_access,
  MAX_BY(price, time) AS price_last_access, 
  MIN_BY(price, time) AS price_first_access
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY member_id
ORDER BY last_access DESC
LIMIT 10

最頻値
最頻値（mode）とは，最も頻繁に現れる値のことで，平均値や中央値と並ぶ重要な情報です。Prestoには最頻値を求める関数がありませんが，前述のMAXおよびMAX_BY関数を利用して求めることができます。
sales_slip全体（2011年）におけるpriceの最頻値を求めてみましょう。ここで注意が必要なのは，1つのレコードが必ずしも1点の購入とは限らず，amountに購入点数が入っていることです。
まず簡単のため，1つのレコードが1点の購入とみなして考えます。
SELECT MAX_BY(price, cnt) AS mode, MAX(cnt) AS mode_frequency
FROM
(
  SELECT price, COUNT(1) AS cnt
  FROM sales_slip
  WHERE member_id IS NOT NULL
  AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY price
)

この結果は，最も登場回数が多いのは price=2,839 のレコードで，48,232回登場することを示しています。
次に，amount（購入点数）を考慮したpriceの最頻値を算出します。
SELECT MAX_BY(price, amount) AS mode, MAX(amount) AS mode_frequency
FROM
(
  SELECT price, SUM(amount) AS amount
  FROM sales_slip
  WHERE member_id IS NOT NULL
  AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY price
)

この結果は，最も購入回数が多いのは price=2,839 のアイテムで，51,319点購入されていることを示しています。
さらに発展として，最頻値ではなく最頻「アイテム」を特定してみましょう。
SELECT MAX_BY(goods_id, amount) AS mode_goods_id, MAX(amount) AS mode_frequency
FROM
(
  SELECT goods_id, SUM(amount) AS amount
  FROM sales_slip
  WHERE member_id IS NOT NULL
  AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY goods_id
)

この結果は，最も購入回数が多いのは goods_id=48745 のアイテムで，7,059点購入されていることを示しています。
最後に，member_idごとの最頻アイテムを求め，頻度が高いmember_idの上位10件の結果を出してみます。
SELECT member_id, MAX_BY(goods_id, amount) AS mode_goods_id, MAX(amount) AS mode_frequency
FROM
(
  SELECT member_id, goods_id, SUM(amount) AS amount
  FROM sales_slip
  WHERE member_id IS NOT NULL
  AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
  GROUP BY member_id, goods_id
)
GROUP BY member_id
ORDER BY mode_frequency DESC
LIMIT 10

BOOL_AND，BOOL_MIN
sales_slipでは，購入後にキャンセルがあった場合，「is_canceled」カラムが1となっています。このカラムを使って，「一度でもキャンセルしたことのある人」と「キャンセルしかしていない人」を見つけてみましょう。前者は is_canceled=1 が1つでもある人，後者はis_canceledに1しかない人です。（sales_slip内に該当人数が少ないので，下記のクエリでは時間範囲を指定しておらず，実行に時間がかかります。）
SELECT member_id, is_canceled_any, is_canceled_every
FROM
(
  SELECT member_id, 
    BOOL_OR( IF(is_canceled=1,TRUE,FALSE)) AS is_canceled_any,
    BOOL_AND(IF(is_canceled=1,TRUE,FALSE)) AS is_canceled_every
  FROM sales_slip
  WHERE member_id IS NOT NULL
  GROUP BY member_id
  HAVING BOOL_OR(IF(is_canceled=1,TRUE,FALSE)) --=TRUE
)
ORDER BY is_canceled_every DESC, is_canceled_any

上位2名がキャンセルしかしていない人，残り1名がキャンセルをしたことがある人です。この3人のキャンセル率を見てみましょう。
SELECT member_id,
  1.0 * COUNT(IF(is_canceled=1,1,NULL)) / COUNT(1) AS cancel_ratio
FROM sales_slip
WHERE member_id IN ('2259091','949366','323685')
GROUP BY member_id
ORDER BY cancel_ratio DESC

キャンセルしかしていない人のキャンセル率は100%であることが見てとれます。
中央値，パーセンタイル（近似）
ここまで平均値と最頻値を求める関数を紹介しましたが，中央値を求める関数は登場していません。実は，集約関数の中には中央値を求める関数がなく，これを厳密に求めるためにはWindow関数を用いる必要があります。ただし，近似的に中央値およびパーセンタイル（分位点）を求めることはAPPROX_PERCENTILE関数により可能です。
SELECT
  MAX(price) AS max_price,
  approx_percentile(price, 1)    AS max_price2,
  approx_percentile(price, 0.75) AS per75_price,
  approx_percentile(price, 0.50) AS median,
  approx_percentile(price, 0.25) AS per25_price,
  approx_percentile(price, 0)    AS min_price2,
  MIN(price) AS min_price
FROM sales_slip
WHERE member_id IS NOT NULL
AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')

APPROX_PERCENTILE(price, x)のxに0から1の間の割合を記述することで，中央値（x=0.5）およびxパーセンタイル（分位数）を求めることが可能です。上記のクエリでは代表的な4分位数を求めています。これらの結果はあくまで近似であることに注意してください。
このLessonで注意したい点まとめ
COUNT(1)とCOUNT(col)はNULLの存在で結果が異なる場合があり，その理由を理解すること
集計中の値の条件指定はHAVING句で。WHEREとの違いと指定する位置に注意
AVGの結果もNULLの存在で意図せぬ結果を招く可能性がある
MAXとMINだけでなくMAX_BYとMIN_BYも有用なので，覚えておこう
TD_TIME_FORMTAなどの時間によるGROUP BYは定番なので，日次や月次の出し方を覚えておこう

