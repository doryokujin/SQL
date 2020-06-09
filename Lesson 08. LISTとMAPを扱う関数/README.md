# Lesson 08. LISTとMAPを扱う関数

## LISTの作り方

LIST（配列）をレコードの値から生成するには，ARRAY_AGG関数を使用します。この関数は集約関数なのでGROUP BY節が必要です。

```sql
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('a',2),('a',3),('a',4),('a',5),('a',6) ) AS t(k,v)
)

SELECT k, ARRAY_AGG(v) AS list
FROM a1
GROUP BY k
```
|k                                          |list                                             |
|-------------------------------------------|-------------------------------------------------|
|a                                          |[1, 2, 3, 4, 5, 6]                               |


## 要素のNULLを除外する

LISTの要素に含まれるNULLは，多くのLIST関数の挙動を歪めてしまうため，悪であるといえます。LISTを作成したり操作したりする場合には，以下の手順でNULLを除外してから扱うようにしましょう。

### 方法1.（集約時）条件式で除外する

```sql
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',NULL),('a',1),('a',2),('a',NULL),('a',3),('a',4),('a',5) ) AS t(k,v)
),
list_table AS
(
  SELECT k, ARRAY_AGG(v) AS list
  FROM a1
  WHERE v IS NOT NULL
  GROUP BY k
)
SELECT list FROM list_table
```
|list                                       |
|-------------------------------------------|
|[1, 2, 3, 4, 5]                            |


### 方法2.（集約時）COALESCEでNULLの要素を置き換える

NULLに意味があり，他の要素に置き換え可能ならば，COALESCE関数で置き換えることができます。

```sql
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',NULL),('a',1),('a',2),('a',NULL),('a',3),('a',4),('a',5) ) AS t(k,v)
),
list_table AS
(
  SELECT k, ARRAY_AGG(COALESCE(v,0)) AS list
  FROM a1
  GROUP BY k
)

SELECT list FROM list_table
```
|list                                       |
|-------------------------------------------|
|[0, 0, 3, 4, 5, 1, 2]                      |


### 方法3. FILTER関数で除外する

既にLISTとなっているものに対しては，FILTER関数でNULLを除外できます。

```sql
SELECT FILTER( ARRAY[NULL,1,2,NULL,3,4,5], x -> x IS NOT NULL ) AS list
```
|list                                       |
|-------------------------------------------|
|[1, 2, 3, 4, 5]                            |


### （間違い）ARRAY_REMOVEを使う
ARRAY_REMOVEを使って取り除く要素としてNULLを指定しても，結果がNULLとなり正しく動作しません。
```sql
SELECT ARRAY_REMOVE(ARRAY[NULL,1,2,NULL,3,4,5],NULL) AS list
```
|list                                       |
|-------------------------------------------|
|NULL                                       |


## 1つのLISTに関する関数

### ALL_MATCH(array(T), function(T, boolean)) → boolean
LISTの各要素に対して条件式を適用していきます。すべての要素がこの条件でTRUEとなるとき，この関数はTRUEを返し，そうでない場合にはFALSEを返します。
ただし，要素に対する条件式の結果が1つ以上のNULLを返し，かつNULLの結果以外の要素の判定結果がすべてTRUEの場合には返り値がNULLになること，NULLがあってもそれ以外の要素でFALSEならばFALSEを返すことには細心の注意が必要です。

また，条件式の書き方が「x -> (xの条件式)」という特殊な書き方にであることに注意してください。

### ANY_MATCH(array(T), function(T, boolean)) → boolean
LISTの各要素に対して条件式を適用していきます。少なくとも1つの要素がこの条件でTRUEとなるとき，この関数はTRUEを返し，そうでない場合にはFALSEを返します。
ただし，要素に対する条件式の結果が1つ以上のNULLを返し，かつNULLの結果以外の要素の判定結果がすべてFALSEの場合には返り値がNULLになること，NULLがあってもそれ以外の要素でTRUEならばTRUEを返すことには細心の注意が必要です。

### NON_MATCH(array(T), function(T, boolean)) → boolean

LISTの各要素に対して条件式を適用していきます。すべての要素がこの条件でFALSEになるとき，この関数はTRUEを返し，そうでない場合にはFALSEを返します。
ただし，要素に対する条件式の結果が1つ以上のNULLを返し，かつNULLの結果以外の要素の判定結果がすべてFALSEの場合には返り値がNULLになること，NULLがあってもそれ以外の要素でTRUEならばTRUEを返すことには細心の注意が必要です。

```sql
WITH list_table AS
( SELECT ARRAY[1,2,3,4,5,6]  AS list )

SELECT 
  ALL_MATCH(  list, x -> x<10  ) AS all1, ALL_MATCH( list, x -> x%2=0 ) AS all2,
  ANY_MATCH(  list, x -> x>5   ) AS any1, ANY_MATCH( list, x -> x%2=0  ) AS any2,
  NONE_MATCH( list, x -> x>=10 ) AS none1
FROM list_table
```
|all1                                       |all2 |any1|any2|none1|
|-------------------------------------------|-----|----|----|-----|
|true                                       |false|true|true|true |


```sql
WITH list_table AS
( SELECT ARRAY[1,2,3,4,5,NULL]  AS list )

SELECT
  ALL_MATCH(  list, x -> x<10  ) AS all1, ALL_MATCH( list, x -> x%2=0 ) AS all2,
  ANY_MATCH(  list, x -> x>5   ) AS any1, ANY_MATCH( list, x -> x%2=0  ) AS any2,
  NONE_MATCH( list, x -> x>=10 ) AS none1
FROM list_table
```
|all1                                       |all2 |any1|any2|none1|
|-------------------------------------------|-----|----|----|-----|
|NULL                                       |false|NULL|true|NULL |


### ARRAY_DISTINCT(x) → array
与えられたLISTの重複する要素を取り除いてユニークな要素のLISTを返します。NULLも他の要素と同じように扱われ重複が取り除かれます。

```sql
WITH list_table AS
( SELECT ARRAY[1,1,2,2,3,NULL,NULL]  AS list )

SELECT
  list, ARRAY_DISTINCT(list) AS uniq_list
FROM list_table
```
|list                                       |uniq_list|
|-------------------------------------------|---------|
|[1, 1, 2, 2, 3, NULL, NULL]                |[1, 2, 3, NULL]|


### ARRAY_JOIN(x, delimiter, null_replacement) → varchar
LISTの要素を指定した区切り文字で結合して文字列にします。NULLの要素に対する代替を指定することができます。そのためのnull_replacementオプションを使用しないと，NULLがある場合に区切り文字が残ってしまう結果になることがあります。

```sql
WITH list_table AS
( SELECT ARRAY[NULL,1,2,2,NULL,3,NULL]  AS list )

SELECT
  ARRAY_JOIN(list,' - ') AS joined_str1, ARRAY_JOIN(list,' - ','0') AS joined_str2, ARRAY_JOIN(list,' - ','') AS joined_str3
FROM list_table
```
|joined_str1                                |joined_str2|joined_str3           |
|-------------------------------------------|-----------|----------------------|
|1 - 2 - 2 - 3 -                            |0 - 1 - 2 - 2 - 0 - 3 - 0| - 1 - 2 - 2 -  - 3 - |


### ARRAY_MAX(x) → x
LISTの要素の最大値を返します。

### ARRAY_MIN(x) → x
LISTの要素の最小値を返します。

```sql
WITH list_table AS
( SELECT ARRAY[1,1,2,2,3,4,5]  AS list )

SELECT
  ARRAY_MAX(list) AS elm_max, ARRAY_MIN(list) AS elm_min
FROM list_table
```
|elm_max                                    |elm_min|
|-------------------------------------------|-------|
|5                                          |1      |


ARRAY_MAXやARRAY_MIN関数は，LISTの要素にNULLが含まれると台無しになってしまいます。

```sql
WITH list_table AS
( SELECT ARRAY[NULL,1,2,2,3,4,5]  AS list )

SELECT
  ARRAY_MAX(list) AS elm_max, ARRAY_MIN(list) AS elm_min
FROM list_table
```
|elm_max                                    |elm_min|
|-------------------------------------------|-------|
|NULL                                       |NULL   |


### ARRAY_REMOVE(x, element) → array

特定の要素を除外します。除外要素としてNULLを指定するとNULLを返すので注意してください。条件式によって一度に複数の要素を除外したい場合にはFILTER関数を使います。

```sql
SELECT ARRAY_REMOVE( ARRAY[NULL,1,2,NULL,3,4,5], 1 ) AS list
```
|list                                       |
|-------------------------------------------|
|[NULL, 2, NULL, 3, 4, 5]                   |


### CONTAINS(x, element) → boolean
第2引数に指定した要素がLISTに含まれるか否かを判定します。NULLを含むかどうかの判定には使えません。

```sql
WITH list_table AS
( SELECT ARRAY[NULL,1,5,NULL,4,3,2] AS list )

SELECT CONTAINS(list,1), CONTAINS(list,NULL)
FROM list_table
```
|_col0                                      |_col1|
|-------------------------------------------|-----|
|true                                       |NULL |


### ARRAY_SORT(x) → array

LISTの要素を並び替えます。NULLは最後に配置されます。

```sql
SELECT ARRAY_SORT(ARRAY[NULL,1,5,NULL,4,3,2]) AS list
```
|list                                       |
|-------------------------------------------|
|[1, 2, 3, 4, 5, NULL, NULL]                |


この関数には，Comparatorによるマニュアルな並び替えルールを適用できますが，ここでは割愛します。

### CARDINALITY(x) → bigint
LISTの要素数を返します。NULLの要素も数えられます。

```sql
SELECT CARDINALITY(ARRAY[NULL,1,5,NULL,4,3,2]) AS size
```
|size                                       |
|-------------------------------------------|
|7                                          |


### FILTER(array(T), function(T, boolean)) -> array(T)

条件式をフィルターとしてLISTの要素を絞ります。

```sql
WITH list_table AS
( SELECT ARRAY[NULL,1,5,NULL,4,3,2] AS list )

SELECT FILTER(list, x-> x IS NOT NULL) AS fil1, 
       FILTER(list, x-> x%2=1) AS fil2
FROM list_table
```
|fil1                                       |fil2     |
|-------------------------------------------|---------|
|[1, 5, 4, 3, 2]                            |[1, 5, 3]|


### FLATTEN(x) → array

LISTの要素を平坦化します。

```sql
SELECT FLATTEN(ARRAY[ ARRAY[1,2], ARRAY[3,4], ARRAY[5,6,7]])
FROM ( VALUES 1 ) AS t(n)
```
|_col0                                      |
|-------------------------------------------|
|[1, 2, 3, 4, 5, 6, 7]                      |


### REVERSE(x) → array
LISTの要素の順番を反転します。

### SHUFFLE(x) → array
LISTの要素の順番をシャッフルします。

 ### SLICE(x, start, length) → array
LISTをスライスします。指定する開始インデックスは1からです。-1などのマイナスの値を指定した場合は後ろから数えた位置になり，そこから前へlength分だけスライスします。

```sql
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT SLICE( list, 1, 4 )  AS slice1,
       SLICE( list, -3, 3 ) AS slice2
FROM list_table
```
|slice1                                     |slice2               |
|-------------------------------------------|---------------------|
|[1, 2, 3, NULL]                            |[4, 5, 6]            |


### TRANSFORM(array(T), function(T, U)) -> array(U)

LISTの各要素に対し，指定したfunctionを適用した新しいLISTを作成します。ただし，多くの場合にはNULLの要素の結果はNULLのままとなります。functionにはラムダ式で記述します。

```sql
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT TRANSFORM( list, x -> x+1 ) AS t1,
       TRANSFORM( list, x -> COALESCE(x,0)+1 ) AS t2,
       TRANSFORM( list, x -> 'elm_' || CAST(x AS VARCHAR) ) AS t3
FROM list_table
```
|t1                                         |t2                   |t3                                                         |
|-------------------------------------------|---------------------|-----------------------------------------------------------|
|[2, 3, 4, NULL, 5, 6, 7]                   |[2, 3, 4, 1, 5, 6, 7]|["elm_1", "elm_2", "elm_3", NULL, "elm_4", "elm_5", "elm_6"]|


### REDUCE(array(T), initialState S, inputFunction(S, T, S), outputFunction(S, R)) → R
REDUCE関数は，要素を何らかのfunctionによって集約した1つの値（LISTではない数値や文字列など）を返す関数です。要素同士のfunctionの記述方法はラムダ式となります。また，初期値を指定する必要もあります。単純な算術平均でも，REDUCEで求めようとすると，それなりに複雑な記述になります。以下に，NULLを0に変換してから算術平均を求めるクエリを示します。
```sql
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT REDUCE( list, 0, (s,x)->s+x, s->s ) AS t1,
       REDUCE( list, 0, (s,x)->s+COALESCE(x,0), s->s ) AS t2,
       REDUCE( list, 
               CAST(ROW(0.0, 0) AS ROW(sum DOUBLE, count INTEGER)),
               (s, x) -> CAST(ROW(COALESCE(x,0) + s.sum, s.count + 1) AS ROW(sum DOUBLE, count INTEGER)),
               s -> IF(s.count = 0, NULL, s.sum / s.count)) AS t3
FROM list_table
```
|t1                                         |t2                   |t3 |
|-------------------------------------------|---------------------|---|
|NULL                                       |21                   |3.0|


ちなみに，下記のようにLISTを行に展開したうえで普通の集約関数を使うほうが同様のことをわかりやすく記述できるのでおすすめです。

```sql
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT AVG(COALESCE(x,0))
FROM list_table
CROSS JOIN UNNEST(list) AS t(x)
```
|_col0                                      |
|-------------------------------------------|
|3.0                                        |


## 2つ以上のLISTに関する関数

### ARRAY_EXCEPT(x, y) → array
LIST xの要素から，LIST yに含まれている要素を除外します。さらに，残った要素は重複が取り除かれます。

```sql
SELECT ARRAY_EXCEPT(
  ARRAY [1,1,2,2,3,4,5,6],
  ARRAY [1,3,5,7]
)
```
|_col0                                      |
|-------------------------------------------|
|[2, 4, 6]                                  |


### ARRAY_INTERSECT(x, y) → array
LIST xの要素とLIST yの要素の共通部分を取り出します。さらに，選ばれた要素は重複が取り除かれます。

```sql
SELECT ARRAY_INTERSECT(
  ARRAY [1,1,2,2,3,4,5,6],
  ARRAY [1,3,5,7]
)
```
|_col0                                      |
|-------------------------------------------|
|[1, 3, 5]                                  |


### ARRAY_UNION(x, y) → array
LIST xかLIST yのいずれかに含まれる要素を取り出します。さらに，選ばれた要素は重複が取り除かれます。
```sql
SELECT ARRAY_UNION(
  ARRAY [1,1,2,2,3,4,5,6],
  ARRAY [1,3,5,7]
)
```
|_col0                                      |
|-------------------------------------------|
|[1, 2, 3, 4, 5, 6, 7]                      |


### ARRAYS_OVERLAP(x, y) → boolean
LIST xとLIST yにNULL以外の共通要素があるか判定します。1つでも共通部分があればTRUE，そうでなければFALSEを返しますが，共通部分がなく片方にNULLが含まれるならNULLを返してしまうことに注意が必要です。
```sql
SELECT 
  ARRAYS_OVERLAP(
    ARRAY [1,2,3],
    ARRAY [2,4,6]
  ) AS ol1,
  ARRAYS_OVERLAP(
    ARRAY [1,2,3],
    ARRAY [4,5,6]
  ) AS ol2,
  ARRAYS_OVERLAP(
    ARRAY [NULL,1,2,3],
    ARRAY [2,4,6]
  ) AS ol3,
  ARRAYS_OVERLAP(
    ARRAY [NULL,1,2,3],
    ARRAY [4,5,6]
  ) AS ol4,
  ARRAYS_OVERLAP(
    ARRAY [NULL,1,2,3],
    ARRAY [NULL,4,5,6]
  ) AS ol5
```
|ol1                                        |ol2  |ol3 |ol4 |ol5 |
|-------------------------------------------|-----|----|----|----|
|true                                       |false|true|NULL|NULL|


### CONCAT(array1, array2, ..., arrayN) → array
複数のLISTを連結して，（入れ子でない）LISTを返します。この関数は「||」でも記述可能です。要素の重複などは考慮されず，単純に左側に指定したLISTの要素から順に並べられます。

```sql
SELECT 
  ARRAY[1,2,3]||ARRAY[2,3,4]||ARRAY[NULL,0],
  CONCAT(ARRAY[1,2,3],ARRAY[2,3,4],ARRAY[NULL,0])
```
|_col0                                      |_col1|
|-------------------------------------------|-----|
|[1, 2, 3, 2, 3, 4, NULL, 0]                |[1, 2, 3, 2, 3, 4, NULL, 0]|


### ZIP(array1, array2[, ...]) -> array(row)
複数のLISTの同じ位置の要素をペアリングし，行にして返します。要素が足りない場合はNULLが補完されます。
```sql
SELECT 
  ZIP(
    ARRAY['A','B','C'],
    ARRAY['a','b','c'],
    ARRAY[1,2]
  )
```
|_col0                                      |
|-------------------------------------------|
|[["A", "a", 1], ["B", "b", 2], ["C", "c", NULL]]|


ZIP関数の結果はUNNESTを使うことによってきれいに行と列に展開できます。

```sql
WITH zip_table AS
(  SELECT ZIP( ARRAY['A','B','C'], ARRAY['a','b','c'], ARRAY[1,2] ) AS z )

SELECT c1, c2, c3
FROM zip_table
CROSS JOIN UNNEST(z) AS t(c1,c2,c3)
```
|c1                                         |c2 |c3 |
|-------------------------------------------|---|---|
|A                                          |a  |1  |
|B                                          |b  |2  |
|C                                          |c  |NULL|


### ZIP_WITH(array(T), array(U), function(T, U, R)) -> array(R)

ZIPのように2つのLISTの同じ位置をペアリングしますが，その際にfunctionを実行することができます。例えば，要素をマージするファンクションを使うことで，ペアでなく1つの値としてLISTに格納することができます。要素が足りずNULLで補完されたペアについては，大多数のファンクションでNULLが返ります。

```sql
SELECT 
  ZIP_WITH( ARRAY['A','B','C'], ARRAY['a','b','c'], (x,y)->(y,x) ),
  ZIP_WITH( ARRAY['A','B','C'], ARRAY['a','b'], (x,y)->x||y ),
  ZIP_WITH( ARRAY[1,2,3], ARRAY[3,2], (x,y)->x+y )
```
|_col0                                      |_col1|_col2|
|-------------------------------------------|-----|-----|
|[["a", "A"], ["b", "B"], ["c", "C"]]       |["Aa", "Bb", NULL]|[4, 4, NULL]|


上記の1つ目の結果をUNNESTでテーブルに展開してみましょう。

```sql
WITH zip_table AS
(
  SELECT ZIP_WITH( ARRAY['A','B','C'], ARRAY['a','b','c'], (x,y)->(y,x) ) AS z
)
SELECT c1,c2
FROM zip_table
CROSS JOIN UNNEST(z) AS t(c1,c2)
```
|c1                                         |c2 |
|-------------------------------------------|---|
|a                                          |A  |
|b                                          |B  |
|c                                          |C  |


## MAPの作り方
MAPはkeyとvalueのペアが1つの要素として格納されるLISTと似たものですが，インデックスで要素を取り出していたLISTに対し，MAPではkeyを指定して要素のvalueを取り出します。
MAPをレコードの値から生成するには，MAP_AGG関数を使用します。この関数は集約関数なので，場合によってはGROUP BY節が必要です。

```sql
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('b',2),('c',NULL),('d',4),('e',5),('f',NULL) ) AS t(k,v)
)

SELECT MAP_AGG(k,v) AS mp
FROM a1
```
|mp                                         |
|-------------------------------------------|
|{"a"=>1, "b"=>2, "c"=>NULL, "d"=>4, "e"=>5, "f"=>NULL}|


MAPではkeyの重複が許されないので，元のレコードでkeyが重複している場合は注意してください。また，NULLのkeyは後で取り出せないので存在できません。

```sql
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('b',2),('c',3),('a',4),('b',5),(NULL,6) ) AS t(k,v)
)

SELECT MAP_AGG(k,v) AS mp
FROM a1
```
|mp                                         |
|-------------------------------------------|
|{"a"=>1, "b"=>2, "c"=>3}                   |


## 要素のNULLを除外する
MAPの要素におけるNULLは，多くのMAP関数の挙動を歪めてしまうため，悪といえます。MAPを作成したり操作したりする場合には，NULLを除外した状態で扱うようにしましょう。

## 方法1.（集約時）条件式で除外する

```sql
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('b',2),('c',NULL),('d',4),('e',5),('f',NULL) ) AS t(k,v)
),
map_table AS
(
  SELECT MAP_AGG(k,v) AS mp
  FROM a1
  WHERE v IS NOT NULL
)
SELECT mp FROM map_table
```
|mp                                         |
|-------------------------------------------|
|{"a"=>1, "b"=>2, "d"=>4, "e"=>5}           |


## 方法2.（集約時）COALESCE関数でNULLの要素を置き換える
NULLに意味があり，他の要素に置き換え可能ならば，COALESCE関数で置き換えるとよいでしょう。
```sql
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('b',2),('c',NULL),('d',4),('e',5),('f',NULL) ) AS t(k,v)
),
map_table AS
(
  SELECT MAP_AGG(k,COALESCE(v,0)) AS mp
  FROM a1
)
SELECT mp FROM map_table
```
|mp                                         |
|-------------------------------------------|
|{"a"=>1, "b"=>2, "c"=>0, "d"=>4, "e"=>5, "f"=>0}|


## 方法3. MAP_FILTER関数で除外する
既にMAPとなっている場合は，MAP_FILTER関数でNULLを除外できます。
```sql
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d','e','f'], ARRAY[1,2,NULL,4,5,NULL]) AS mp )

SELECT MAP_FILTER( mp, (k,v) -> v IS NOT NULL ) AS list
FROM map_table
```
|list                                       |
|-------------------------------------------|
|{"a"=>1, "b"=>2, "d"=>4, "e"=>5}           |


## MAPに関する関数

### CARDINALITY(x) → bigint
MAP xのサイズを返します。

### ELEMENT_AT(map(K, V), key) → V

MAPから指定したkeyを持つvalueを探し，見つかればそのvalueを，見つからなければNULLを返します。valueが見つからない場合と，keyのvalueがNULLである場合との区別ができないので，注意してください。

```sql
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d','e','f'], ARRAY[1,2,NULL,4,5,NULL]) AS mp )

SELECT ELEMENT_AT( mp, 'a' ), ELEMENT_AT( mp, 'c' ), ELEMENT_AT( mp, 'g' )
FROM map_table
```
|_col0                                      |_col1|_col2|
|-------------------------------------------|-----|-----|
|1                                          |NULL |NULL |


### MAP(array(K), array(V)) -> map(K, V)

2つのLISTの同じ位置の要素をペアリングしてMAPを作成します。双方の長さが一致しなければエラーを返します。

### MAP_FROM_ENTRIES(array(row(K, V))) -> map(K, V)

keyとvalueのペアを並べたLISTをMAPに変換します。
```sql
SELECT MAP_FROM_ENTRIES( ARRAY[('a',1),('b',2),('c',NULL),('d',4),('e',5),('f',NULL)] )
```
|_col0                                      |
|-------------------------------------------|
|{"a"=>1, "b"=>2, "c"=>NULL, "d"=>4, "e"=>5, "f"=>NULL}|


### MAP_ENTRIES(map(K, V)) -> array(row(K, V))
MAPを与えると，それをkeyとvalueのペアからなるLISTに変換します。
```sql
SELECT MAP_ENTRIES( MAP(ARRAY['a','b','c','d','e','f'], ARRAY[1,2,NULL,4,5,NULL]) )
```
|_col0                                      |
|-------------------------------------------|
|[["a", 1], ["b", 2], ["c", NULL], ["d", 4], ["e", 5], ["f", NULL]]|


### MAP_CONCAT(map1(K, V), map2(K, V), ..., mapN(K, V)) -> map(K, V)
複数のMAPを連結します。MAPの連結には「||」が使えないので注意してください。keyが重複する場合には，いずれか1つに書き換えられます。
```sql
WITH map_table AS
(
  SELECT
    MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp1,
    MAP(ARRAY['c','d','e','f'], ARRAY[5,6,7,8]) AS mp2
)

SELECT MAP_CONCAT(mp1,mp2)
FROM map_table
```
|_col0                                      |
|-------------------------------------------|
|{"a"=>1, "b"=>2, "c"=>5, "d"=>6, "e"=>7, "f"=>8}|


### MAP_FILTER(map(K, V), function(K, V, boolean)) -> map(K, V)

```sql
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,2,3,4]) AS mp )

SELECT MAP_FILTER( mp, (k,v)-> v>=3 )
FROM map_table
```
|_col0                                      |
|-------------------------------------------|
|{"c"=>3, "d"=>4}                           |


### MAP_KEYS(x(K, V)), MAP_VALUES(x(K, V)) -> array(V)
MAPのkeyまたはvalueだけをLISTとして返します。

```sql
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,NULL,3,NULL]) AS mp )

SELECT MAP_KEYS(mp) AS keys, MAP_VALUES(mp) AS vals
FROM map_table
```
|keys                                       |vals            |
|-------------------------------------------|----------------|
|["a", "b", "c", "d"]                       |[1, NULL, 3, NULL]|


### MAP_ZIP_WITH(map(K, V1), map(K, V2), function(K, V1, V2, V3)) -> map(K, V3)

2つのMAPの同じkeyのvalueに対して何らかのfunctionを適用し，1つのvalueにしてkeyに格納し直します。
片方にしかないkeyについては，もう片方の対応するvalueはNULLとして扱われます。
そして，片方がNULLのvalueである場合，大多数のfunctionではNULLが返ります。

```sql
WITH map_table AS
(
  SELECT
    MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp1,
    MAP(ARRAY['c','d','e','f'], ARRAY[5,6,7,NULL]) AS mp2
)

SELECT MAP_ZIP_WITH(mp1,mp2,(k,v1,v2)->v1+v2),
  MAP_ZIP_WITH(mp1,mp2,(k,v1,v2)->COALESCE(v1,0)+COALESCE(v2,0))
FROM map_table
```
|_col0                                      |_col1           |
|-------------------------------------------|----------------|
|{"a"=>NULL, "b"=>NULL, "c"=>NULL, "d"=>10, "e"=>NULL, "f"=>NULL}|{"a"=>1, "b"=>2, "c"=>5, "d"=>10, "e"=>7, "f"=>0}|


### TRANSFORM_KEYS(map(K1, V), function(K1, V, K2)) -> map(K2, V)
key自身にfunctionを適用して新しいkeyとし，それを使ってMAPを再構築します。新しいkeyがNULLになる場合はエラーとなります。

```sql
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp )

SELECT TRANSFORM_KEYS(mp,(k,v)->COALESCE(v,0)),
       TRANSFORM_KEYS(mp,(k,v)->k||CAST(COALESCE(v,0) AS VARCHAR))
FROM map_table
```
|_col0                                      |_col1           |
|-------------------------------------------|----------------|
|{"0"=>NULL, "1"=>1, "2"=>2, "4"=>4}        |{"a1"=>1, "b2"=>2, "d4"=>4, "c0"=>NULL}|


### TRANSFORM_VALUES(map(K, V1), function(K, V1, V2)) -> map(K, V2)
MAPのvalueのみにfunctionを適用し，新しいvalueでMAPを再構築します。新しいvalueはNULLになってもかまいません。

```sql
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp )

SELECT TRANSFORM_VALUES(mp,(k,v)->v),
       TRANSFORM_VALUES(mp,(k,v)->k||CAST(v AS VARCHAR))
FROM map_table
```
|_col0                                      |_col1           |
|-------------------------------------------|----------------|
|{"a"=>1, "b"=>2, "c"=>NULL, "d"=>4}        |{"a"=>"a1", "b"=>"b2", "c"=>NULL, "d"=>"d4"}|

