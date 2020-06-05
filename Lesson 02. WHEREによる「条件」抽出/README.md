# Lesson 02. WHEREによる「条件」抽出

## 比較演算子による条件抽出 [ WHERE ]

WHERE節の後に条件を記述することによって，その条件に合致するレコードだけを抽出します。次の例では特定のtd_client_idを指定して結果を絞り込んでいます。

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE td_client_id = 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
LIMIT 10
```
|td_client_id|time       |
|------------|-----------|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1464572103 |
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1464573326 |
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1464570933 |

WHEREの後に記述するものは「条件式」と呼ばれ，必ずTRUEまたはFALSEのどちらかを返す必要があります。条件式がNULLを返す場合にはそのレコードは参照されません（抽出されません）。

その他の基本的な比較演算子のクエリ例をいくつか挙げておきます。

### 特定のtd_client_idを含まない

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE td_client_id <> 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a' --含まない
LIMIT 10
```
|td_client_id|time       |
|------------|-----------|
|32258bfe-444d-44fa-ad90-e71eea183008|1467187504 |
|c48672da-1b44-46a0-ab9d-c24cdbe9f146|1467190495 |
|3ee1cc12-5cc9-4b93-f1cb-31a4f3872d4e|1467187876 |

### 特定のtimeの値以上

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE time >= 1463640700 --time の値が 1463640700 以上
LIMIT 10
```
|td_client_id|time       |
|------------|-----------|
|411b1c21-3e98-4a1a-a6cd-f41aaf69a5dc|1464239250 |
|2705c701-3b7c-4ef3-d88d-50529a3d1060|1466410889 |
|228e28e6-1753-4e69-b35f-5362fc725ec5|1464239515 |

### 特定のtimeより小さい

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE time < 1463640700 -- time の値が 1463640700 より小さい
LIMIT 10
```
|td_client_id|time       |
|------------|-----------|
|c6c0c9cd-fd18-4c21-c795-5bca1ab63066|1462089548 |
|bdd71c57-06cf-4572-f504-eceadbec4e91|1462089432 |
|bdd71c57-06cf-4572-f504-eceadbec4e91|1462089362 |

## 複数の条件指定 [ WHERE A AND/OR/NOT B ]

複数の条件を組合せて絞り込みを行いたい場合，ANDやORといった論理演算子を使います。

### 特定のtd_client_idを含み，かつ特定のtimeの値以上

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE td_client_id = 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
AND time >= 1463640700
ORDER BY time
LIMIT 10
```
|td_client_id|time       |
|------------|-----------|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640700 |
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640811 |
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640889 |

比較演算子を複数つなげるには，論理演算子を使います。基本的な論理演算子のクエリ例をいくつか挙げていきます。

### td_client_idの特定の3ユーザーのレコードのみ抽出

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE td_client_id = 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
OR td_client_id = '7f47d05f-bd12-4553-e69c-763064738631'
OR td_client_id = '780e865b-35f2-4e56-990a-5ed67cf733a3'
ORDER BY time
LIMIT 10
```
|td_client_id|time       |
|------------|-----------|
|780e865b-35f2-4e56-990a-5ed67cf733a3|1461348266 |
|780e865b-35f2-4e56-990a-5ed67cf733a3|1461348343 |
|780e865b-35f2-4e56-990a-5ed67cf733a3|1461348350 |

複数のtd_client_idの値で絞り込みを行い場合は，ORをつなげていきます。結果が確かに3種のtd_client_idしか含んでいないことを確認するために，以下のクエリを書きます。

このクエリは，まず内側のSELECT文で対象となるtd_client_idのレコードを絞り込み，外側のSELECT DISTINCT文でユニークなtd_client_idのリストを生成して列挙しています。

```sql
SELECT DISTINCT td_client_id
FROM
(
  SELECT td_client_id, time
  FROM sample_accesslog
  WHERE
     td_client_id = 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
  OR td_client_id = '7f47d05f-bd12-4553-e69c-763064738631'
  OR td_client_id = '780e865b-35f2-4e56-990a-5ed67cf733a3'
)
```
|td_client_id|
|------------|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|
|7f47d05f-bd12-4553-e69c-763064738631|
|780e865b-35f2-4e56-990a-5ed67cf733a3|


### td_client_idの特定の3ユーザーのみで，かつ特定のtimeより大きい

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE
( 
     td_client_id = 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
  OR td_client_id = '7f47d05f-bd12-4553-e69c-763064738631'
  OR td_client_id = '780e865b-35f2-4e56-990a-5ed67cf733a3'
)
AND time >= 1463640700
ORDER BY time
LIMIT 10
```
|td_client_id|time      |
|------------|----------|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640700|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640811|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640889|

複数のカラムを条件に指定する場合にANDとORを併用する場合，どこまでが条件の範囲なのかを「()」を用いて明示しないと異なる結果をもたらす可能性があります。上記の例は，「td_client_idが指定の3ユーザーで」かつ「timeが1463640700以上」であることを意味しています。
　　
### td_client_idの特定の3ユーザー「以外」のユーザーで，かつ特定のtimeより大きい

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE
NOT ( 
     td_client_id = 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
  OR td_client_id = '7f47d05f-bd12-4553-e69c-763064738631'
  OR td_client_id = '780e865b-35f2-4e56-990a-5ed67cf733a3'
)
AND time >= 1463640700
ORDER BY time
LIMIT 10
```
|td_client_id|time      |
|------------|----------|
|749cd3ba-e549-4fc7-9c83-0263999fa8bf|1463640703|
|749cd3ba-e549-4fc7-9c83-0263999fa8bf|1463640724|
|6c94bf0f-39de-4ee2-fc29-f0af74475b71|1463640735|

NOTを使うと判定を反転させることができます。2つ前の例と比べて，この例は「特定の3ユーザー「でない」td_client_id」かつ「timeが1463640700以上」であることを意味しています。これは以下のクエリと同義です。

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE
( 
      td_client_id != 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
  AND td_client_id != '7f47d05f-bd12-4553-e69c-763064738631'
  AND td_client_id != '780e865b-35f2-4e56-990a-5ed67cf733a3'
)
AND time >= 1463640700
ORDER BY time
LIMIT 10
```
|td_client_id|time      |
|------------|----------|
|749cd3ba-e549-4fc7-9c83-0263999fa8bf|1463640703|
|749cd3ba-e549-4fc7-9c83-0263999fa8bf|1463640724|
|6c94bf0f-39de-4ee2-fc29-f0af74475b71|1463640735|


NOTを使わない論理演算子による条件文が「特定の××による『絞り込み』」といえるのに対して，NOTを含む条件文は「特定の××を『除外』」といえます。条件を指定する目的が「絞り込み」か「除外」かをまず明確にすることで，論理演算子をうまく活用してください。もちろん，実際においては両者の組合せになることもあります。

# 範囲演算子BETWEENによる条件抽出 [ WHERE BETWEEN A AND B ]

カラムが数値型の場合，「A以上B以下」という範囲を指定したいことがよくあります。「>=」と「<=」を使って下記のように書けば実現できます。

```sql
time >= 1463640700 AND time <= 1463727100
```

しかし，BETWEENを用いて下記のように書くこともできます。

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE
( 
     td_client_id = 'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a'
  OR td_client_id = '7f47d05f-bd12-4553-e69c-763064738631'
  OR td_client_id = '780e865b-35f2-4e56-990a-5ed67cf733a3'
)
AND time BETWEEN 1463640700 AND 1463727100
ORDER BY time
LIMIT 10
```
|td_client_id|time      |
|------------|----------|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640700|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640811|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640889|

また，BETWEENの前にNOTを付ければ，指定した範囲「外」のレコードを抽出できます。
BETWEENは文字列カラムに対しても使えますが，文字列の範囲というのは簡単ではないので使わないほうが無難です。カラムにNULLが入っていた場合はそのレコードは無条件でNULLを返します。

# IN演算子による条件抽出 [ WHERE IN (A,B)  ]

IN演算子を使い，リスト内に複数の値を列挙することで，同カラムの複数の値のいずれかに一致する条件を記述できます。これにより，冗長になりがちなORによる連述を簡潔に記述することができます。

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE
td_client_id IN 
( 
  'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a',
  '7f47d05f-bd12-4553-e69c-763064738631',
  '780e865b-35f2-4e56-990a-5ed67cf733a3'
)
AND time BETWEEN 1463640700 AND 1463727100
ORDER BY time
LIMIT 10
```
|td_client_id|time      |
|------------|----------|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640700|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640811|
|f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a|1463640889|

さらに，NOT INとすることで「除外」，すなわちtd_client_idからリスト内に含まれない値を抽出してこられます。（下記の例では，さらに特定のtime値の範囲で絞り込んでいます。）

```sql
SELECT td_client_id, time
FROM sample_accesslog
WHERE
td_client_id NOT IN 
( 
  'f4d99634-4f1f-4e01-8d7f-9cd0a4a6613a',
  '7f47d05f-bd12-4553-e69c-763064738631',
  '780e865b-35f2-4e56-990a-5ed67cf733a3'
)
AND time BETWEEN 1463640700 AND 1463727100
ORDER BY time
LIMIT 10
```
|td_client_id|time      |
|------------|----------|
|749cd3ba-e549-4fc7-9c83-0263999fa8bf|1463640703|
|749cd3ba-e549-4fc7-9c83-0263999fa8bf|1463640724|
|6c94bf0f-39de-4ee2-fc29-f0af74475b71|1463640735|

## IS NULLの重要性 [ WHERE A IS (NOT) NULL ]

SQLの世界には，NULLという，（文字列的な意味で）「''」でも0でもFALSEでもない，そもそも値でない「印」があり，これによりカラムのレコードが表現されていることがあります。

NULLは非常に重要かつセンシティブな概念なので，しっかりと理解しておく必要があります。以下の表にまとめた内容を頭の片隅に入れてもらうため，1つひとつ結果を確かめていきましょう。

まず，以下のクエリにより，aカラムの各行にFALSE，TRUE，NULLという値がそれぞれ入ったテーブルを出力します。

```sql
SELECT a
FROM ( VALUES FALSE, TRUE, NULL ) AS t(a)
```
|a         |
|----------|
|false     |
|true      |
|NULL      |

結果を眺めていきましょう。まず，FALSEとNULLは似ているようで違います。FALSEは型を持ったきちんとした値です。一方，NULLは値でない無を表現する「印」です。SQLにおいては，以下の点をきちんと理解する必要があります。

- FALSEとNULLを比較する演算子には一致するものが存在しないこと
- FALSEをNULLと比較するときは「=」でなく「IS NULL」を使うこと

このテーブルに対して，aカラムから値がNULLであるものを抽出するクエリを考えます。まず間違った例を紹介します。

```sql
SELECT a
FROM ( VALUES FALSE, TRUE, NULL ) AS t(a)
WHERE a = NULL
```

上記の結果は何も返ってきません。なぜなら，以下の2点の特筆すべき挙動を取るからです：

- NULLと，あらゆる型の値との「=」による比較は，常にNULLを返す
- WHEREの条件式にTRUEかFALSEではなくNULLを返す場合は，それを見ない

上の間違ったクエリでは「a = NULL」の結果がすべてNULLなので，すべてのレコードが見られません。カラムの値とNULLを比較するには「a IS NULL」もしくは「a is NOT NULL」を使います。

以下の例では，3行目のNULLのレコードのみを返します。

```sql
SELECT a
FROM ( VALUES FALSE, TRUE, NULL ) AS t(a)
WHERE a IS NULL
```
|a         |
|----------|
|NULL      |


以下の例では，1行目と2行目のNULLでないレコードのみを返します。

```sql
SELECT a
FROM ( VALUES FALSE, TRUE, NULL ) AS t(a)
WHERE a IS NOT NULL
```
|a         |
|----------|
|false     |
|true      |

次は以下を確かめましょう。
- 0とNULLを比較する演算子には一致するものが存在しないこと
- 0をNULLと比較するときは「=」でなく「IS NULL」を使うこと

そのために，まずは0，1，NULLの3行からなるaカラムを作ります。

```sql
SELECT a
FROM ( VALUES 0, 1, NULL ) AS t(a)
```
|a         |
|----------|
|0     |
|1      |
|NULL      |

最初に，WHEREを使わずSELECT節に論理式を書く形で比較結果を確認してみます。

```sql
SELECT a, a = NULL AS eq_null, a IS NULL AS is_null
FROM ( VALUES 0, 1, NULL ) AS t(a)
```
|a         |eq_null|is_null|
|----------|-------|-------|
|0         |NULL   |false  |
|1         |NULL   |false  |
|NULL      |NULL   |true   |


結果からわかるように「0 IS NULL」はFALSEとなります。反転した「0 IS NOT NULL」はTRUEとなります。もう少し例を見てみましょう。
下記の例は，結果が何も得られません。

```sql
SELECT a
FROM ( VALUES 0, 1, NULL ) AS t(a)
WHERE a = NULL
```

下記の例は，3行目のNULLのみが返ります。

```sql
SELECT a
FROM ( VALUES 0, 1, NULL ) AS t(a)
WHERE a IS NULL
```
|a         |
|----------|
|NULL      |

下記の例は，1行目と2行目の0および1が返ります。

```sql
SELECT a
FROM ( VALUES 0, 1, NULL ) AS t(a)
WHERE a IS NOT NULL
```
|a         |
|----------|
|0     |
|1      |


最後に，長さ0の文字列「''」について下記を確認しましょう。
- 「''」とNULLを比較する演算子には一致するものが存在しないこと
- 「''」をNULLと比較するときは「=」でなく「IS NULL」を使うこと

そのために，以下の結果を見てみます。

```sql
SELECT a, a = NULL AS eq_null, a IS NULL AS is_null
FROM ( VALUES '', ' ', NULL ) AS t(a)
```
|a         |eq_null|is_null|
|----------|-------|-------|
|          |NULL   |false  |
|          |NULL   |false  |
|          |NULL   |true   |

余談ですが，空文字と半角スペースの比較「'' = ' '」はFALSEになります。

## IS DISTINCT FROMの重要性 [ WHERE A IS (NOT) DISTINCT FROM B ]

前回の課題では，カラムaの値にあるNULLを正しく比較するための条件節「WHERE IS」を考えました。今回は，NULLが入っているかもしれないカラムaとカラムbの正しい比較を考えます。

ここで「正しい比較」とは，NULLとそれ以外の値は「異なる」と判定し，NULL同士は「同じ」と判定するという意味です。先程までの結果では，NULLとの比較では他の値に関係なくNULLを返していました。そうではなく，FALSEまたはTRUEを返したいということです。

これから試してみる条件節とその結果を以下の表にまとめます。前提として，カラムaとbは同じ型の値であるとします。

前回の課題の文脈から，賢明な読者には「a = bではうまくいかない」ことは想像できるかもしれません。「IS」を使った結果については想像できるでしょうか？ それらを順番に確かめてみましょう。まずは必要なテーブルを用意します。

```sql
SELECT a, b
FROM 
( 
  VALUES
    (1, 1), (1, 2), (1, NULL), (NULL, NULL)
) AS t(a,b)
```
|a         |b   |
|----------|----|
|1         |1   |
|1         |2   |
|1         |NULL|
|NULL      |NULL|

上記のクエリは，カラムaおよびbが以下の値の組合せになるテーブルを出力します。

このテーブルに対して，「aとbの値が同じレコード（または異なるレコード）」だけを絞り込むにはどうすればいいかを考えます。ここでいう「同じ（異なる）」とは，NULLとそれ以外のあらゆる値は異なり，NULLとNULLは同じであるという意味です。

まっさきに考えられるクエリは「a = b」および「a <> b」ですが，以下のように所望の結果は得られません。

### NULLとNULLを同じとみなしてくれない（無条件にNULL）

```sql
SELECT a, b
FROM 
( 
  VALUES (1, 1), (1, 2), (1, NULL), (NULL, NULL)
) AS t(a,b)
WHERE a = b
```
|a         |b   |
|----------|----|
|1         |1   |

aとbがともにNULLである4行目のレコードが選択されるべきですが，結果には現れていません。

### NULLとその他の値を異なるとみなしてくれない（無条件にNULL）

```sql
SELECT a, b
FROM 
( 
  VALUES (1, 1), (1, 2), (1, NULL), (NULL, NULL)
) AS t(a,b)
WHERE a <> b
```
|a         |b   |
|----------|----|
|1         |2   |


3行目の1とNULLが異なるものとされるべきですが，同様に結果に現れていません。

このように，NULLが入っているレコードにおいては，「=」および「<>」で比較した場合に所望の結果は得られません。

次に，ISを使うことを考えてみましょう。正しいと思われるかもしれない以下のクエリですが，エラーとなります。

### A IS [NOT] B は使えない

```sql
SELECT a, b
FROM
( 
  VALUES (1, 1), (1, 2), (1, NULL), (NULL, NULL)
) AS t(a,b)
WHERE a IS b -- エラー（a IS NOT NULLも同様）
```

ISは，前回のようにaと明示的なNULLとの比較「a IS NULL」には使えますが，NULLかどうかが不明なbとの比較「a IS b」には使えないのです。

その代わりに用いられるのが「a IS NOT DISTINCT FROM b」です。DISTINCTは「異なる」という意味なので，aとbが同じであるかを見るにはNOTを付ける必要があります。

### NULLとNULLを同じとみなしてくれる比較クエリ

```sql
SELECT a, b
FROM
( 
  VALUES (1, 1), (1, 2), (1, NULL), (NULL, NULL)
) AS t(a,b)
WHERE a IS NOT DISTINCT FROM b
```
|a         |b   |
|----------|----|
|1         |1   |
|NULL      |NULL|


両方ともNULLの組合せも「同じ」として選択されています。NULLに関しても異なるものを絞り込むクエリは下記のようになります。

### NULLと他の値を異なるとみなしてくれる比較クエリ

```sql
SELECT a, b
FROM
( 
  VALUES (1, 1), (1, 2), (1, NULL), (NULL, NULL)
) AS t(a,b)
WHERE a IS DISTINCT FROM b
```
|a         |b   |
|----------|----|
|1         |2   |
|1         |NULL|

こちらも1とNULLが「異なる」として選択されています。

## LIKEによる文字列の部分一致 [ WHERE A LIKE pattern ]

文字列カラムに対して「=」により完全に一致するものを絞り込むのではなく，ある文字列を含む場合やあるパターンを持つ場合に絞り込みを行いたいケースがよくあります。

例えば，td_urlやtd_referrerなどのURLに関する文字列や，様々なOSのバージョンを含むtd_osのような文字列に対して統括的な処理を実行したい場合があります。

実は，SQLでは文字列のパターンマッチについて非常に柔軟な記述が可能です。ただし，ここでは基本的な内容の紹介にとどめ，より複雑な内容は後の章で詳しく丁寧に述べることにします。

OSのバージョンが入ったtd_osカラムに対して，Windowsユーザー（バージョン問わず）を絞り込むことを考えます。

本格的な正規表現を必要としない簡単なパターンマッチによる部分一致にはLIKE演算子が使えます（PrestoではLIKE演算子を内部的に正規表現を扱うREGEXP_LIKE関数に変更されています）。ここで言う「簡単」とは，以下のパターンに対するマッチに限られるという意味です。

- 一部の文字を曖昧にできる
- 〜で始まる，〜で終わる
- （複数の）〜を含む

順に見ていきましょう。
まず，td_osに入っている文字列が「windows」なのか「Windows」なのか「WINDOWS」なのかが自明ではありません。そこで下記のクエリにより，どのパターンで値が入っているか当たりをつけます。

```sql
SELECT DISTINCT td_os
FROM sample_accesslog
WHERE td_os LIKE '_indows%'
```
|td_os     |
|----------|
|Windows 7 |
|Windows 8 |
|Windows Phone|
|Windows XP|
|Windows Vista|
|Windows RT 8.1|
|Windows   |
|Windows 8.1|


正解は「Windows」のようですね。

上記では，LIKEに厳密な文字列一致ではなく「曖昧性」を持った条件を指定するために，ワイルドカードと呼ばれる特殊な記号が使っています。
- 「%」：何らかの文字列（空文字でも空白文字でもよい）
- 「_」：何かしらの1文字（空白文字でもよい）

LIKEの後に記述される上記の記号を含んだ文字列は「パターン」と呼ばれることがあります。
パターンについてもう少し細かく見ていきましょう。

```sql
SELECT DISTINCT td_os
FROM sample_accesslog
WHERE td_os LIKE 'windows'
```

上記のクエリはヒットしません。なぜなら，この条件で求められているのは文字列全体が厳密に「windows」に一致することで，今の例では小文字のみの「windows」の文字列は含まれていないからです。

```sql
SELECT DISTINCT td_os
FROM sample_accesslog
WHERE td_os LIKE 'Windows'
```
|td_os     |
|----------|
|Windows   |


上記のクエリでは，厳密に「Windows」の文字列だけを含むレコード（前の結果における6行目のレコード）が1つだけがヒットします。「%」や「_」を含まない文字列に対しては，「=」と同じ挙動となっていることがわかります。

```sql
SELECT DISTINCT td_os
FROM sample_accesslog
WHERE td_os LIKE 'Windows%' --または '%windows%'
```
|td_os     |
|----------|
|Windows 8.1|
|Windows RT 8.1|
|Windows 7 |
|Windows 8 |
|Windows Vista|
|Windows   |
|Windows Phone|
|Windows XP|


上記のクエリでは，様々なバージョンのWindowsユーザーすべてにヒットすることができます。「%」は空文字であってもよいので「Windows」だけの文字列もヒットします。
次に，「Windows 8」と「Windows 8.1」と「Windows RT 8.1」のみにヒットするパターンを考えます。

```sql
SELECT DISTINCT td_os
FROM sample_accesslog
WHERE td_os 
LIKE 'Windows%8%'
```
|td_os     |
|----------|
|Windows 8.1|
|Windows RT 8.1|
|Windows 8 |


さらに「Windows RT 8.1」を除外したい場合はどう書けばよいでしょうか？ 正解は，「8」と「8.1」では「Windows」と「8」の間に1つの空白文字だけがあるのに対し，「RT 8.1」では間に「RT」と空白文字が入っていることに着目して，パターンを「Windows 8%」と書くか，もしくは「_」を利用して下記のように書くことです。

```sql
SELECT DISTINCT td_os
FROM sample_accesslog
WHERE td_os 
LIKE 'Windows_8%'
```
|td_os     |
|----------|
|Windows 8.1|
|Windows 8 |

最後に，いくつか注意点を挙げておきます。
- 「%」や「_」そのものを純粋な文字としてパターンに含ませる場合には，「\\%」および「\\_」と書く
- LIKEによる曖昧検索は正規表現による検索よりも遅い

## このLessonで注意したい点まとめ

- WHEREの後にはTRUEまたはFALSEを返す条件式を記述する
- ANDやORが並列する場合には対象範囲を「( )」で囲んで明確化する
- 1つのカラムの複数の値の中での一致にはIN演算子が便利
- 範囲演算子BETWEENは，「A以上B以下」（両端A，Bを含む）の数値型および文字列型で使用可
- 条件式がNULLの場合のWHEREの挙動に注意（そのレコードは見られない）
- NULLとカラム値を比較する際，「IS [NOT] NULL」や「=」では無条件でNULLになる
- カラム値にNULLが入っているかもしれないカラム同士の比較は「IS [NOT] DISTINCT FROM」を使う
- LIKEによる曖昧検索は簡潔である代わりに柔軟性とパフォーマンスに劣る
