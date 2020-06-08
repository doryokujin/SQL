# Lesson 06. 文字列関数，正規表現

## 文字列の情報を得る関数
```sql
LENGTH(string) → bigint：文字列の長さを取得する
SELECT s, LENGTH(s) AS num_of_digits
FROM (
  VALUES '1','10','100','1000'
) AS t(s)
```

### CHR(n) → varchar：改行コードやタブを文字列に加える
nで指定した文字コード番号に対応する文字を返します。この関数は，改行文字など，クエリ内で直接記述できない文字を入力するために主に使用されます。
|文字  |文字コード表記|補足   |
|----|-------|-----|
|タブ  |CHR(9) |\t 相当|
|CR  |CHR(13)|\r 相当（CTRL+M）|
|LF  |CHR(10)|\n 相当（CTRL+J）|
|改行  |CHR(13) &#124;&#124; CHR(10)|Windowsプラットフォーム|
|    |CHR(10)|UNIX，LINUX系プラットフォーム|
|スペース|CHR(32)|     |
|引用符（'）|CHR(39)|     |

```sql
-- I  have a 'pen'.
-- I have an 'apple'.
SELECT 'I'||CHR(32)||CHR(32)||'have'||CHR(32)||'a'||CHR(32)||CHR(39)||'pen'||CHR(39)||'.'||CHR(10)||'I'||CHR(32)||'have'||CHR(32)||'an'||CHR(32)||CHR(39)||'apple'||CHR(39) ||'.' AS pico
```

### POSITION(substring IN string)，STRPOS(string, substring) → bigint：部分文字列の登場位置を調べる

ある文字列の部分文字列が，先頭を1として数えてどの位置から始まるかを返します。部分文字列に完全に一致しなかった場合には0を返します。POSITION(substring IN string)とSTRPOS(string, substring)の2つの関数で部分文字列の引数の位置の違いと記法が異なることに注意してください。

```sql
SELECT POSITION('pen' IN 'I have a pen') AS pos1, STRPOS('I have a pen', 'pen') AS pos2, STRPOS('pen', 'I have a pen') AS pos_wrong
```
### LEVENSHTEIN_DISTANCE(string1, string2)，HAMMING_DISTANCE(string1, string2) → bigint：2つの文字列の類似度を数値化する

HAMMING_DISTANCEは，文字数が等しい2つの文字列の中で，対応する位置にある異なる文字の個数を返します。この個数は主に2進数の比較に用いられます。

```sql
SELECT HAMMING_DISTANCE('1111111','1010101') AS dist_ham1, HAMMING_DISTANCE('lunch','punch') AS dist_ham2
```

文字数が異なる文字列を比較する場合や，文字の比較だけではなく挿入や削除が求められる場合には，より一般化されたLEVENSHTEIN_DISTANCEが使えます。

```sql
SELECT LEVENSHTEIN_DISTANCE(    'apple', 'pineapple') AS dist_lev1,
       LEVENSHTEIN_DISTANCE('pineapple',  'applepen') AS dist_lev2
```

## 文字列を整形する関数

### TRIM(string)，RTRIM(string)，LTRIM(string) → varchar：文字列の前後の余計な空白（タブ，改行）を除去する

両側に作用するTRIM，右側に作用するRTRIM，左側に作用するLTRIMがあります。下記のクエリでは，CHR(9)はタブ（→），CHR(10)は改行（⏎）を表します。

```sql
SELECT s, TRIM(s) AS trim_str, RTRIM(s) AS rtrim_str, LTRIM(s) AS ltrim_str
FROM ( VALUES CHR(9)||' Pine Apple '||CHR(10)) AS t(s)
```
|s   |trim_str|rtrim_str|ltrim_str   |
|----|--------|---------|------------|
|→Pine Apple ⏎|Pine Apple|→Pine Apple|Pine Apple ⏎|

### LOWER(string)，UPPER(string) → varchar：文字列の全部を小文字／大文字に変換する

```sql
SELECT LOWER(s) AS lower_str, UPPER(s) AS upper_str
FROM ( VALUES 'PineApple') AS t(s)
```

## 文字列を加工する関数

### ||，CONCAT(string1, ..., stringN) → varchar：文字列を結合する

オペレーターである「||」またはCONCAT関数により，複数の文字列を結合できます。

```sql
SELECT s1, s2, s3, s1||'_'||s2||'_'||s3 AS concat_str1, CONCAT(s1,'_',s2,'_',s3) AS concat_str2
FROM (
  VALUES 
    ('A', 'a', '1'),
    ('B', 'b', '2'),
    ('C', 'c', '3'),
    ('D', 'd', '4')
) AS t(s1,s2,s3)
```

### SUBSTR(string, start, length) → varchar：部分文字列を抜き出す

先頭は1として，指定した位置から始まる部分文字列を抜き出します。第3引数で抜き出す長さを指定できます。長さを指定しない場合は末尾まで抜き出します。また，開始位置を負の数にすると，最後から順に数えることになります。つまり，-1は1番最後の文字，-nは最後から順にn番目の文字を指します。

```sql
SELECT s, SUBSTR(s,1,7) AS os, TRIM(SUBSTR(s,8)) AS version
FROM (
  VALUES
    'Windows',
    'Windows 7',
    'Windows 8',
    'Windows 8.1',
    'Windows Phone',
    'Windows RT 8.1',
    'Windows Vista',
    'Windows XP'
) AS t(s)
```

### LPAD(string, size, padstring)，RPAD(string, size, padstring) → varchar：特定の長さになるまで文字列を埋める
padstringで指定したsizeまでstringで埋めます。下記は，数値文字（最大1000）を並び替え可能なように4桁の文字列に変換するクエリです。

```sql
SELECT s, LPAD(s,4,'0') AS pad_str
FROM (
  VALUES '1','10','100','1000'
) AS t(s)
```

### （応用）異なる接頭辞と異なる桁数の数値からなる文字列を並び替え可能にする

実用例として，「am4」や「fm1232」といった（異なる）接頭辞と異なる桁数の数字が結合された文字列を，並び替え可能な文字列に変換するクエリを考えます。
まず，元の文字列を接頭辞と数値部分に分けます。この処理では，後述の正規表現を使い，文字列の先頭（先頭記号^）から非数値（[^0-9]*）の部分を接頭辞として，（接尾辞がないことを仮定して）数値が始まるところから文字列最後（最終記号$）までを数値部分として抜き出します。

```sql
WITH sample AS
(
  SELECT s
  FROM (
    VALUES 'am1','fm10','am100','fm1000'
  ) AS t(s)
)

SELECT REGEXP_EXTRACT(s,'^[^0-9]*') AS prefix, REGEXP_EXTRACT(s,'[0-9]*$') AS num_str
FROM sample
```

次に，存在する数値部分のうちで最長の桁数を求めます。それより桁が少ない数値部分は，この最長の桁数分になるまで0で埋められることになります。

```sql
WITH sample AS
(
  SELECT s
  FROM (
    VALUES 'am1','fm10','am100','fm1000'
  ) AS t(s)
),
split_sample AS
(
  SELECT REGEXP_EXTRACT(s,'^[^0-9]*') AS prefix, REGEXP_EXTRACT(s,'[0-9]*$') AS num_str
  FROM sample
)

SELECT MAX(length(num_str)) AS max_len
FROM split_sample
```

最後に，LPADですべての行の数値部分が4桁になるように左から0で埋め，接頭辞を前から結合して完成です。全体のクエリは以下になります。

```sql
WITH sample AS
(
  SELECT s
  FROM (
    VALUES 'am1','fm10','am100','fm1000'
  ) AS t(s)
),
split_sample AS
(
  SELECT REGEXP_EXTRACT(s,'^[^0-9]*') AS prefix, REGEXP_EXTRACT(s,'[0-9]*$') AS num_str
  FROM sample
),
stat AS
(
  SELECT MAX(length(num_str)) AS max_len
  FROM split_sample
)

SELECT prefix || LPAD(num_str, max_len, '0') AS s_can_sort
FROM split_sample,stat
ORDER BY s_can_sort
```

### SPLIT(string, delimiter) → array，SPLIT_PART(string, delimiter, index) → varchar：文字列を分割する

SPLIT関数は，指定した区切り文字によって文字列を分割する関数です。返り値はLISTとなることに注意してください。LISTの要素を直接取り出して使いたい場合には，SPLIT_PART関数を利用して分割後のLISTの要素番号（1から開始）を指定します。

```sql
SELECT s, 
  SPLIT(s,' ') AS splitted_strs, 
  SPLIT_PART(s,' ',1) AS os, 
  SPLIT_PART(s,' ',2) AS version
FROM (
  VALUES
    'Windows',
    'Windows 7',
    'Windows 8',
    'Windows 8.1',
    'Windows Phone',
    'Windows RT 8.1',
    'Windows Vista',
    'Windows XP'
) AS t(s)
```

## 正規表現の基礎を学ぶ
「LIKEによる文字列の部分一致」では，文字列カラムの完全一致ではなく，ワイルドカードを用いた曖昧な一致の方法を見ました。以降では，曖昧さを残して文字列カラムと一致させるための記述を「パターン（正規表現）」と呼び，パターンに合う文字列を探し出すことを「パターンマッチ」と呼ぶことにします。
正規表現を使うことで，ワイルドカードよりはるかに柔軟で豊富な記号により，パターンマッチの有用性を高めることができます。正規表現はSQLに限らず様々なプログラミング言語に共通する手段なので覚えておいて損はありません。
以下，正規表現のパターン内に記述できる文字（記号）を紹介し，それぞれどのような文字にマッチするかを確認していきます。

### 正規表現文字

「正規表現文字」，はタブや改行文字など，直接キーボードで打ち込めない特殊な文字をパターンに記述するための記号です。「\」に小文字アルファベット1文字で記述します。
|構文  |Matches|
|----|-------|
|\t  |タブ文字（'\u0009'）|
|\n  |改行文字（'\u000A'）|
|\r  |キャリッジリターン文字（'\u000D'）|
|\f  |用紙送り文字（'\u000C'）|
|\a  |警告（ベル）文字（'\u0007'）|
|\e  |エスケープ文字（'\u001B'）|

### 文字クラス（基本）

「文字クラス」は，複数の文字のいずれかに一致すればよいという意図で用いられる記号で，[ ]内に一致させたい文字を列挙したり，文字の範囲を指定したりすることで記述します。[ ]内の先頭に「^」を付けることで否定（「含まない」）の意味になります。
|構文  |Matches|
|----|-------|
|[abc]|a，b，またはc|
|[^abc]|a，b，c以外の文字（否定）|
|[a-zA-Z]|a〜zまたはA〜Z（範囲）|
|[a-d[m-p]]|a〜dまたはm〜p（[a-dm-p]と同じ）（結合）|
|[a-z&&[def]]|d，e，f（交差）|
|[a-z&&[^bc]]|bとcを除くa〜z（ [ad-z]と同じ）（減算）|
|[a-z&&[^m-p]]|m〜pを除くa〜z（ [a-lq-z]と同じ）（減算）|

### 定義済みの文字クラス

文字クラスには予め定義済みのものがあり，その多くは「\」とアルファベット1文字記述できます。その意味で正規表現文字と似ていますが，1つの記号で複数の文字にマッチする点が異なります。なお，「\」と大文字のアルファベットで表される文字クラスは，対応する小文字で表される文字クラスの否定となっています。また，最も多用されるであろう文字クラスは「.」で，これはワイルドカードでいう「_」のように任意の1文字にマッチします。
|構文  |Matches|
|----|-------|
|.   |任意の文字（行末記号とマッチする場合もある）|
|\d  |数字：[0-9]|
|\D  |数字以外：[^0-9]|
|\s  |空白文字：[\t\n\x0B\f\r]|
|\S  |非空白文字：[^\s]|
|\w  |単語構成文字：[a-zA-Z0-9_]（アルファベット，数字，_）|
|\W  |非単語文字：[^\w]|

これらの文字クラスを用いた正規表現の簡単な例を見ていきましょう。

#### 任意の文字が2文字目にくる4文字のパターン
```sql
'a.cd'
```
マッチする文字列
```sql
abcd
a1cd
a_cd
```
マッチしない文字列
```sql
abbcd # a と cd の間に2文字以上ある
abc   # d がない
acd   # a と cd の間に0文字
```
確認クエリ
```sql
SELECT str, REGEXP_LIKE(str,'a.cd') AS is_matched
FROM (
  VALUES 'abcd', 'a1cd', 'a_cd', 'abbcd', 'abc', 'acd'
) AS t(str)
```

#### 数字が2文字目にくる4文字のパターン
```sql
'a[\d]cd', 'a[0-9]cd' 
```
マッチする文字列
```sql
a1cd
```
マッチしない文字列
```sql
abcd  # a と cd の間が数字でない
a1c   # c の後に d がない
a12cd # a と cd の間に2文字以上
```
確認クエリ
```sql
SELECT str, 
  REGEXP_LIKE(str,'a[\d]cd')  AS is_matched, 
  REGEXP_LIKE(str,'a[0-9]cd') AS is_matched
FROM (
  VALUES 'a1cd', 'abcd', 'a1c', 'a12cd'
) AS t(str)
```

#### 数字以外の文字が2文字目にくる4文字のパターン

```sql
'a[^\d]cd', 'a[\D]cd', 'a[^0-9]cd'
```

マッチする文字列
```sql
abcd
a_cd
```

マッチしない文字列
```sql
a1cd  # a と cd の間が数字
abbcd # a と cd の間に2文字以上ある
acd   # a と cd の間に0文字
```
確認クエリ
```sql
SELECT str, 
  REGEXP_LIKE(str,'a[^\d]cd')  AS is_matched, 
  REGEXP_LIKE(str,'a[\D]cd')   AS is_matched,
  REGEXP_LIKE(str,'a[^0-9]cd') AS is_matched
FROM (
  VALUES 'abcd', 'a_cd', 'a1cd', 'abbcd', 'acd'
) AS t(str)
```

### 数量子（1. 最長一致）
|構文  |Matches|
|----|-------|
|X?  |Xが1または0回|
|X*  |Xが0回以上 |
|X+  |Xが1回以上 |
|X{n}|Xがn回   |
|X{n,}|Xがn回以上 |
|X{n,m}|Xがn回以上，m回以下|

まずは「?」を含むパターンについて例を用いて検証していきましょう。

#### bの0〜1回の繰り返しを挟むパターン

```sql
'ab?cd'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[acd]
[abcd]
[abcd][acd]
```

マッチしない文字列

```sql
abbcd   # b が2回以上
abbbcd  # b が2回以上
ab1cd   # a と cd の間に b と b 以外の文字がある
```

確認クエリ

```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'ab?cd') AS matched
FROM (
  VALUES 'acd', 'abcd', 'abcdacd', 'abbcd', 'abbbcd', 'ab1cd'
) AS t(str)
```

#### bcの0〜1回の繰り返しを挟むパターン

1文字の繰り返しでなく，文字列の繰り返しを記述するためには，繰り返したい文字列を「( )」で括って次のようにします。
```sql
'a(bc)?d'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[ad]     # (bc) は含まれなくてもよい
[abcd]
```

マッチしない文字列

```sql
abd    # (bc) がない
acd    # (bc) がない
abce   # a(bc) の後が d でなく e
abcexd # a(bc) と d の間の x が邪魔
abcbce # (bc) が2回挟まっている
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'a(bc)?d') AS matched
FROM (
  VALUES 'ad', 'abd', 'acd', 'abcd', 'abce', 'abcxd', 'abcbcd'
) AS t(str)
```

#### bかcの0〜1回の繰り返しを挟むパターン

複数の文字のいずれかの繰り返しを記述するためには，「[ ]」による文字クラスを使います。

```sql
'a[bc]?d'
```

マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
[ad]
[abd]
[acd]
[abd]e
```
マッチしない文字列
```sql
abcd    # b か c が2回挟まっている
abbd    # b か c が2回挟まっている
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'a[bc]?d') AS matched
FROM (
  VALUES 'ad', 'abd', 'acd', 'acde', 'abcd', 'abbd'
) AS t(str)
```

#### 0〜1回の数字の繰り返しを挟むパターン

数字などの文字クラスも繰り返しの対象にできます。
```sql
'X[\d]?Y[\d]?'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[XY]
[X1Y]
[X1Y2]3
```

マッチしない文字列
```sql
X123Y # X と Y の間に数字が3回挟まっている
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'X[\d]?Y[\d]?') AS matched
FROM (
  VALUES 'XY','X1Y','X123Y','X1Y23'
) AS t(str)
```

ここまでは「?」を含むパターンの例を見てきました。ここからは，「*」（0回以上の繰り返し）を含むパターンについて例を使って検証していきます。

#### bの0回以上の繰り返しを挟むパターン
```sql
'ab*cd'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[acd]
[abcd]
[abcd][acd]
[abbcd]
[abbbcd]
```

マッチしない文字列

```sql
ab1cd   # a と cd の間に b と b 以外の文字がある
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'ab*cd') AS matched
FROM (
  VALUES 'acd', 'abcd', 'abcdacd', 'abbcd', 'abbbcd', 'ab1cd'
) AS t(str)
```

#### bcの0回以上の繰り返しを挟むパターン

```sql
'a(bc)*d'
```

マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
[ad]     # (bc) は含まれなくてもよい
[abcd]
[abcbcd]
```

マッチしない文字列

```sql
abd    # (bc) がない
acd    # (bc) がない
abcxd  # a(bc) と d の間の x が邪魔
abcbce # a(bc)(bc) の後が d でなく e
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'a(bc)*d') AS matched
FROM (
  VALUES 'ad', 'abd', 'acd', 'abcd', 'abcxd', 'abcbcd', 'abcbce'
) AS t(str)
```

#### bかcの0回以上の繰り返しを挟むパターン
```sql
'a[bc]*d'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[ad]
[abd]
[acd]
[abcd]
[abcbbcd]
```

マッチしない文字列
```sql
abce    # d で終わらず e で終わっている
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'a[bc]*d') AS matched
FROM (
  VALUES 'ad', 'abd', 'acd', 'abcd', 'abcbbcd', 'abce'
) AS t(str)
```

#### 数字の0回以上の繰り返しを挟むパターン

```sql
'X[\d]*Y[\d]*'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[XY]
[X1Y]
[X1Y23]
[X123Y]
```

マッチしない文字列

```sql
X12Z34Y # 12 と 34 の間に文字が挟まっている
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'X[\d]*Y[\d]*') AS matched
FROM (
  VALUES 'XY','X1Y','X123Y','X1Y23','X12Z34Y'
) AS t(str)
```

ここからは，「+」（1回以上の繰り返し）を含むパターンを例を用いて検証します。

#### bの1回以上の繰り返しを挟むパターン
```sql
'ab+cd'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[abcd]
[abcd]acd
[abbcd]
[abbbcd]
```

マッチしない文字列

```sql
acd   # a と cd の間に b がない
ab1cd # a と cd の間に b と b 以外の文字がある
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'ab+cd') AS matched
FROM (
  VALUES 'acd', 'abcd', 'abcdacd', 'abbcd', 'abbbcd', 'ab1cd'
) AS t(str)
```

#### abの1回以上の繰り返しを挟むパターン
```sql
'(ab)+cd'
```

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
[abcd]   # (ab)cd
[ababcd] # (ab)(ab)cd
```

マッチしない文字列
```sql
bcd     # (ab) ではなく b のみ
abbcd   # (ab) と cd の間に (ab) でなく b のみ
ababxcd # (ab)(ab) と cd の間の x が邪魔
```
確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'(ab)+cd') AS matched
FROM (
  VALUES 'bcd', 'abcd', 'abbcd', 'ababcd', 'ababxcd'
) AS t(str)
```
#### bかcの1回以上の繰り返しを挟むパターン

```sql
'a[bc]+d'
```
マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
[abd]
[acd]
[abcd]
[abcbd]
```
マッチしない文字列

```sql
ad    # b か c が 1 回も挟まらない
abce  # d で終わらず e で終わっている
```

確認クエリ

```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'a[bc]+d') AS matched
FROM (
  VALUES 'ad', 'abd', 'acd', 'abcd', 'abce', 'abcbd'
) AS t(str)
```

#### 数字の1回以上の繰り返しを挟まむパターン
```sql
'X[\d]+Y[\d]*'
```
マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
[X1Y]
[X1Y23]
[X123Y]
```
マッチしない文字列
```sql
XY      # 数字が1回も挟まっていない
X12Z34Y # 12 と 34 の間に文字が挟まっている
```
確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'X[\d]+Y[\d]*') AS matched
FROM (
  VALUES 'XY','X1Y','X123Y','X1Y23','X12Z34Y'
) AS t(str)
```

ここからは，{n,m}（n回以上m回以下の繰り返し）を含むパターンについて例を用いて検証します。

#### bに対する{n,m}の繰り返しを挟んだパターンの例

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
'ab{3}cd'   # b が3回（ちょうど）含まれる ⇒ [abbbcd]
'ab{1,}cd'  # b が1回以上含まれる ⇒ [abcd], [abcd]acd, [abbcd], [abbbcd]
'ab{1,2}cd' # b が1回から2回含まれる ⇒ [abcd], [abcd]acd, [abbcd]
'ab{0,2}cd' # b が0回から2回含まれる ⇒ [acd], [abcd], [abcd][acd], [abbcd]
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'ab{3}cd')   AS m1,
  REGEXP_EXTRACT_ALL(str,'ab{1,}cd')  AS m2,
  REGEXP_EXTRACT_ALL(str,'ab{1,2}cd') AS m3,
  REGEXP_EXTRACT_ALL(str,'ab{0,2}cd') AS m4
FROM (
  VALUES 'acd', 'abcd', 'abcdacd', 'abbcd', 'abbbcd', 'ab1cd'
) AS t(str)
```

#### (ab)に対する{n,m}の繰り返しを挟んだパターンの例
マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
'(ab){2}cd'   # (ab) が2回（ちょうど）含まれる ⇒ [(ab)(ab)cd]
'(ab){1,}cd'  # (ab) が1回以上含まれる ⇒ [(ab)cd], [(ab)(ab)cd]
'(ab){1,2}cd' # (ab) が1回から2回含まれる ⇒ [(ab)cd], [(ab)(ab)cd]
'(ab){0,2}cd' # (ab) が0回から2回含まれる ⇒ a[cd], b[cd], [abcd], (ab)b[cd], ab1[cd]
```
確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'(ab){2}cd')   AS m1,
  REGEXP_EXTRACT_ALL(str,'(ab){1,}cd')  AS m2,
  REGEXP_EXTRACT_ALL(str,'(ab){1,2}cd') AS m3,
  REGEXP_EXTRACT_ALL(str,'(ab){0,2}cd') AS m4
FROM (
  VALUES 'acd', 'bcd','abcd', 'abbcd', 'ababcd', 'ab1cd'
) AS t(str)
```

#### 数字に対する{n,m}の繰り返しを挟んだパターンの例

マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
# 03-123-4567
'[\d]{3}'   # ⇒ 03-[123]-[456]
'[\d]{5}'   # ⇒ []
'[\d]{1,}'  # ⇒ [03]-[123]-[4567]
'[\d]{1,2}' # ⇒ [03]-[12][3]-[45][67]
'[\d]{0,2}' # ⇒ [03]-[12][3]-[45][67]
```

確認クエリ

```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'[\d]{3}')   AS m1,
  REGEXP_EXTRACT_ALL(str,'[\d]{5}')   AS m2,
  REGEXP_EXTRACT_ALL(str,'[\d]{1,}')  AS m3,
  REGEXP_EXTRACT_ALL(str,'[\d]{1,2}') AS m4,
  REGEXP_EXTRACT_ALL(str,'[\d]{0,2}') AS m5
FROM (
  VALUES '03-123-4567'
) AS t(str)
```

### 数量子（2. 控えめなものと強欲なもの）
数量子には「控えめ」な数量子と「強欲」な数量子が存在します。普通の数量子との違いは，結果的にマッチする文字列をパターンの最小の範囲にとどめるか，パターンの最大の範囲まで取ってくるかにあります。
控えめな数量子を以下に示します。
|構文  |Matches|
|----|-------|
|X?? |Xの1または0回の最短の繰り返し|
|X*? |Xの0回以上の最短の繰り返し|
|X+? |Xの1回以上の最短の繰り返し|
|X{n}?|Xのn回の最短の繰り返し|
|X{n,}?|Xのn回以上の最短の繰り返し|
|X{n,m}?|Xのn回以上，m回以下の最短の繰り返し|

強欲な数量子を以下に示します。
|構文  |Matches|
|----|-------|
|X?+ |Xの1または0回の最大の繰り返し|
|X*+ |Xの0回以上の最大の繰り返し|
|X++ |Xの1回以上の最大の繰り返し|
|X{n}+|Xのn回の最大の繰り返し|
|X{n,}+|Xのn回以上の最大の繰り返し|
|X{n,m}+|Xのn回以上，m回以下の最大の繰り返し|

多くの場合，普通の繰り返しは強欲な繰り返しと結果が同じになります。一方，控えめな一致では常に最短の文字列にしかマッチしなくなるので，意識して使い分けるシーンが出てきます。ここでは，普通の繰り返し，控えめな繰り返し，強欲な繰り返しのそれぞれのパターンの例を検証します。
bに対する3種類の繰り返しを行うパターンの例
マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
# acd
'ab+'  # ⇒ []
'ab+?' # ⇒ []
'ab++' # ⇒ []

'ab*'  # ⇒ [a]
'ab*?' # ⇒ [a]
'ab*+' # ⇒ [a]

'ab?'  # ⇒ [a]
'ab??' # ⇒ [a]
'ab?+' # ⇒ [a]

# abcd
'ab+'  # ⇒ [ab]
'ab+?' # ⇒ [ab]
'ab++' # ⇒ [ab]

'ab*'  # ⇒ [ab] 1回
'ab*?' # ⇒ [a]  0回
'ab*+' # ⇒ [ab] 1回

'ab?'  # ⇒ [ab] 1回
'ab??' # ⇒ [a]  0回
'ab?+' # ⇒ [ab] 1回

# abbbcd
'ab+'  # ⇒ [abbb] 3回
'ab+?' # ⇒ [ab]   1回
'ab++' # ⇒ [abbb] 3回

'ab*'  # ⇒ [abbb] 3回
'ab*?' # ⇒ [a]    0回
'ab*+' # ⇒ [abbb] 3回

'ab?'  # ⇒ [ab] 1回
'ab??' # ⇒ [a]  0回
'ab?+' # ⇒ [ab] 1回
```

確認クエリ
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'ab+')  AS m1,       --普通
  REGEXP_EXTRACT_ALL(str,'ab+?') AS m1_short, --最短
  REGEXP_EXTRACT_ALL(str,'ab++') AS m1_long,  --強欲
  
  REGEXP_EXTRACT_ALL(str,'ab*')  AS m2,       --普通
  REGEXP_EXTRACT_ALL(str,'ab*?') AS m2_short, --最短
  REGEXP_EXTRACT_ALL(str,'ab*+') AS m2_long,  --強欲
  
  REGEXP_EXTRACT_ALL(str,'ab?')  AS m3,       --普通
  REGEXP_EXTRACT_ALL(str,'ab??') AS m3_short, --最短
  REGEXP_EXTRACT_ALL(str,'ab?+') AS m3_long   --強欲
FROM (
  VALUES 'acd', 'abcd', 'abbcd', 'abbbcd'
) AS t(str)
```

#### 数字の3種類の繰り返しに対応したパターンの例

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
# 03-123-4567
'[\d]+'   # ⇒ [03]-[123]-[4567]
'[\d]+?'  # ⇒ [0][3]-[1][2][3]-[4][5][6][7]
'[\d]++'  # ⇒ [03]-[123]-[4567]

'[\d]*'   # ⇒ [03]-[123]-[4567]
'[\d]*?'  # ⇒ []
'[\d]*+'  # ⇒ [03]-[123]-[4567]

'[\d]?'   # ⇒ [0][3]-[1][2][3]-[4][5][6][7]
'[\d]??'  # ⇒ []
'[\d]?+'  # ⇒ [0][3]-[1][2][3]-[4][5][6][7]
```

確認クエリ（結果は省略）

```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'[\d]+')  AS m1,
  REGEXP_EXTRACT_ALL(str,'[\d]+?') AS m1_short,
  REGEXP_EXTRACT_ALL(str,'[\d]++') AS m1_long,

  REGEXP_EXTRACT_ALL(str,'[\d]*')  AS m2,
  REGEXP_EXTRACT_ALL(str,'[\d]*?') AS m2_short,
  REGEXP_EXTRACT_ALL(str,'[\d]*+') AS m2_long,
  
  REGEXP_EXTRACT_ALL(str,'[\d]?')  AS m3,
  REGEXP_EXTRACT_ALL(str,'[\d]??') AS m3_short,
  REGEXP_EXTRACT_ALL(str,'[\d]?+') AS m3_long
FROM (
  VALUES '03-123-4567'
) AS t(str)
```

#### 数字とハイフンの3種類の繰り返しに対応したパターンの例

マッチする文字列（「[ ]」はマッチする部分を示す）

```sql
# 03-123-4567
'[\d-]+'   # ⇒ [03-123-456]
'[\d-]+?'  # ⇒ [0][3]-[1][2][3]-[4][5][6][7]
'[\d-]++'  # ⇒ [03-123-4567]

'[\d-]*'   # ⇒ [03]-[123]-[4567]
'[\d-]*?'  # ⇒ []
'[\d-]*+'  # ⇒ [03]-[123]-[4567]

'[\d-]?'   # ⇒ [0][3]-[1][2][3]-[4][5][6][7]
'[\d-]??'  # ⇒ []
'[\d-]?+'  # ⇒ [0][3]-[1][2][3]-[4][5][6][7]
```

確認クエリ（結果は省略）
```sql
SELECT str, 
  REGEXP_EXTRACT_ALL(str,'[\d-]+')  AS m1,
  REGEXP_EXTRACT_ALL(str,'[\d-]+?') AS m1_short,
  REGEXP_EXTRACT_ALL(str,'[\d-]++') AS m1_long,
  
  REGEXP_EXTRACT_ALL(str,'[\d-]*')  AS m2,
  REGEXP_EXTRACT_ALL(str,'[\d-]*?') AS m2_short,
  REGEXP_EXTRACT_ALL(str,'[\d-]*+') AS m2_long,
  
  REGEXP_EXTRACT_ALL(str,'[\d-]?')  AS m3,
  REGEXP_EXTRACT_ALL(str,'[\d-]??') AS m3_short,
  REGEXP_EXTRACT_ALL(str,'[\d-]?+') AS m3_long
FROM (
  VALUES '03-123-4567'
) AS t(str)
```

### 境界正規表現エンジン
「境界正規表現エンジン」は，文字列の先頭や末尾などの「境界」を指示するための記号です。境界正規表現エンジンが使われたパターンでは，「〜を含む」だけでなく，「〜から始まる」や「〜で終わる」といった柔軟な記述が可能になります。
境界正規表現エンジンには多くの種類がありますが，^および$以外の使用頻度は少なめです。
|構文  |Matches|
|----|-------|
|^   |行の先頭   |
|$   |行の末尾   |
|\b  |単語境界   |
|\B  |非単語境界  |
|\A  |入力の先頭  |
|\G  |前回のマッチの末尾|
|\Z  |最後の行末記号がある場合は，それを除く入力の末尾|
|\z  |入力の末尾（行末文字があれば，それが末尾）|

#### 境界正規表現エンジンを含むパターン例
マッチする文字列（「[ ]」はマッチする部分を示す）
```sql
# abc, abcxyz, xyzabc, xabcx, abc\n
'abc'     # ⇒ [abc], [abc]xyz, xyz[abc], x[abc]x, [abc]\n
'^abc'    # ⇒ [abc], [abc]xyz,                    [abc]\n
'abc$'    # ⇒ [abc],           xyz[abc],          [abc]\n
'^abc$'   # ⇒ [abc],                              [abc]\n
'^abc\n$' # ⇒                                     [abc]\n
```

上記の例で注意してほしいのは，改行コードを含む「abc\n」という文字列に「'^abc$'」というパターンがマッチすることです。改行コードより後ろの部分は次の行のため，最初の行の末尾はcとなることから，「'^abc$'」というパターンにマッチします。

確認クエリ
```sql
SELECT str, 
  REGEXP_LIKE(str,'abc')     AS is_m1,
  REGEXP_LIKE(str,'^abc')    AS is_m2,
  REGEXP_LIKE(str,'abc$')    AS is_m3,
  REGEXP_LIKE(str,'^abc$')   AS is_m4,
  REGEXP_LIKE(str,'^abc\n$') AS is_m5
FROM (
  VALUES 'abc', 'abcxyz', 'xyzabc', 'xabcx',
         'abc' || CHR(10) --ラインフィード (LF)
) AS t(str)
```

#### よく遭遇するパターン例
最後に，よく目にするであろう正規表現のパターンを一覧にして少しだけまとめておきます。
頻出するマッチさせたい文字列
|頻出するマッチさせたい文字列|例    |正規表現                              |
|--------------|-----|----------------------------------|
|半角数値のみで構成されている， もしくは空白|123456789|^[0-9]*$                          |
|英字小文字のみで構成されている，もしくは空白|abcdefg|^[a-z]*$                          |
|英字大文字のみで構成されている，もしくは空白|ABCDEFG|^[A-Z]*$                          |
|英字小文字大文字のみで構成されている，もしくは空白|ABCdefg|^[a-zA-Z]*$                       |
|英数字のみで構成されている， もしくは空白|12aaAA|^[0-9a-zA-Z]*$                    |
|郵便番号          |123-1234|^[0-9]{3}-[0-9]{4}$               |
|日付（yyyy/M/d形式）|2009/7/29|^[0-9]{4}/[01]?[0-9]/[0123]?[0-9]$|

```sql
SELECT
 REGEXP_EXTRACT_ALL('123456789', '^[0-9]*$') AS m1,
 REGEXP_EXTRACT_ALL('abcdefg', '^[a-z]*$') AS m2,
 REGEXP_EXTRACT_ALL('ABCDEFG', '^[A-Z]*$') AS m3,
 REGEXP_EXTRACT_ALL('ABCdefg', '^[a-zA-Z]*$') AS m4,
 REGEXP_EXTRACT_ALL('12aaAA', '^[0-9a-zA-Z]*$') AS m5,
 REGEXP_EXTRACT_ALL('123-1234', '^[0-9]{3}-[0-9]{4}$') AS m6,
 REGEXP_EXTRACT_ALL('2009/7/29', '^[0-9]{4}/[01]?[0-9]/[0123]?[0-9]$') AS m7
```

## 正規表現で文字列を抽出する

正規表現で文字列を抽出したい場合にはREGEXP_EXTRACT関数を使います。

REGEXP_EXTRACTは，正規表現で記述されたパターンに対するマッチを試みて，マッチすれば最初のマッチストリングを返し，マッチしなければNULLを返す関数です。
REGEXP_EXTRACT(string, pattern)のように使い，patternに正規表現のパターンを表す文字列を指定します。返り値は，一番はじめにマッチした部分文字列です。
引数をもう1つ指定してREGEXP_EXTRACT(string, pattern, group)とした場合は，patternに正規表現のパターン文字列を，groupに何番目のグループのマッチストリングを返すかの数値（1から開始）を指定します。

REGEXP_EXTRACT(string, pattern, group)を使う場合は，正規表現を1つのグループごとに必ず「( )」で囲みます。patternの中で「( )」を1つも使っていない場合は，0 Groupsエラーが出ます。

### 特定の文字列を含み，最初にマッチした文字列を返す
REGEXP_EXTRACT(string, pattern)を使って，「td_urlが'fluentd.org'を含むかどうか」を調べるクエリを書いてみましょう。
```sql
SELECT td_url, REGEXP_EXTRACT(td_url,'fluentd.org') AS match_str
FROM sample_accesslog_fluentd
LIMIT 10
```

### いずれかの文字列を含む文字列で最初にマッチしたものを返す
正規表現では，「|」で区切って複数の文字列を並べることにより，そのいずれかにマッチするかどうかを調べることができます。これにより，URLの最後が以下のいずれかのグループにマッチするものだけを抽出するクエリを書いてみましょう。
```sql
(out_splunk$|out_file$|out_forward$|out_secure_forward$|out_exec$|out_exec_filter$|out_copy$)
```
「$」は行末を表す境界正規表現エンジンです。例えば，「out_splunk$」は「hoge/out_splunk」にはマッチしますが「out_splunk/hoge」にはマッチしません。
```sql
SELECT td_url, match_str1, match_str2
FROM
(
  SELECT td_url,
    REGEXP_EXTRACT(td_url,'out_splunk$|out_file$|out_forward$|out_secure_forward$|out_exec$|out_exec_filter$|out_copy$') AS match_str1,
    REGEXP_EXTRACT(td_url,'out_.*$') AS match_str2
  FROM sample_accesslog_fluentd
)
WHERE match_str1 IS NOT NULL OR match_str2 IS NOT NULL
LIMIT 10
```
上記のパターンは「(out_.*$)」と非常に簡単に記述できます。このパターンであれば他のoutを含む文字列を逃すこともありません。

なお，マッチしない文字列を除外したい場合にはREGEXP_LIKEという関数が使えますが，この関数については後述します。ここでは，PrestoではREGEXP_EXTRACTでマッチしなかった場合にNULLが返ることを利用して，WHERE節でNULLを除外するSELECT節により外側から包んでいます。ただし，REGEXP_EXTRACTでマッチしなかった場合の返り値はプラットフォームにより異なるので（例えばHiveでは空文字が返ります），マッチしない場合にFALSEを返す後述の判定関数REGEXP_LIKEを使うことを強くお薦めします。

### 特定の文字列を含む任意のグループを取り出す
td_urlが「docs.fluentd.org」を含むかどうかを見るクエリを書いてみましょう。LISTでなく文字列を返すように，REGEXP_EXTRACT(string, pattern, group)を使います。
まず，patternを「docs.fluentd.org」ではなく「(docs.fluentd.org)」と書きます。これにより，(docs.fluentd.org)が1つのグループとして扱われます。このグループにマッチする文字列が複数登場する場合には，group番目にマッチした部分文字列が（LISTではなく）文字列として返されます。パターンに1つも「( )」によるグループがないと，0 Groupsエラー（マッチさせられない）が出ます。
以下のクエリでは，最初（group=1）のマッチストリングを返すように記述しています。
```sql
SELECT td_url, REGEXP_EXTRACT(td_url,'(docs.fluentd.org)',1) AS match_str
FROM sample_accesslog_fluentd
LIMIT 10
```

繰り返しになりますが，マッチしなかった場合のREGEXP_EXTRACTの返り値は，PrestoではNULLですが，Hiveでは空文字です。挙動が異なるので注意しましょう。

### 複数のグループを記述する
td_urlが「docs.fluentd.org」を含み，かつその後ろに「out_file」が含まれるものを直接抽出することを考えてみましょう。
この場合，patternは「(docs.fluentd.org).*(out_file)」と書きます。グループ間の「.*」は，「その部分に何らかの文字が入る（文字がなくてもよい）」という意味です。「( )」を使わずに「docs.fluentd.org.*out_file」とした場合との違いは，「( )」を使うと1番目のグループに最初にマッチした部分文字列，2番目のグループに次にマッチした文字列を直接取り出せるのに対し，「( )」を使わないとマッチした全体しか返せないことです。
```sql
WITH 
sample AS
( 
  SELECT s FROM
  ( 
    VALUES 
    'https://docs.fluentd.org/v0.12/articles/out_file',
    'https://docs.fluentd.org/v0.12/articles/out_forward',
    'https://www.fluentd.org/v0.12/articles/out_file',
    'out_file/article/docs.fluentd.org/'
  ) AS t(s)
)
SELECT
  s, REGEXP_EXTRACT(s,'(docs.fluentd.org).*(out_file)',1) AS match_str1,
     REGEXP_EXTRACT(s,'(docs.fluentd.org).*(out_file)',2) AS match_str2
FROM sample
```

上記では以下のケースを試しています。

```sql
OK：https://docs.fluentd.org/v0.12/articles/out_file
NG：https://docs.fluentd.org/v0.12/articles/out_forward （2番目のグループを含まない）
NG：https://www.fluentd.org/v0.12/articles/out_file （1番目のグループを含まない）
NG：out_file/article/docs.fluentd.org/ （グループの出現順序が異なる）
```

なお，groupの値を1から2に変えると，マッチする場合の返り値は変わりますが，マッチしない場合の結果はNULLで変わりません。

### いずれかのグループにマッチさせる
「(docs.fluentd.org|out_file)」というパターンは，「docs.fluentd.orgまたはout_file」の意味になり，どちらか片方にマッチすればよいことになります。順番が違っていても問題ありません。両方マッチした場合の返り値は，先にマッチしたほうの値です。

```sql
WITH 
sample AS
( 
  SELECT s FROM
  ( 
    VALUES 
    'https://docs.fluentd.org/v0.12/articles/out_file',
    'https://docs.fluentd.org/v0.12/articles/out_forward',
    'https://www.fluentd.org/v0.12/articles/out_file',
    'out_file/article/docs.fluentd.org/'
  ) AS t(s)
)
SELECT
  s, REGEXP_EXTRACT(s,'(docs.fluentd.org|out_file)',1) AS match_str
FROM sample
```
### マッチした文字列をすべて取り出す
REGEXP_EXTRACT_ALLは，マッチした文字列をarrayの返り値としてすべて取り出します。この関数ではグループを意識してもあまり意味がありません。
```sql
WITH 
sample AS
( 
  SELECT s FROM
  ( 
    VALUES 
    'https://docs.fluentd.org/v0.12/articles/out_file',
    'https://docs.fluentd.org/v0.12/articles/out_forward',
    'https://www.fluentd.org/v0.12/articles/out_file',
    'out_file/article/docs.fluentd.org/'
  ) AS t(s)
)
SELECT
  s, REGEXP_EXTRACT_ALL(s,'docs.fluentd.org.*out_file') AS match_strs1,
     REGEXP_EXTRACT_ALL(s,'docs.fluentd.org|out_file') AS match_strs2
FROM sample
```

### メタ文字をエスケープしてマッチさせる
パターンを記述するために記号として使われるメタ文字```（\ * + . ? { } ( ) [ ] ^ $ - | ）```自身を文字としてマッチさせるにはどうしたらよいでしょうか？ その場合は，それぞれのメタ文字の前にエスケープ「\」を1つ置くことで，単なる文字とみなされるようにします。
複雑な例として，メタ文字のような文字列「^\\(.+*?)[|]$」をマッチさせてみましょう。ただし，以下の例はPrestoのみで実行でき，Hiveでは実行できません（Hiveでどう書けばよいかは模索中です…）。
1. 文字列全体```「^\\(.+*?)[|]$」```をマッチさせる
すべてのメタ文字にエスケープを丁寧に付けます（1つでも付け忘れると意図しない結果になります）。
```sql
SELECT 
  REGEXP_EXTRACT(
    '^\\(.+*?)[|]$',
    '\^\\\\\(\.\+\*\?\)\[\|\]\$'
)
```
2. ```「(.+*?)」```で取り出す
両端の「(」と「)」はエスケープする必要がありますが，その間に続く文字は内容を気にせず「.*」と書いてマッチさせられます。
```sql
SELECT 
  REGEXP_EXTRACT(
    '^\\(.+*?)[|]$',
    '\(.*\)'
)
```
3. ```「(.+*?)」```と```「[|]」```を，それぞれGROUP1，GROUP2としてマッチさせる
それぞれを取り出すためのパターンの外側に括弧「( )」を付けます。
```sql
SELECT 
  REGEXP_EXTRACT(
    '^\\(.+*?)[|]$',
    '(\(.*\))|(\[.*\])',
    1
  ) AS group1,
  REGEXP_EXTRACT(
    '^\\(.+*?)[|]$',
    '(\(.*\))(\[.*\])',
    2
  ) AS group2
```

## 正規表現で文字列を条件判定する
REGEXP_LIKEは，正規表現で記述されたパターンにマッチするか否かを試し，マッチすればTRUEを，マッチしなければFALSEを返す関数です。REGEXP_EXTRACT関数と違ってマッチした部分文字列は返しませんが，条件判定で正しく利用することができます。この関数は，主に特定の正規表現にマッチする文字列を含んだレコードのみを抽出したい場合に，WHERE節とともに使います。

### 特定の文字列を含むか否か
```sql
SELECT td_url
FROM sample_accesslog_fluentd
WHERE REGEXP_LIKE(td_url,'docs.fluentd.org')
LIMIT 10
```

## 正規表現で文字列を置換する
REGEXP_REPLACEは，正規表現によって記述されたパターンにマッチした部分を別の文字列で置換する関数です。マッチしなかった場合には何もせず元の文字列を返します。

### 文字列の除去

以下のクエリは，REGEXP_REPLACEを使って特定のパターンにマッチする文字列を除去する最も簡単な例です。空文字「''」を置換文字として指定しています。

```sql
SELECT td_url,
  REGEXP_REPLACE(td_url,'http(.)*://','') AS replaced_url
FROM sample_accesslog_fluentd
LIMIT 10
```
さらに，複数のパターンを「|」で列挙することで，それらを一度に変換することができます。
```sql
SELECT td_url,
  -- 'http(s)://' を除外 | '?'以降のパラメータを除外 | レコードの末尾の '/' を除外
  REGEXP_REPLACE(td_url,'http(.)*://|\?(.)*|/$','') AS replaced_url
FROM sample_accesslog_fluentd
WHERE REGEXP_LIKE(td_url,'\?')
LIMIT 10
```
なお，文字列の中でパターンが複数回マッチする場合には，そのすべての箇所が置換されることになります。下記のクエリでは，td_urlの「/」という文字がすべて「@」に置換されます。
```sql
SELECT td_url,
  REGEXP_REPLACE(td_url,'/','@') AS replaced_url
FROM sample_accesslog_fluentd
LIMIT 10
```
マッチする文字列の一部を，置換後の文字列で再利用したい場合には，「$N」（Nはグループ番号）を使います。グループの概念を使うので，パターンのうち再利用したい部分を「( )」で囲みます。
```sql
SELECT td_url,
  --'http' と 'docs.fluentd.org' を入れ替え | で区切る
  -- $1=http, $2=://, $3=docs.fluentd.org --
  REGEXP_REPLACE(td_url,'(http)(.*)(docs.fluentd.org)','$3 | $2 | $1') AS replaced_url
FROM sample_accesslog_fluentd
WHERE REGEXP_LIKE(td_url,'docs.fluentd.org')
LIMIT 10
```

## 正規表現で文字列を分割する
REGEXP_SPLITは，正規表現のパターンによって文字列を分割する関数です。

```sql
SELECT td_url,
  REGEXP_SPLIT(td_url,'/') AS splitted_strs
FROM sample_accesslog_fluentd
LIMIT 10
```
