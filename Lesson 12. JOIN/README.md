# Lesson 12. JOIN

JOINは，複数の異なるテーブルを共通のキーを通じて横に結合する手法です。例えば上の3つのテーブルを見てみましょう。「sales_slip_sp」テーブルは購買履歴ログで，このタイプの「取引単位で（1回の購入ごとに）レコードが刻まれる」データは「トランザクションデータ」と呼ばれます。トランザクションデータでは，レコードは書き換えられることなく大量に蓄積されていきます。アクセスログや毎秒蓄積される走行履歴データなどの「時系列データ」も，同様のタイプのデータです。
このタイプのデータは非常に大量のレコードとなるため，レコードが持つ情報は最小限に抑えます。sales_slip_spでも，「いつ（time），誰が（member_id），何を（good_id），いくつ（amount）買ったのか」は必要な情報ですが，「member_idの属性（年齢，性別，住所など）」や「goods_idの属性（価格，詳細情報，正式名称）」はそのつど記録しておかなくても後から参照できる情報なので省かれています。

一方，ユーザー単位や商品単位でその属性を記録したデータは「マスタデータ」と呼ばれます。ユーザーや商品が増えない限りはレコードが増えることはなく，基本的には同じユーザーが複数のレコードになったりすることがない構成です。ただし，属性情報は後々変わることがあるので，レコードの書き換えで対応します。また，ユーザーが退会すればそのレコードは削除されることになり，「履歴」として残ることはありません。
JOINの主な役割は，時系列データの集計や分析に必要な属性情報をマスタデータから参照してきて結合（1つのテーブルに構成する）することにあります。それ以外にも，例えばバスケット分析のような組合せ計算へ応用できるCROSS JOINなど，JOINには非常に大きな活用可能性があります。

なお，すべてのJOINには方向性があり「A JOIN B」と「B JOIN A」では結果が異なってくる場合があるので注意してください。一般的には，JOINの前に書くテーブル（LEFT TABLE）を「時系列データ」，後（RIGHT TABLE）を「マスタデータ」にして，「'時系列データ' JOIN 'マスタデータ'」という形式とし，「時系列データに足りない属性情報を（結合キーを通じて）マスタデータから参照する」と解釈されます。

## INNER JOIN
INNER JOIN（省略してJOIN）では，相手が見つかった時系列データだけが抽出されます。相手が見つからなかった時系列レコードは除外されます。また，相手が複数見つかった場合には相手の数だけ自身のレコードが複製されて結合されます。ここでは，まず必ず相手が1つだけ見つかる理想的なケースでJOINを説明します。この場合，結果のレコード数は元の時系列データのレコード数と変わりません。

### レコード数が変わらないケース
#### 主に使用するデータ：sales_slip_sp
|receit_id                                  |member_id|goods_id                 |
|-------------------------------------------|---------|-------------------------|
|1                                          |member01'|1                        |
|2                                          |member02'|2                        |
|3                                          |member03'|3                        |
|4                                          |member01'|4                        |
|5                                          |member04'|1                        |

#### 主に使用するデータ：master_memers
|member_id                                  |mail|gender                   |age|
|-------------------------------------------|----|-------------------------|---|
|member01'                                  |takayama_kazuhisa@example.com'|m'                       |27 |
|member02'                                  |akiyama_hiromasa@example.com'|m'                       |33 |
|member03'                                  |iwasawa_kogan@example.com'|m'                       |30 |
|member04'                                  |takao_ayaka@example.com'|f'                       |40 |

## 主に使用するデータ：master_smartphones
|goods_id                                   |os |goods_name               |price|
|-------------------------------------------|---|-------------------------|-----|
|1                                          |iOS'|Apple iPhone XS 256GB'   |117480|
|2                                          |android'|ASUS ROG Phone 2 512GB'  |91080|
|3                                          |iOS'|Apple iPhone 8 256GB'    |78540|
|4                                          |android'|HUAWEI Mate 20 Pro'      |79200|
|5                                          |iOS'|Apple iPhone 7 32GB'     |37180|
|6                                          |android'|SHARP AQUOS zero'        |48180|


以下のクエリでは，「member_id」を結合キーとして，「sales_slip_sp」に登場したmember_idの属性情報（mail，gender，age）を「master_members」から取得しようとしています。sales_slip_spに登場するmember01，member02，member03，member04はすべてマスタデータにも存在しているので，相手が見つかることになります。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id FROM ( 
  VALUES 
(1,'member01',1),
(2,'member02',2),
(3,'member03',3),
(4,'member01',4),
(5,'member04',1)
) AS t(receit_id,member_id,goods_id) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',   'm',30),
('member04','takao_ayaka@example.com',      'f',40)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, goods_id, mail, gender, age 
FROM sales_slip_sp slip
JOIN master_members members
ON slip.member_id = members.member_id
```
|receit_id                                  |member_id|goods_id                 |mail|gender|age|
|-------------------------------------------|---------|-------------------------|----|------|---|
|1                                          |member01 |1                        |takayama_kazuhisa@example.com|m     |27 |
|2                                          |member02 |2                        |akiyama_hiromasa@example.com|m     |33 |
|3                                          |member03 |3                        |iwasawa_kogan@example.com|m     |30 |
|4                                          |member01 |4                        |takayama_kazuhisa@example.com|m     |27 |
|5                                          |member04 |1                        |takao_ayaka@example.com|f     |40 |


このクエリの中で，JOINは一番はじめに実行されます。
下記のクエリでは，「member_id」と「goods_id」を結合キーとして，「sales_slip_sp」に登場したmember_idの属性情報（mail，gender，age）を「master_members」から，goods_idの属性情報（os，goods_name，price）を「master_smartphones」から取得しようとしています。

```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id FROM ( 
  VALUES 
(1,'member01',1),
(2,'member02',2),
(3,'member03',3),
(4,'member01',4),
(5,'member04',1)
) AS t(receit_id,member_id,goods_id) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',   'm',30),
('member04','takao_ayaka@example.com',      'f',40)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, mail, gender, age, 
  slip.goods_id, os, goods_name, price
FROM sales_slip_sp slip
JOIN master_members members
ON slip.member_id = members.member_id
JOIN master_smartphones smartphones
ON slip.goods_id = smartphones.goods_id
```
|receit_id                                  |member_id|mail                     |gender|age|goods_id|os     |goods_name            |price |
|-------------------------------------------|---------|-------------------------|------|---|--------|-------|----------------------|------|
|1                                          |member01 |takayama_kazuhisa@example.com|m     |27 |1       |iOS    |Apple iPhone XS 256GB |117480|
|2                                          |member02 |akiyama_hiromasa@example.com|m     |33 |2       |android|ASUS ROG Phone 2 512GB|91080 |
|3                                          |member03 |iwasawa_kogan@example.com|m     |30 |3       |iOS    |Apple iPhone 8 256GB  |78540 |
|4                                          |member01 |takayama_kazuhisa@example.com|m     |27 |4       |android|HUAWEI Mate 20 Pro    |79200 |
|5                                          |member04 |takao_ayaka@example.com  |f     |40 |1       |iOS    |Apple iPhone XS 256GB |117480|


また，上の書き方は下のクエリと同義になります。JOINするほうのテーブルのすべてのカラムを必要としない場合には，「*」の代わりに必要なカラムだけ記述することでパフォーマンスが改善できます。

```sql
-- WITH 文は省略
SELECT receit_id, slip.member_id, mail, gender, age,
  slip.goods_id, os, goods_name, price
FROM sales_slip_sp slip
JOIN
( 
  SELECT * FROM master_members 
) members
ON slip.member_id = members.member_id
JOIN
( 
  SELECT * FROM master_smartphones
) smartphones
ON slip.goods_id = smartphones.goods_id
```

### レコード数が減るケース
次に，JOINの結果としてレコード数が減るケースを紹介します。これは，時系列データの中に，相手に恵まれなかったキーを持つ不幸なレコードがいたことを意味します。説明のため，使用するデータを以下のように少し変更します。黄色の網がけのレコードに注目してください。

#### 主に使用するデータ：sales_slip_sp
|receit_id                                  |member_id|goods_id                 |gender|os      |
|-------------------------------------------|---------|-------------------------|------|--------|
|1                                          |member01'|1                        |m'    |iOS'    |
|2                                          |member02'|2                        |m'    |android'|
|3                                          |member03'|3                        |m'    |iOS'    |
|4                                          |member01'|4                        |m'    |android'|
|5                                          |member04'|1                        |f'    |iOS'    |
|6                                          |member05'|4                        |f'    |android'|
|7                                          |member02'|7                        |m'    |android'|

#### 主に使用するデータ：master_memers
|member_id                                  |mail|gender                   |age|
|-------------------------------------------|----|-------------------------|---|
|member01'                                  |takayama_kazuhisa@example.com'|m'                       |27 |
|member02'                                  |akiyama_hiromasa@example.com'|m'                       |33 |
|member03'                                  |iwasawa_kogan@example.com'|m'                       |30 |
|member04'                                  |takao_ayaka@example.com'|f'                       |40 |
|member06'                                  |noriko_tanaka@example.com'|f'                       |22 |

#### 主に使用するデータ：master_smartphones
|goods_id                                   |os |goods_name               |price|
|-------------------------------------------|---|-------------------------|-----|
|1                                          |iOS'|Apple iPhone XS 256GB'   |117480|
|2                                          |android'|ASUS ROG Phone 2 512GB'  |91080|
|3                                          |iOS'|Apple iPhone 8 256GB'    |78540|
|4                                          |android'|HUAWEI Mate 20 Pro'      |79200|
|5                                          |iOS'|Apple iPhone 7 32GB'     |37180|
|6                                          |android'|SHARP AQUOS zero'        |48180|
|8                                          |android'|SHARP AQUOS sense3'      |21800|


#### receit_id = 6 が消滅する例

先程と同様に，member_idを結合キーとして属性情報を取得してみます。異なる点は，sales_slip_spにはマスタにない「member05」が存在し，かつmaster_membersには「member06」という時系列にないキーが存在していることです。このとき，マスタ側に相手が見つからなかったsales_slip_spのmember05のキーを持つレコード（receit_id = 6）は消滅することになります。
クエリを書く際の注意点として，JOINした後のカラムを記述する外側のSELECT節では結合キーがLEFT TABLEにもRIGHT TABLEにも存在するので，どちらのキーのカラムの値を取得するのかを明示する必要があることです。INNER JOINではどちらも値は同じですが，それ以外では異なってくることに注意してください。
なお，マスタ側に時系列を持たないキーのレコードがいくつあっても，INNER JOINの（この結合順序での）文脈においては結果に影響はありません。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, mail, members.gender, age, goods_id
FROM sales_slip_sp slip
JOIN master_members members
ON slip.member_id = members.member_id
```
|receit_id                                  |member_id|mail                     |gender|age|goods_id|
|-------------------------------------------|---------|-------------------------|------|---|--------|
|2                                          |member02 |akiyama_hiromasa@example.com|m     |33 |2       |
|1                                          |member01 |takayama_kazuhisa@example.com|m     |27 |1       |
|4                                          |member01 |takayama_kazuhisa@example.com|m     |27 |4       |
|3                                          |member03 |iwasawa_kogan@example.com|m     |30 |3       |
|5                                          |member04 |takao_ayaka@example.com  |f     |40 |1       |
|7                                          |member02 |akiyama_hiromasa@example.com|m     |33 |7       |


### receit_id = 7 が消滅する例

goods_idをキーにした結合も，マスタ側に存在しないgoods_id = 7のレコード（receit_id = 7）が消滅します。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, member_id, slip.goods_id, smartphones.os, goods_name, price
FROM sales_slip_sp slip
JOIN master_smartphones smartphones
ON slip.goods_id = smartphones.goods_id
```
|receit_id                                  |member_id|goods_id                 |os |goods_name|price|
|-------------------------------------------|---------|-------------------------|---|----------|-----|
|1                                          |member01 |1                        |iOS|Apple iPhone XS 256GB|117480|
|4                                          |member01 |4                        |android|HUAWEI Mate 20 Pro|79200|
|5                                          |member04 |1                        |iOS|Apple iPhone XS 256GB|117480|
|6                                          |member05 |4                        |android|HUAWEI Mate 20 Pro|79200|
|2                                          |member02 |2                        |android|ASUS ROG Phone 2 512GB|91080|
|3                                          |member03 |3                        |iOS|Apple iPhone 8 256GB|78540|

### receit_id = 6および7が消滅する例
member_idとgoods_idを結合キーにしてJOINを2回行うと，結果は想定の通りになります。

```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, mail, members.gender, age, 
  smartphones.goods_id, smartphones.os, goods_name, price
FROM sales_slip_sp slip
JOIN master_members  members
ON slip.member_id = members.member_id
JOIN master_smartphones smartphones
ON slip.goods_id = smartphones.goods_id
```
|receit_id                                  |member_id|mail                     |gender|age|goods_id|os     |goods_name            |price |
|-------------------------------------------|---------|-------------------------|------|---|--------|-------|----------------------|------|
|4                                          |member01 |takayama_kazuhisa@example.com|m     |27 |4       |android|HUAWEI Mate 20 Pro    |79200 |
|5                                          |member04 |takao_ayaka@example.com  |f     |40 |1       |iOS    |Apple iPhone XS 256GB |117480|
|1                                          |member01 |takayama_kazuhisa@example.com|m     |27 |1       |iOS    |Apple iPhone XS 256GB |117480|
|2                                          |member02 |akiyama_hiromasa@example.com|m     |33 |2       |android|ASUS ROG Phone 2 512GB|91080 |
|3                                          |member03 |iwasawa_kogan@example.com|m     |30 |3       |iOS    |Apple iPhone 8 256GB  |78540 |


### レコード数が増えるケース（間違い）
相手が見つからなくてレコード数が減ってしまうだけでなく，相手が複数見つかることでレコード数が意図せずに増えてしまうケースも存在します。レコードが増えるINNER JOINでは，データかクエリのどちらかに不備がある場合が多くあります。
以下のクエリは，「gender」を結合キーにしてmaster_membersと結合してしまっているケースです。genderを結合キーにしてしまうと，同じキーのレコードが複数あることになります（「m」が3レコード，「f」が2レコード）。その結果，sales_slip_spのレコードでは，genderが「m」のレコードは3つのマスタ側のレコードと結合し，3レコードに増えることになります。「f」のレコードも同様に2レコードに増えることになります。この結果はもちろん正しい参照ではないので間違ったケースとなります。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, slip.gender, mail, age
FROM sales_slip_sp slip
JOIN master_members members
ON slip.gender = members.gender
```
|receit_id                                  |member_id|gender                   |mail|age|
|-------------------------------------------|---------|-------------------------|----|---|
|1                                          |member01 |m                        |iwasawa_kogan@example.com|30 |
|1                                          |member01 |m                        |akiyama_hiromasa@example.com|33 |
|1                                          |member01 |m                        |takayama_kazuhisa@example.com|27 |
|...                                        |         |                         |    |   |
|7                                          |member02 |m                        |iwasawa_kogan@example.com|30 |
|7                                          |member02 |m                        |akiyama_hiromasa@example.com|33 |
|7                                          |member02 |m                        |takayama_kazuhisa@example.com|27 |


また，master_smartphonesからosを結合キーにしても同様の間違った結果となります。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, slip.gender, slip.goods_id, slip.os, goods_name, price
FROM sales_slip_sp slip
JOIN master_members members
ON slip.gender = members.gender
JOIN master_smartphones smartphones
ON slip.os = smartphones.os
```
|receit_id                                  |member_id|gender                   |goods_id|os |goods_name            |price |
|-------------------------------------------|---------|-------------------------|--------|---|----------------------|------|
|1                                          |member01 |m                        |1       |iOS|Apple iPhone XS 256GB |117480|
|1                                          |member01 |m                        |1       |iOS|Apple iPhone 7 32GB   |37180 |
|1                                          |member01 |m                        |1       |iOS|Apple iPhone 8 256GB  |78540 |
|...                                        |         |                         |        |   |                      |      |
|3                                          |member03 |m                        |3       |iOS|Apple iPhone XS 256GB |117480|
|3                                          |member03 |m                        |3       |iOS|Apple iPhone 7 32GB   |37180 |
|3                                          |member03 |m                        |3       |iOS|Apple iPhone 8 256GB  |78540 |

## LEFT OUTER JOIN
LEFT OUTER JOINは，相手が見つからなかったレコードに対して除外するのではなく，属性項目をNULLで埋めることによってレコードを残すことを目的とした方法です。元のレコードが消滅して見えなくなるのと，明示的にNULLによって「結合しなかったレコード」として残すのとでは，大きく意味合いが違います。

### レコードが変わらないケース
以下のクエリによりmember_idを結合キーにしてLEFT OUTER JOINした結果は，マスタ側で存在しないmember05の属性情報（geder，age）がNULLで埋められたものになります。重要なのは，LEFTとRIGHTのmember_idの値が異なることで，「slip.member_id=member05」に対して「members.member_id=NULL」になります。よって，一番外側のSELECT節では，どちらのテーブルの結合キーの値を取ってくるのかに注意する必要があります（多くはLEFT側です）。
その他のレコードについてはINNER JOINのときの結果と変わりません。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, members.member_id, mail, members.gender, age, goods_id
FROM sales_slip_sp slip
LEFT OUTER JOIN master_members members
ON slip.member_id = members.member_id
```
|receit_id                                  |member_id|member_id                |mail|gender|age                   |goods_id|
|-------------------------------------------|---------|-------------------------|----|------|----------------------|--------|
|2                                          |member02 |member02                 |akiyama_hiromasa@example.com|m     |33                    |2       |
|3                                          |member03 |member03                 |iwasawa_kogan@example.com|m     |30                    |3       |
|1                                          |member01 |member01                 |takayama_kazuhisa@example.com|m     |27                    |1       |
|4                                          |member01 |member01                 |takayama_kazuhisa@example.com|m     |27                    |4       |
|5                                          |member04 |member04                 |takao_ayaka@example.com|f     |40                    |1       |
|6                                          |member05 |NULL                         |NULL    |NULL      |NULL                      |4       |
|7                                          |member02 |member02                 |akiyama_hiromasa@example.com|m     |33                    |7       |

member_idとgoods_idで同時にLEFT OUTER JOINすれば，各々で結合できなかった属性情報の値がNULLで埋まっていることが確認できます。

```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, mail, members.gender, age, 
  smartphones.goods_id, smartphones.os, goods_name, price
FROM sales_slip_sp slip
LEFT OUTER JOIN master_members members
ON slip.member_id = members.member_id
LEFT OUTER JOIN master_smartphones smartphones
ON slip.goods_id = smartphones.goods_id
```
|receit_id                                  |member_id|mail                     |gender|age|goods_id              |os    |goods_name            |price |
|-------------------------------------------|---------|-------------------------|------|---|----------------------|------|----------------------|------|
|2                                          |member02 |akiyama_hiromasa@example.com|m     |33 |2                     |android|ASUS ROG Phone 2 512GB|91080 |
|1                                          |member01 |takayama_kazuhisa@example.com|m     |27 |1                     |iOS   |Apple iPhone XS 256GB |117480|
|3                                          |member03 |iwasawa_kogan@example.com|m     |30 |3                     |iOS   |Apple iPhone 8 256GB  |78540 |
|4                                          |member01 |takayama_kazuhisa@example.com|m     |27 |4                     |android|HUAWEI Mate 20 Pro    |79200 |
|5                                          |member04 |takao_ayaka@example.com  |f     |40 |1                     |iOS   |Apple iPhone XS 256GB |117480|
|6                                          |member05 |NULL                         |NULL      |NULL   |4                     |android|HUAWEI Mate 20 Pro    |79200 |
|7                                          |member02 |akiyama_hiromasa@example.com|m     |33 |NULL                      |NULL      |NULL                      |NULL      |

### 結合できなかったレコードのほうを抽出する
LEFT OUTER JOINをうまく使えば，「結合できなかった」レコードのみを抽出することが可能です。INNER JOINでは結合できなかったレコードが消滅して特定できなかったのに対し，RIGHT TABLEの結合キーがNULLとなって残るこちらの方法であれば「WHERE IS NULL」で抽出できるからです。先程の2つのクエリで試してみましょう。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, members.member_id, mail, members.gender, age, goods_id
FROM sales_slip_sp slip
LEFT OUTER JOIN master_members members
ON slip.member_id = members.member_id
WHERE members.member_id IS NULL
```
|receit_id                                  |member_id|member_id                |mail|gender|age                   |goods_id|
|-------------------------------------------|---------|-------------------------|----|------|----------------------|--------|
|6                                          |member05 |NULL                         |NULL    |NULL      |NULL                      |4       |


```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, mail, members.gender, age, 
  smartphones.goods_id, smartphones.os, goods_name, price
FROM sales_slip_sp slip
LEFT OUTER JOIN master_members members
ON slip.member_id = members.member_id
LEFT OUTER JOIN master_smartphones smartphones
ON slip.goods_id = smartphones.goods_id
WHERE members.member_id IS NULL OR smartphones.goods_id IS NULL
```
|receit_id                                  |member_id|mail                     |gender|age|goods_id              |os    |goods_name        |price|
|-------------------------------------------|---------|-------------------------|------|---|----------------------|------|------------------|-----|
|6                                          |member05 |NULL                         |NULL      |NULL   |4                     |android|HUAWEI Mate 20 Pro|79200|
|7                                          |member02 |akiyama_hiromasa@example.com|m     |33 |NULL                      |NULL      |NULL                  |NULL     |

### マスタデータがLEFT TABLEとなるケース
INNER JOINの説明では，時系列データがLEFT TABLE（JOINの前に書く），マスタデータがRIGHT TABLE（JOINの後に書く）と説明しましたが，特殊な目的においてはマスタデータがLEFT TABLEになることがあります。それは，単純な入れ替えではなく，「マスタデータと（集計した）時系列データを結合する」場合です。
今，それぞれのmember_idごとの過去の購入総額と購入名目を知りたいという目的があったとします。求められる結果は，member_idごとに1レコードです。この場合，時系列データのほうがmember_idをGROUP BYキーにして集計され，それがマスタ情報となって結合されることになります。ただし，購入総額が0のユーザーも0として集計したいので，LEFT OUTER JOINによって結合せずNULLとなった該当部分を0で埋める作業を行います。

もし逆に，集計した時系列データをLEFT TABLE，master_membersをRIGHT TABLEとしてLEFT OUTER JOINした場合には，集計で0だったユーザーはLEFTに登場しないので，結合した結果にそのユーザーが現れません。そのため，購入総額0も含めたすべてのユーザーを集計するには，やはりマスタテーブルをLEFT TABLEにするしかありません。

```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT members.member_id, COALESCE(sales,0) AS sales, goods_names,
  mail, gender, age
FROM master_members members
LEFT OUTER JOIN
( 
  SELECT member_id, SUM(price) AS sales, ARRAY_AGG(goods_name) AS goods_names
  FROM sales_slip_sp slip
  JOIN
  ( SELECT goods_id, price, goods_name FROM master_smartphones ) smartphones
  ON slip.goods_id = smartphones.goods_id
  GROUP BY member_id
) slip
ON members.member_id = slip.member_id
ORDER BY member_id
```
|member_id                                  |sales|goods_names              |mail|gender|age                   |
|-------------------------------------------|-----|-------------------------|----|------|----------------------|
|member01                                   |196680|["Apple iPhone XS 256GB", "HUAWEI Mate 20 Pro"]|takayama_kazuhisa@example.com|m     |27                    |
|member02                                   |91080|["ASUS ROG Phone 2 512GB"]|akiyama_hiromasa@example.com|m     |33                    |
|member03                                   |78540|["Apple iPhone 8 256GB"] |iwasawa_kogan@example.com|m     |30                    |
|member04                                   |117480|["Apple iPhone XS 256GB"]|takao_ayaka@example.com|f     |40                    |
|member06                                   |0    |NULL                         |noriko_tanaka@example.com|f     |22                    |


## RIGHT OUTER JOIN
RIGHT OUTER JOINは，LEFT OUTER JOINのテーブルの順序を入れ替えたものです。
以下のクエリは，master_membersに対してsales_slip_spを結合していることになるので，今まで書いていた方向とは逆になっています。このクエリの結果では，購入のあったmember_idについては購入した分だけのレコード数が現れ，購入のなかったmember_idについては購入情報がNULLとなった結果が得られることになります。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT members.member_id, receit_id, mail, members.gender, age, goods_id
FROM sales_slip_sp slip
RIGHT OUTER JOIN master_members members
ON members.member_id = slip.member_id
ORDER BY members.member_id, receit_id
```
|member_id                                  |receit_id|mail                     |gender|age|goods_id              |
|-------------------------------------------|---------|-------------------------|------|---|----------------------|
|member01                                   |1        |takayama_kazuhisa@example.com|m     |27 |1                     |
|member01                                   |4        |takayama_kazuhisa@example.com|m     |27 |4                     |
|member02                                   |2        |akiyama_hiromasa@example.com|m     |33 |2                     |
|member02                                   |7        |akiyama_hiromasa@example.com|m     |33 |7                     |
|member03                                   |3        |iwasawa_kogan@example.com|m     |30 |3                     |
|member04                                   |5        |takao_ayaka@example.com  |f     |40 |1                     |
|member06                                   |NULL         |noriko_tanaka@example.com|f     |22 |NULL                      |


以下のクエリは，これまでの例で書いていたLEFT OUTER JOINと同じ順序になっているので，同じ結果が得られます。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT smartphones.goods_id, receit_id, member_id, os, goods_name, price
FROM master_smartphones smartphones
RIGHT OUTER JOIN
( 
  SELECT receit_id, member_id, goods_id  FROM sales_slip_sp
) slip
ON smartphones.goods_id = slip.goods_id
ORDER BY goods_id, receit_id
```
|goods_id                                   |receit_id|member_id                |os |goods_name|price                 |
|-------------------------------------------|---------|-------------------------|---|----------|----------------------|
|1                                          |1        |member01                 |iOS|Apple iPhone XS 256GB|117480                |
|1                                          |5        |member04                 |iOS|Apple iPhone XS 256GB|117480                |
|2                                          |2        |member02                 |android|ASUS ROG Phone 2 512GB|91080                 |
|3                                          |3        |member03                 |iOS|Apple iPhone 8 256GB|78540                 |
|4                                          |4        |member01                 |android|HUAWEI Mate 20 Pro|79200                 |
|4                                          |6        |member05                 |android|HUAWEI Mate 20 Pro|79200                 |
|NULL                                       |7        |member02                 |NULL|NULL      |NULL                  |


## FULL OUTER JOIN
FULL OUTER JOINは，LEFT TABLEとRIGHT TABLEに順序をつけず，双方の結合キーをすべて残したまま，片側で結合できなかった情報はNULLで埋めて返す手法です。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, members.member_id, mail, members.gender, age, goods_id
FROM sales_slip_sp slip
FULL OUTER JOIN master_members members
ON slip.member_id = members.member_id
```
|receit_id                                  |member_id|member_id                |mail|gender|age                   |goods_id|
|-------------------------------------------|---------|-------------------------|----|------|----------------------|--------|
|6                                          |member05 |NULL                     |NULL|NULL  |NULL                  |4       |
|5                                          |member04 |member04                 |takao_ayaka@example.com|f     |40                    |1       |
|3                                          |member03 |member03                 |iwasawa_kogan@example.com|m     |30                    |3       |
|1                                          |member01 |member01                 |takayama_kazuhisa@example.com|m     |27                    |1       |
|2                                          |member02 |member02                 |akiyama_hiromasa@example.com|m     |33                    |2       |
|4                                          |member01 |member01                 |takayama_kazuhisa@example.com|m     |27                    |4       |
|7                                          |member02 |member02                 |akiyama_hiromasa@example.com|m     |33                    |7       |
|NULL                                       |NULL     |member06                 |noriko_tanaka@example.com|f     |22                    |NULL    |


```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, mail, members.gender, age, 
  smartphones.goods_id, smartphones.os, goods_name, price
FROM sales_slip_sp slip
FULL OUTER JOIN master_members members
ON slip.member_id = members.member_id
FULL OUTER JOIN master_smartphones smartphones
ON slip.goods_id = smartphones.goods_id
```
|receit_id                                  |member_id|mail                     |gender|age|goods_id              |os  |goods_name            |price |
|-------------------------------------------|---------|-------------------------|------|---|----------------------|----|----------------------|------|
|NULL                                       |NULL     |noriko_tanaka@example.com|f     |22 |NULL                  |NULL|NULL                  |NULL  |
|6                                          |member05 |NULL                     |NULL  |NULL|4                     |android|HUAWEI Mate 20 Pro    |79200 |
|5                                          |member04 |takao_ayaka@example.com  |f     |40 |1                     |iOS |Apple iPhone XS 256GB |117480|
|3                                          |member03 |iwasawa_kogan@example.com|m     |30 |3                     |iOS |Apple iPhone 8 256GB  |78540 |
|4                                          |member01 |takayama_kazuhisa@example.com|m     |27 |4                     |android|HUAWEI Mate 20 Pro    |79200 |
|1                                          |member01 |takayama_kazuhisa@example.com|m     |27 |1                     |iOS |Apple iPhone XS 256GB |117480|
|7                                          |member02 |akiyama_hiromasa@example.com|m     |33 |                      |    |                      |      |
|2                                          |member02 |akiyama_hiromasa@example.com|m     |33 |2                     |android|ASUS ROG Phone 2 512GB|91080 |
|NULL                                       |NULL     |NULL                     |NULL  |NULL|5                     |iOS |Apple iPhone 7 32GB   |37180 |
|NULL                                       |NULL     |NULL                     |NULL  |NULL|6                     |android|SHARP AQUOS zero      |48180 |
|NULL                                       |NULL     |NULL                     |NULL  |NULL|8                     |android|SHARP AQUOS sense3    |21800 |


## WHEREとONの違い
JOINにおけるON節にはWHERE節と同じ条件式を記述しますが，これら2つの実行順序（それゆえのパフォーマンス）の違いを知ることは大変重要です。

### ON → JOIN（推奨）
ON節は，JOINが実行される前に先に双方のテーブルに対して評価されます。必要なレコードのみが抽出された後にJOINが実行されるので，後述するWHERE節の併用よりも効率が良くなります。
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, age
FROM sales_slip_sp slip
JOIN master_members members
ON slip.member_id = members.member_id
AND 30 <= members.age
```
|receit_id                                  |member_id|age                      |
|-------------------------------------------|---------|-------------------------|
|2                                          |member02 |33                       |
|5                                          |member04 |40                       |
|7                                          |member02 |33                       |
|3                                          |member03 |30                       |

ただしHive0.13ではON節の制約があり，上記のクエリは動くものの，以下のような両テーブルのキーの比較で「=」以外を使ったクエリは動かないので注意してください。上記のクエリは定数と片方のテーブルmembers.ageの比較なので「=」以外が使えています。

```sql
-- Hive
WITH sales_slip_sp AS 
( SELECT STACK(
    7,
    1,'member01',1,'m',    'iOS',
    2,'member02',2,'m','android',
    3,'member03',3,'m',    'iOS',
    4,'member01',4,'m','android',
    5,'member04',1,'f',    'iOS',
    6,'member05',4,'f','android',
    7,'member02',7,'m','android'
  ) AS (receit_id,member_id,goods_id,gender,os)
),

master_members AS
( SELECT STACK(
    5,
    'member01','takayama_kazuhisa@example.com','m',27,
    'member02','akiyama_hiromasa@example.com', 'm',33,
    'member03','iwasawa_kogan@example.com',    'm',30,
    'member04','takao_ayaka@example.com',      'f',40,
    'member06','noriko_tanaka@example.com',    'f',22
  ) AS (member_id,mail,gender,age)
)

SELECT receit_id, slip.member_id, age
FROM sales_slip_sp slip
JOIN master_members members
ON slip.member_id = members.member_id
AND slip.goods_id <= members.age
```

Hive0.13で上記を実行すると
```sql
SemanticException: Line 33:4 Both left and right aliases encountered in JOIN 'age'
```
というエラーになります。

### JOIN → WHERE（非推奨）
WHERE節はJOINよりも後に実行されます。例えば以下のクエリは，まずJOINが行われた後に必要なレコードが抽出されるので，結果よりも大きなレコードがJOIN時にいったん生成されることになり，メモリおよびパフォーマンスに悪影響を及ぼします。

```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT receit_id, slip.member_id, age
FROM sales_slip_sp slip
JOIN master_members members
ON slip.member_id = members.member_id
WHERE 30 <= members.age
```
|receit_id                                  |member_id|age                      |
|-------------------------------------------|---------|-------------------------|
|5                                          |member04 |40                       |
|2                                          |member02 |33                       |
|7                                          |member02 |33                       |
|3                                          |member03 |30                       |


しかしながら，ON節に制約のあるHiveでは，「=」以外の両テーブルのキー比較はWHERE節で書くしかありません。
```sql
-- Hive
WITH sales_slip_sp AS 
( SELECT STACK(
    7,
    1,'member01',1,'m',    'iOS',
    2,'member02',2,'m','android',
    3,'member03',3,'m',    'iOS',
    4,'member01',4,'m','android',
    5,'member04',1,'f',    'iOS',
    6,'member05',4,'f','android',
    7,'member02',7,'m','android'
  ) AS (receit_id,member_id,goods_id,gender,os)
),

master_members AS
( SELECT STACK(
    5,
    'member01','takayama_kazuhisa@example.com','m',27,
    'member02','akiyama_hiromasa@example.com', 'm',33,
    'member03','iwasawa_kogan@example.com',    'm',30,
    'member04','takao_ayaka@example.com',      'f',40,
    'member06','noriko_tanaka@example.com',    'f',22
  ) AS (member_id,mail,gender,age)
)

SELECT receit_id, slip.member_id, age
FROM sales_slip_sp slip
JOIN master_members members
ON slip.member_id = members.member_id
WHERE slip.goods_id <= members.age
```
|receit_id                                  |member_id|age                      |
|-------------------------------------------|---------|-------------------------|
|4                                          |member01 |27                       |
|1                                          |member01 |27                       |
|7                                          |member02 |33                       |
|2                                          |member02 |33                       |
|3                                          |member03 |30                       |
|5                                          |member04 |40                       |


### WHERE TD_TIME_RANGE → JOIN（推奨）
一番はじめに実行されるTD_TIME_RANGEなどのTIME/DATE UDFは，WHERE節のほうに記述しないと効かないので注意してください。

```sqlSELECT goods_id, members.member_id, age
FROM sales_slip slip
LEFT OUTER JOIN
( 
  SELECT member_id, gender, age FROM master_members
) members
ON slip.member_id = members.member_id AND slip.member_id IS NOT NULL
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
-- AND TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST') --NG
```
|goods_id                                   |member_id|age                      |
|-------------------------------------------|---------|-------------------------|
|548453                                     |920431   |47                       |
|537833                                     |1225349  |35                       |
|544411                                     |233096   |63                       |

```sql
** Time indexes:
  - tdce.sales_slip : [2010-12-31 15:00:00 UTC, 2011-12-31 14:59:59 UTC]
**
```
上記のクエリのWHEREをANDにしてしまうと，時間指定によるTimeIndex（必要なパーティ書ションにのみアクセス）が効きません。

### WHERE TD_TIME_RANGE → ON → JOIN（推奨）
以上をまとめると，先読みしたい時間範囲だけをWHERE節に記述し，その他はなるべくON節に記述するのがセオリーです。

```sql
SELECT goods_id, members.member_id, age
FROM sales_slip slip
LEFT OUTER JOIN
( 
  SELECT member_id, gender, age FROM master_members
) members
ON slip.member_id = members.member_id 
AND 30 <= members.age
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
```
|goods_id                                   |member_id|age                      |
|-------------------------------------------|---------|-------------------------|
|478234                                     |452218   |71                       |
|533750                                     |1328980  |36                       |
|471748                                     |96391    |57                       |


## JOINと集計を同時に
JOINした後で集計を行うケースはよくあります。以下にいくつか例を挙げてみました。以降の章では，より複雑で応用的なクエリがたくさん出てくるので，その際に混乱したら一度本章に戻ってきてください。

### グッズ名ごとの売上点数
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT goods_name, COUNT(1) AS cnt
FROM sales_slip_sp slip
LEFT OUTER JOIN
( 
  SELECT * FROM master_smartphones
) smartphones
ON slip.goods_id = smartphones.goods_id
GROUP BY goods_name
```　　　　　　　　　　　
|goods_name                                 |cnt|
|-------------------------------------------|---|
|HUAWEI Mate 20 Pro                         |2  |
|Apple iPhone XS 256GB                      |2  |
|ASUS ROG Phone 2 512GB                     |1  |
|NULL                                       |1  |
|Apple iPhone 8 256GB                       |1  |


### グッズ名ごとの購入者の平均年齢
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT goods_name, COUNT(1) AS cnt, AVG(age) AS avg_age
FROM sales_slip_sp slip
LEFT OUTER JOIN
( 
  SELECT member_id, gender, age FROM master_members 
) members
ON slip.member_id = members.member_id
LEFT OUTER JOIN
( 
  SELECT goods_id, goods_name, price FROM master_smartphones
) smartphones
ON slip.goods_id = smartphones.goods_id
GROUP BY goods_name
```
|goods_name                                 |cnt|avg_age|
|-------------------------------------------|---|-------|
|ASUS ROG Phone 2 512GB                     |1  |33.0   |
|Apple iPhone 8 256GB                       |1  |30.0   |
|HUAWEI Mate 20 Pro                         |2  |27.0   |
|Apple iPhone XS 256GB                      |2  |33.5   |
|NULL                                       |1  |33.0   |


### グッズ名ごとの購入者の男女別平均年齢
```sql
WITH sales_slip_sp AS 
( SELECT receit_id,member_id,goods_id,gender,os FROM ( 
  VALUES 
(1,'member01',1,'m',    'iOS'),
(2,'member02',2,'m','android'),
(3,'member03',3,'m',    'iOS'),
(4,'member01',4,'m','android'),
(5,'member04',1,'f',    'iOS'),
(6,'member05',4,'f','android'),
(7,'member02',7,'m','android')
) AS t(receit_id,member_id,goods_id,gender,os) ),

master_members AS
( SELECT member_id,mail,gender,age FROM ( 
  VALUES
('member01','takayama_kazuhisa@example.com','m',27),
('member02','akiyama_hiromasa@example.com', 'm',33),
('member03','iwasawa_kogan@example.com',    'm',30),
('member04','takao_ayaka@example.com',      'f',40),
('member06','noriko_tanaka@example.com',    'f',22)
) AS t(member_id,mail,gender,age) ),

master_smartphones AS 
( SELECT goods_id,os,goods_name,price FROM (
  VALUES
(1,'iOS',    'Apple iPhone XS 256GB',117480),
(2,'android','ASUS ROG Phone 2 512GB',91080),
(3,'iOS',    'Apple iPhone 8 256GB',   78540),
(4,'android','HUAWEI Mate 20 Pro',    79200),
(5,'iOS',    'Apple iPhone 7 32GB',   37180),
(6,'android','SHARP AQUOS zero',      48180),
(8,'android','SHARP AQUOS sense3',    21800)
) AS t(goods_id,os,goods_name,price) )

SELECT goods_name, 
  COUNT(IF(members.gender='m',1,NULL)) AS cnt_m,
  COUNT(IF(members.gender='f',1,NULL)) AS cnt_f,
  AVG(IF(members.gender='m',age,NULL)) AS avg_age_m,
  AVG(IF(members.gender='f',age,NULL)) AS avg_age_f
FROM sales_slip_sp slip
LEFT OUTER JOIN
( 
  SELECT member_id, gender, age FROM master_members 
) members
ON slip.member_id = members.member_id
LEFT OUTER JOIN
( 
  SELECT goods_id, goods_name, price FROM master_smartphones
) smartphones
ON slip.goods_id = smartphones.goods_id
GROUP BY goods_name
```
|goods_name                                 |cnt_m|cnt_f|avg_age_m|avg_age_f|
|-------------------------------------------|-----|-----|---------|---------|
|ASUS ROG Phone 2 512GB                     |1    |0    |33.0     |NULL     |
|Apple iPhone 8 256GB                       |1    |0    |30.0     |NULL     |
|Apple iPhone XS 256GB                      |1    |1    |27.0     |40.0     |
|NULL                                       |1    |0    |33.0     |NULL     |
|HUAWEI Mate 20 Pro                         |1    |0    |27.0     |NULL     |

### より多くのデータでの実行例
最後に，より多くのデータで同じことをやってみましょう。以下では1レシートあたりの男女の平均購入価格も加えて求めています。

```sql
SELECT goods_id, 
  COUNT(IF(members.gender='男',1,NULL)) AS cnt_m,
  COUNT(IF(members.gender='女',1,NULL)) AS cnt_f,
  AVG(IF(members.gender='男',price*amount,NULL)) AS avg_sales_per_receit_m,
  AVG(IF(members.gender='女',price*amount,NULL)) AS avg_sales_per_receit_f,
  AVG(IF(members.gender='男',age,NULL)) AS avg_age_m,
  AVG(IF(members.gender='女',age,NULL)) AS avg_age_f
FROM sales_slip slip
LEFT OUTER JOIN
( 
  SELECT member_id, gender, age FROM master_members
) members
ON slip.member_id = members.member_id
WHERE TD_TIME_RANGE(time, '2011-01-01','2012-01-01','JST')
GROUP BY goods_id
ORDER BY cnt_m+cnt_f DESC
```
|goods_id                                   |cnt_m|cnt_f|avg_sales_per_receit_m|avg_sales_per_receit_f|avg_age_m         |avg_age_f         |
|-------------------------------------------|-----|-----|----------------------|----------------------|------------------|------------------|
|547453                                     |1123 |1090 |154.22083704363314    |153.19266055045873    |50.454140694568125|50.10825688073395 |
|541456                                     |449  |394  |932.0445434298441     |910.4568527918782     |52.55902004454343 |50.621827411167516|
|109601                                     |259  |461  |301.1583011583012     |371.5835140997831     |45.84169884169884 |49.22776572668113 |


