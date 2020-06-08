# Lesson 07. 数学・統計関数

この章で紹介する数学関数は，集約関数とは異なり，指定したカラムのその行の値について個々に演算を行うものです。列全体を通して演算を行うものではないことに注意してください。

## ABS(x) → [same as input]：絶対値

絶対値を求める関数です。

```sql
SELECT val, ABS(val) AS abs_val
FROM ( VALUES -2, -1.1, 0, 1.1, 2 ) AS t(val)
```

## FLOOR(x)，CEILING(x)，ROUND(x, d) → [same as input]，TRUNCATE(x) → double：整形関数

FLOORは切り捨てを行う関数で，「引数の値より小さい最大の整数」を返します。CEILおよびCEILINGは切り上げを行う関数で，「引数の値より大きい最小の整数」を返します。
ROUNDは四捨五入を行う関数で，オプションの第2引数に1以上の整数を記入すれば，その小数点位置で四捨五入されます。例えば，ROUND(0.15, 1) = 0.2，ROUND(0.155, 2) = 0.16です。
TRUNCATEはやや特殊で，小数点を削ぎ落とした整数を返します。これは，正の値に関してはFLOORと同じ挙動を示しますが，負の値に関してはCEILINGと同じ挙動を示します。

```sql
SELECT val, 
  FLOOR(val) AS floor_val, 
  CEILING(val) AS ceiling_val, 
  TRUNCATE(val) AS truncate_val, 
  ROUND(val) round_val, ROUND(val,1) round_val2
FROM ( VALUES -1.50, -1.22, -0.65, -0.33, 0.15, 0.50, 0.99, 1.49, 1.50 ) AS t(val)
```

## POWER(x, p)，SQRT(x)，CBRT(x) → double：指数関数と平方根
POWER(x, p)はxpを意味します。例えば以下のようになります。
- POWER(2,2) = 2^2 = 4,
- POWER(3,2) = 3^2 = 9, 
- POWER(2,3) = 2^3 = 8,
- POWER(3,3) = 3^3 = 27
なお，x0は常に1，x1は常にxを返します。また，xは「底」，pは「指数」と呼ばれます。
SQRTは平方根，CBRTは三乗根の値を返します。例えば以下のようになります。
- SQRT(4) = √(2^2) = 2, 
- SQRT(9) = √(3^2) = 3, 
- CBRT(8)  = (2^3)の3乗根 = 2,
- CBRT(27) = (3^3)の3乗根 = 3
なお，√0および0の3乗根は常に0，√1および1の3乗根は常に1となります。
```sql
SELECT val, 
  POWER(val,2) AS pow2, SQRT(val) AS sqrt_val, SQRT(POWER(val,2)) AS val,
  POWER(val,3) AS pow3, CBRT(val) AS cbrt_val, CBRT(POWER(val,3)) AS val
FROM ( VALUES 0, 1, 2, 3, 4, 8, 9 ) AS t(val)
```

## LOG2(x)，LOG10(x) → double：対数関数

対数関数LOGは，先程紹介した指数関数（POWER）との間に次のような深い関係があります。
```2^3⇔3=log2(8)```
指数関数が引数の値に対して冪乗を計算するのに対して，対数関数は引数の値が「何乗された」結果なのかを計算するものです。例えば，log2(8)は「2を何乗したら8になるのか？」を求めることになります。また，log10(100)は「10を何乗したら100になるのか？（答えは2）」を求めることになります。より一般的に書くと，以下のような関係があります。
```ax=Mx=loga(M)```
このとき，aを「底」，Mを「真数」と呼びます。SQLの世界では，任意の底についての対数を計算できるわけではなく，ここで紹介した2と10のほか，後述するeの3種類だけが可能です。とはいえ，実用領域ではこれだけで十分です。

```sql
SELECT 2 AS base,
  LOG2(antilogarithm) AS logarithm, antilogarithm
FROM ( VALUES 0, 1, 2, 4, 8, 9 ) AS t(antilogarithm)
UNION ALL
SELECT 10 AS base,
  LOG10(antilogarithm) AS logarithm, antilogarithm
FROM ( VALUES 0, 1, 10, 100, 2) AS t(antilogarithm)
ORDER BY base ASC
```

## EXP(x)，LN(x) → double：指数関数eと自然対数
ネイピア数e（= 2.718281828…）を底とする特殊な（しかし最も有用な）指数関数をEXP(x)で計算します。また，eを底とする対数関数をLN(x)で計算できます。
```sql
SELECT 'e' AS base,
  LN(antilogarithm) AS logarithm, antilogarithm
FROM ( VALUES EXP(0), EXP(1), EXP(2) ) AS t(antilogarithm)
```

## NORMAL_CDF(mean, sd, x) → double：正規分布
引数で指定した平均meanと標準偏差sdに従う正規分布の累積分布関数の値を求めます。xはカラムの値で，あるカラムの値がこの正規分布に従うと仮定するならば，返り値は「x以下の値を取る確率」となります。
なお，これは「xの値を取る確率」とは異なります。「xの値を取る確率」は累積分布関数ではなく確率密度関数から得られるものであり，この2つの関数の区別は重要です。

左の図が確率密度関数の例です。確率変数Xの値がxである確率がY軸より求められます。
一方，右の図が累積分布関数の例です。この関数からは，確率変数Xの値がx以下である確率がY軸より求められます。言い換えれば，-からxまでの累積の確率を求めていることになり，取りうるすべてのxの値を含むと1に収束します。
繰り返しますが，NORMAL_CDF関数の結果は右図の累積分布関数の値です。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,4,4,4,4,5,5,5,5,5,6,6,6,6,7,7,7,8,8,9 ) AS t(val)
),
stat AS
 (
  SELECT AVG(val) AS av, STDDEV(val) AS sd
  FROM sample
)
SELECT DISTINCT val, NORMAL_CDF(av,sd,val) AS prob_cum, av, sd
FROM sample,stat
UNION ALL
SELECT 20 AS val, NORMAL_CDF(av,sd,20) AS prob_cum, av, sd
FROM stat
ORDER BY val
```

上記の例では，sampleテーブルのvalカラムの数値群（ここでは標本と呼ぶことにします）が，（この標本の平均および標準偏差に従う）正規分布に従って発生していると思われるときに，それぞれのカラム値の分布関数の値（カラム値以下となる確率）を求めています。

この結果からは，例えばval = 5がこの標本の平均であり，5までの値が出る確率が全体のちょうど半分の0.5となっていることがわかります。さらに，この正規分布に従って20の値が出るときの累積分布の値は，ほぼ1に近くなっていることがわかります。
仮に，限られた有限の値を持つ標本ですべての分布関数をこのように一覧できるのなら，（並び順で）隣同士の累積分布関数の差を求める以下のようなクエリの結果は何を意味しているでしょうか？

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,4,4,4,4,5,5,5,5,5,6,6,6,6,7,7,7,8,8,9 ) AS t(val)
),
stat AS
 (
  SELECT AVG(val) AS av, STDDEV(val) AS sd
  FROM sample
),
cum_norm AS
(
  SELECT DISTINCT val AS distinct_val, NORMAL_CDF(av,sd,val) AS prob_cum, av, sd
  FROM sample,stat
  UNION ALL
  SELECT 20 AS val, NORMAL_CDF(av,sd,20) AS prob_cum, av, sd
  FROM stat
)

SELECT pre_val, val, prob, SUM(prob)OVER(ORDER BY val RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS prob_cum
FROM
(
  SELECT LAG(distinct_val,1,-1*infinity())OVER(ORDER BY distinct_val) AS pre_val, distinct_val AS val, 
    prob_cum-LAG(prob_cum,1,0)OVER(ORDER BY distinct_val) AS prob
  FROM cum_norm
)
ORDER BY val
```

この結果のprobは，「pre_val < x <= val」の区間の確率を求めていることになります。

## INVERSE_NORMAL_CDF(mean, sd, p) → double

INVERSE_NORMAL_CDFは，累積分布確率p（0<p<1）が与えられたときに，元の確率変数xの値を求める逆関数です。

```sql
WITH sample AS (
  SELECT val
  FROM ( VALUES 1,2,2,3,3,3,4,4,4,4,5,5,5,5,5,6,6,6,6,7,7,7,8,8,9 ) AS t(val)
),
stat AS
 (
  SELECT AVG(val) AS av, STDDEV(val) AS sd
  FROM sample
)

SELECT INVERSE_NORMAL_CDF(av,sd,prob_cum) AS val_from_invnorm, val
FROM
(
  SELECT DISTINCT val, NORMAL_CDF(av,sd,val) AS prob_cum, av, sd
  FROM sample,stat
)
ORDER BY val
```

ベータ分布に関しても，同様に累積分布関数とその逆関数を求める関数がありますが，複雑なので省略します。

## RAND() → double：乱数

乱数を発生させる関数です。返される結果xは，0.0<=x<1.0の間のdouble型の値です。ただし，引数として整数nを指定すると，結果xは0<=x<n（nは含まない）の整数になります。RAND関数は負の値を返さないので，負の乱数が場合は自前での工夫が必要となります。
```sql
SELECT RAND() AS rnd1, RAND(10) AS rnd2
FROM ( VALUES 1,2,3,4,5 ) AS t(times)
```

上記のクエリでは，意味のない値（1〜5）のレコードを作り，その数だけ乱数を発生させています。また，RAND(10)により，0から9までの整数も返しています。

一般に，RAND関数では「乱数の種」と呼ばれる任意の文字列を設定でき，乱数の種が同じであれば何度RAND関数を実行しても同じ値の乱数列が得られます。つまり，乱数に再現性があります。しかし，Prestoでは，特筆すべき注意点として，乱数の種が指定できません。そのため実行のたびに乱数の値は変わってしまいます。

## STDDEV(x)，VARIANCE(x) → double：標準偏差，分散 （ただし，数学関数ではなく集約関数）

```sql
SELECT name, model, grade, AVG(price) AS avg_price, VARIANCE(price) AS var_price, STDDEV(price) AS stdev_price, COUNT(1) AS cnt
FROM usedcar
GROUP BY name, model, grade
HAVING 100 < COUNT(1)
ORDER BY var_price ASC
```

- 標本分散（サンプルの分散）：VAR_SAMP，VARIANCE
- 不偏分散（母集団の分散）：VAR_POP

## CORR(y, x) → double：相関係数（ただし，数学関数ではなく集約関数）

```sql
SELECT name, model, grade, CORR(odd_km,price) AS cor, COUNT(1) AS cnt
FROM usedcar
GROUP BY name, model, grade
HAVING 100 < COUNT(1)
ORDER BY cor ASC
```

## KURTOSIS(x)，SKEWNESS(x) → double：尖度，歪度（ただし，数学関数ではなく集約関数）

KURTOSIS（尖度）は，正規分布を0とする指標です。正規分布より鋭いピークと長く太い裾を持った分布では大きな尖度を持ち，より丸みがかったピークと短く細い尾を持った分布では小さな尖度を持ちます。

```sql
SELECT name, model, grade, KURTOSIS(odd_km) AS kurt, COUNT(1) AS cnt
FROM usedcar
GROUP BY name, model, grade
HAVING 100 < COUNT(1) AND KURTOSIS(odd_km) < 10
ORDER BY kurt DESC
```
SKEWNESS（歪度）は，分布の非対称性を示す指標です。
```sql
SELECT name, model, grade, SKEWNESS(odd_km) AS skew, COUNT(1) AS cnt
FROM usedcar
GROUP BY name, model, grade
HAVING 100 < COUNT(1)
ORDER BY skew DESC
```

## REGR_SLOPE(y, x)，REGR_INTERCEPT(y, x) → double：単純回帰（ただし，数学関数ではなく集約関数）

最も単純な線形回帰「y = ax + b」における切片b（INTERCEPT）と係数a（SLOPE）の値を求める関数です。下記の例では，負の相関が強かった車種を順番に並べています。

```sql
SELECT name, model, grade, CORR(price,odd_km) AS cor,
  REGR_SLOPE(price,odd_km) AS slope, REGR_INTERCEPT(price,odd_km) AS itcpt, COUNT(1) AS cnt
FROM usedcar
GROUP BY name, model, grade
HAVING 100 < COUNT(1)
ORDER BY cor ASC
```
