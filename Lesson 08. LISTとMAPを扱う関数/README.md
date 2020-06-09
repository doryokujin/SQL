# Lesson 08. LISTとMAPを扱う関数

LISTの作り方
LIST（配列）をレコードの値から生成するには，ARRAY_AGG関数を使用します。この関数は集約関数なのでGROUP BY節が必要です。
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('a',2),('a',3),('a',4),('a',5),('a',6) ) AS t(k,v)
)

SELECT k, ARRAY_AGG(v) AS list
FROM a1
GROUP BY k

要素のNULLを除外する
LISTの要素に含まれるNULLは，多くのLIST関数の挙動を歪めてしまうため，悪であるといえます。LISTを作成したり操作したりする場合には，以下の手順でNULLを除外してから扱うようにしましょう。
方法1.（集約時）条件式で除外する
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

方法2.（集約時）COALESCEでNULLの要素を置き換える
NULLに意味があり，他の要素に置き換え可能ならば，COALESCE関数で置き換えることができます。
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

方法3. FILTER関数で除外する
既にLISTとなっているものに対しては，FILTER関数でNULLを除外できます。
SELECT FILTER( ARRAY[NULL,1,2,NULL,3,4,5], x -> x IS NOT NULL ) AS list

（間違い）ARRAY_REMOVEを使う
SELECT FILTER( ARRAY[NULL,1,2,NULL,3,4,5], x -> x IS NOT NULL ) AS list
FROM list_table

1つのLISTに関する関数
ALL_MATCH(array(T), function(T, boolean)) → boolean
LISTの各要素に対して条件式を適用していきます。すべての要素がこの条件でTRUEとなるとき，この関数はTRUEを返し，そうでない場合にはFALSEを返します。
ただし，要素に対する条件式の結果が1つ以上のNULLを返し，かつNULLの結果以外の要素の判定結果がすべてTRUEの場合には返り値がNULLになること，NULLがあってもそれ以外の要素でFALSEならばFALSEを返すことには細心の注意が必要です。
また，条件式の書き方が「x -> (xの条件式)」という特殊な書き方にであることに注意してください。
ANY_MATCH(array(T), function(T, boolean)) → boolean
LISTの各要素に対して条件式を適用していきます。少なくとも1つの要素がこの条件でTRUEとなるとき，この関数はTRUEを返し，そうでない場合にはFALSEを返します。
ただし，要素に対する条件式の結果が1つ以上のNULLを返し，かつNULLの結果以外の要素の判定結果がすべてFALSEの場合には返り値がNULLになること，NULLがあってもそれ以外の要素でTRUEならばTRUEを返すことには細心の注意が必要です。
NON_MATCH(array(T), function(T, boolean)) → boolean
LISTの各要素に対して条件式を適用していきます。すべての要素がこの条件でFALSEになるとき，この関数はTRUEを返し，そうでない場合にはFALSEを返します。
ただし，要素に対する条件式の結果が1つ以上のNULLを返し，かつNULLの結果以外の要素の判定結果がすべてFALSEの場合には返り値がNULLになること，NULLがあってもそれ以外の要素でTRUEならばTRUEを返すことには細心の注意が必要です。
WITH list_table AS
( SELECT ARRAY[1,2,3,4,5,6]  AS list )

SELECT 
  ALL_MATCH(  list, x -> x<10  ) AS all1, ALL_MATCH( list, x -> x%2=0 ) AS all2,
  ANY_MATCH(  list, x -> x>5   ) AS any1, ANY_MATCH( list, x -> x%2=0  ) AS any2,
  NONE_MATCH( list, x -> x>=10 ) AS none1
FROM list_table

WITH list_table AS
( SELECT ARRAY[1,2,3,4,5,NULL]  AS list )

SELECT
  ALL_MATCH(  list, x -> x<10  ) AS all1, ALL_MATCH( list, x -> x%2=0 ) AS all2,
  ANY_MATCH(  list, x -> x>5   ) AS any1, ANY_MATCH( list, x -> x%2=0  ) AS any2,
  NONE_MATCH( list, x -> x>=10 ) AS none1
FROM list_table

ARRAY_DISTINCT(x) → array
与えられたLISTの重複する要素を取り除いてユニークな要素のLISTを返します。NULLも他の要素と同じように扱われ重複が取り除かれます。
WITH list_table AS
( SELECT ARRAY[1,1,2,2,3,NULL,NULL]  AS list )

SELECT
  list, ARRAY_DISTINCT(list) AS uniq_list
FROM list_table

ARRAY_JOIN(x, delimiter, null_replacement) → varchar
LISTの要素を指定した区切り文字で結合して文字列にします。NULLの要素に対する代替を指定することができます。そのためのnull_replacementオプションを使用しないと，NULLがある場合に区切り文字が残ってしまう結果になることがあります。
WITH list_table AS
( SELECT ARRAY[NULL,1,2,2,NULL,3,NULL]  AS list )

SELECT
  ARRAY_JOIN(list,' - ') AS joined_str1, ARRAY_JOIN(list,' - ','0') AS joined_str2, ARRAY_JOIN(list,' - ','') AS joined_str3
FROM list_table

ARRAY_MAX(x) → x
LISTの要素の最大値を返します。
ARRAY_MIN(x) → x
LISTの要素の最小値を返します。
WITH list_table AS
( SELECT ARRAY[1,1,2,2,3,4,5]  AS list )

SELECT
  ARRAY_MAX(list) AS elm_max, ARRAY_MIN(list) AS elm_min
FROM list_table

ARRAY_MAXやARRAY_MIN関数は，LISTの要素にNULLが含まれると台無しになってしまいます。
WITH list_table AS
( SELECT ARRAY[NULL,1,2,2,3,4,5]  AS list )

SELECT
  ARRAY_MAX(list) AS elm_max, ARRAY_MIN(list) AS elm_min
FROM list_table

ARRAY_REMOVE(x, element) → array
特定の要素を除外します。除外要素としてNULLを指定するとNULLを返すので注意してください。条件式によって一度に複数の要素を除外したい場合にはFILTER関数を使います。
SELECT ARRAY_REMOVE( ARRAY[NULL,1,2,NULL,3,4,5], 1 ) AS list

CONTAINS(x, element) → boolean
第2引数に指定した要素がLISTに含まれるか否かを判定します。NULLを含むかどうかの判定には使えません。
WITH list_table AS
( SELECT ARRAY[NULL,1,5,NULL,4,3,2] AS list )

SELECT CONTAINS(list,1), CONTAINS(list,NULL)
FROM list_table

ARRAY_SORT(x) → array
LISTの要素を並び替えます。NULLは最後に配置されます。
SELECT ARRAY_SORT(ARRAY[NULL,1,5,NULL,4,3,2]) AS list

この関数には，Comparatorによるマニュアルな並び替えルールを適用できますが，ここでは割愛します。
CARDINALITY(x) → bigint
LISTの要素数を返します。NULLの要素も数えられます。
SELECT CARDINALITY(ARRAY[NULL,1,5,NULL,4,3,2]) AS size

FILTER(array(T), function(T, boolean)) -> array(T)
条件式をフィルターとしてLISTの要素を絞ります。
WITH list_table AS
( SELECT ARRAY[NULL,1,5,NULL,4,3,2] AS list )

SELECT FILTER(list, x-> x IS NOT NULL) AS fil1, 
       FILTER(list, x-> x%2=1) AS fil2
FROM list_table

FLATTEN(x) → array
LISTの要素を平坦化します。
SELECT FLATTEN(ARRAY[ ARRAY[1,2], ARRAY[3,4], ARRAY[5,6,7]])
FROM ( VALUES 1 ) AS t(n)

REVERSE(x) → array
LISTの要素の順番を反転します。
SHUFFLE(x) → array
LISTの要素の順番をシャッフルします。
SLICE(x, start, length) → array
LISTをスライスします。指定する開始インデックスは1からです。-1などのマイナスの値を指定した場合は後ろから数えた位置になり，そこから前へlength分だけスライスします。
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT TRANSFORM( list, x -> x+1 ) AS t1,
       TRANSFORM( list, x -> COALESCE(x,0)+1 ) AS t2,
       TRANSFORM( list, x -> 'elm_' || CAST(x AS VARCHAR) ) AS t3
FROM list_table

TRANSFORM(array(T), function(T, U)) -> array(U)
LISTの各要素に対し，指定したfunctionを適用した新しいLISTを作成します。ただし，多くの場合にはNULLの要素の結果はNULLのままとなります。functionにはラムダ式で記述します。
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT TRANSFORM( list, x -> x+1 ) AS t1,
       TRANSFORM( list, x -> COALESCE(x,0)+1 ) AS t2,
       TRANSFORM( list, x -> 'elm_' || CAST(x AS VARCHAR) ) AS t3
FROM list_table

REDUCE(array(T), initialState S, inputFunction(S, T, S), outputFunction(S, R)) → R
REDUCE関数は，要素を何らかのfunctionによって集約した1つの値（LISTではない数値や文字列など）を返す関数です。要素同士のfunctionの記述方法はラムダ式となります。また，初期値を指定する必要もあります。単純な算術平均でも，REDUCEで求めようとすると，それなりに複雑な記述になります。以下に，NULLを0に変換してから算術平均を求めるクエリを示します。
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT REDUCE( list, 0, (s,x)->s+x, s->s ) AS t1,
       REDUCE( list, 0, (s,x)->s+COALESCE(x,0), s->s ) AS t2,
       REDUCE( list, 
               CAST(ROW(0.0, 0) AS ROW(sum DOUBLE, count INTEGER)),
               (s, x) -> CAST(ROW(COALESCE(x,0) + s.sum, s.count + 1) AS ROW(sum DOUBLE, count INTEGER)),
               s -> IF(s.count = 0, NULL, s.sum / s.count)) AS t3
FROM list_table

ちなみに，下記のようにLISTを行に展開したうえで普通の集約関数を使うほうが同様のことをわかりやすく記述できるのでおすすです。
WITH list_table AS
( SELECT ARRAY[1,2,3,NULL,4,5,6] AS list )

SELECT AVG(COALESCE(x,0))
FROM list_table
CROSS JOIN UNNEST(list) AS t(x)

2つ以上のLISTに関する関数
ARRAY_EXCEPT(x, y) → array
LIST xの要素から，LIST yに含まれている要素を除外します。さらに，残った要素は重複が取り除かれます。
SELECT ARRAY_EXCEPT(
  ARRAY [1,1,2,2,3,4,5,6],
  ARRAY [1,3,5,7]
)

ARRAY_INTERSECT(x, y) → array
LIST xの要素とLIST yの要素の共通部分を取り出します。さらに，選ばれた要素は重複が取り除かれます。
SELECT ARRAY_INTERSECT(
  ARRAY [1,1,2,2,3,4,5,6],
  ARRAY [1,3,5,7]
)

ARRAY_UNION(x, y) → array
LIST xかLIST yのいずれかに含まれる要素を取り出します。さらに，選ばれた要素は重複が取り除かれます。
SELECT ARRAY_UNION(
  ARRAY [1,1,2,2,3,4,5,6],
  ARRAY [1,3,5,7]
)

ARRAYS_OVERLAP(x, y) → boolean
LIST xとLIST yにNULL以外の共通要素があるか判定します。1つでも共通部分があればTRUE，そうでなければFALSEを返しますが，共通部分がなく片方にNULLが含まれるならNULLを返してしまうことに注意が必要です。
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

CONCAT(array1, array2, ..., arrayN) → array
複数のLISTを連結して，（入れ子でない）LISTを返します。この関数は「||」でも記述可能です。要素の重複などは考慮されず，単純に左側に指定したLISTの要素から順に並べられます。
SELECT 
  ARRAY[1,2,3]||ARRAY[2,3,4]||ARRAY[NULL,0],
  CONCAT(ARRAY[1,2,3],ARRAY[2,3,4],ARRAY[NULL,0])

ZIP(array1, array2[, ...]) -> array(row)
複数のLISTの同じ位置の要素をペアリングし，行にして返します。要素が足りない場合はNULLが補完されます。
SELECT 
  ZIP(
    ARRAY['A','B','C'],
    ARRAY['a','b','c'],
    ARRAY[1,2]
  )

ZIP関数の結果はUNNESTを使うことによってきれいに行と列に展開できます。
WITH zip_table AS
(  SELECT ZIP( ARRAY['A','B','C'], ARRAY['a','b','c'], ARRAY[1,2] ) AS z )

SELECT c1, c2, c3
FROM zip_table
CROSS JOIN UNNEST(z) AS t(c1,c2,c3)

ZIP_WITH(array(T), array(U), function(T, U, R)) -> array(R)
ZIPのように2つのLISTの同じ位置をペアリングしますが，その際にfunctionを実行することができます。例えば，要素をマージするファンクションを使うことで，ペアでなく1つの値としてLISTに格納することができます。要素が足りずNULLで補完されたペアについては，大多数のファンクションでNULLが返ります。
SELECT 
  ZIP_WITH( ARRAY['A','B','C'], ARRAY['a','b','c'], (x,y)->(y,x) ),
  ZIP_WITH( ARRAY['A','B','C'], ARRAY['a','b'], (x,y)->x||y ),
  ZIP_WITH( ARRAY[1,2,3], ARRAY[3,2], (x,y)->x+y )
上記の1つ目の結果をUNNESTでテーブルに展開してみましょう。
WITH zip_table AS
(
  SELECT ZIP_WITH( ARRAY['A','B','C'], ARRAY['a','b','c'], (x,y)->(y,x) ) AS z
)
SELECT c1,c2
FROM zip_table
CROSS JOIN UNNEST(z) AS t(c1,c2)

MAPの作り方
MAPはkeyとvalueのペアが1つの要素として格納されるLISTと似たものですが，インデックスで要素を取り出していたLISTに対し，MAPではkeyを指定して要素のvalueを取り出します。
MAPをレコードの値から生成するには，MAP_AGG関数を使用します。この関数は集約関数なので，場合によってはGROUP BY節が必要です。
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('b',2),('c',NULL),('d',4),('e',5),('f',NULL) ) AS t(k,v)
)

SELECT MAP_AGG(k,v) AS mp
FROM a1

MAPではkeyの重複が許されないので，元のレコードでkeyが重複している場合は注意してください。また，NULLのkeyは後で取り出せないので存在できません。
WITH a1 AS
(
  SELECT k,v
  FROM ( VALUES ('a',1),('b',2),('c',3),('a',4),('b',5),(NULL,6) ) AS t(k,v)
)

SELECT MAP_AGG(k,v) AS mp
FROM a1

要素のNULLを除外する
MAPの要素におけるNULLは，多くのMAP関数の挙動を歪めてしまうため，悪といえます。MAPを作成したり操作したりする場合には，NULLを除外した状態で扱うようにしましょう。
方法1.（集約時）条件式で除外する
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

方法2.（集約時）COALESCE関数でNULLの要素を置き換える
NULLに意味があり，他の要素に置き換え可能ならば，COALESCE関数で置き換えるとよいでしょう。
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

方法3. MAP_FILTER関数で除外する
既にMAPとなっている場合は，MAP_FILTER関数でNULLを除外できます。
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d','e','f'], ARRAY[1,2,NULL,4,5,NULL]) AS mp )

SELECT MAP_FILTER( mp, (k,v) -> v IS NOT NULL ) AS list
FROM map_table

MAPに関する関数
CARDINALITY(x) → bigint
MAP xのサイズを返します。
ELEMENT_AT(map(K, V), key) → V
MAPから指定したkeyを持つvalueを探し，見つかればそのvalueを，見つからなければNULLを返します。valueが見つからない場合と，keyのvalueがNULLである場合との区別ができないので，注意してください。
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d','e','f'], ARRAY[1,2,NULL,4,5,NULL]) AS mp )

SELECT ELEMENT_AT( mp, 'a' ), ELEMENT_AT( mp, 'c' ), ELEMENT_AT( mp, 'g' )
FROM map_table

MAP(array(K), array(V)) -> map(K, V)
2つのLISTの同じ位置の要素をペアリングしてMAPを作成します。双方の長さが一致しなければエラーを返します。
MAP_FROM_ENTRIES(array(row(K, V))) -> map(K, V)
keyとvalueのペアを並べたLISTをMAPに変換します。
SELECT MAP_FROM_ENTRIES( ARRAY[('a',1),('b',2),('c',NULL),('d',4),('e',5),('f',NULL)] )

MAP_ENTRIES(map(K, V)) -> array(row(K, V))
MAPを与えると，それをkeyとvalueのペアからなるLISTに変換します。
SELECT MAP_ENTRIES( MAP(ARRAY['a','b','c','d','e','f'], ARRAY[1,2,NULL,4,5,NULL]) )

MAP_CONCAT(map1(K, V), map2(K, V), ..., mapN(K, V)) -> map(K, V)
複数のMAPを連結します。MAPの連結には「||」が使えないので注意してください。keyが重複する場合には，いずれか1つに書き換えられます。
WITH map_table AS
(
  SELECT
    MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp1,
    MAP(ARRAY['c','d','e','f'], ARRAY[5,6,7,8]) AS mp2
)

SELECT MAP_CONCAT(mp1,mp2)
FROM map_table

MAP_FILTER(map(K, V), function(K, V, boolean)) -> map(K, V)
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,2,3,4]) AS mp )

SELECT MAP_FILTER( mp, (k,v)-> v>=3 )
FROM map_table

MAP_KEYS(x(K, V)), MAP_VALUES(x(K, V)) -> array(V)
MAPのkeyまたはvalueだけをLISTとして返します。
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,NULL,3,NULL]) AS mp )

SELECT MAP_KEYS(mp) AS keys, MAP_VALUES(mp) AS vals
FROM map_table

MAP_ZIP_WITH(map(K, V1), map(K, V2), function(K, V1, V2, V3)) -> map(K, V3)
2つのMAPの同じkeyのvalueに対して何らかのfunctionを適用し，1つのvalueにしてkeyに格納し直します。
片方にしかないkeyについては，もう片方の対応するvalueはNULLとして扱われます。
そして，片方がNULLのvalueである場合，大多数のfunctionではNULLが返ります。
WITH map_table AS
(
  SELECT
    MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp1,
    MAP(ARRAY['c','d','e','f'], ARRAY[5,6,7,NULL]) AS mp2
)

SELECT MAP_ZIP_WITH(mp1,mp2,(k,v1,v2)->v1+v2),
  MAP_ZIP_WITH(mp1,mp2,(k,v1,v2)->COALESCE(v1,0)+COALESCE(v2,0))
FROM map_table

TRANSFORM_KEYS(map(K1, V), function(K1, V, K2)) -> map(K2, V)
key自身にfunctionを適用して新しいkeyとし，それを使ってMAPを再構築します。新しいkeyがNULLになる場合はエラーとなります。
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp )

SELECT TRANSFORM_KEYS(mp,(k,v)->COALESCE(v,0)),
       TRANSFORM_KEYS(mp,(k,v)->k||CAST(COALESCE(v,0) AS VARCHAR))
FROM map_table

TRANSFORM_VALUES(map(K, V1), function(K, V1, V2)) -> map(K, V2)
MAPのvalueのみにfunctionを適用し，新しいvalueでMAPを再構築します。新しいvalueはNULLになってもかまいません。
WITH map_table AS
( SELECT MAP(ARRAY['a','b','c','d'], ARRAY[1,2,NULL,4]) AS mp )

SELECT TRANSFORM_VALUES(mp,(k,v)->v),
       TRANSFORM_VALUES(mp,(k,v)->k||CAST(v AS VARCHAR))
FROM map_table

