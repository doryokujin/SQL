# Lesson 01. SELECTによる「単純」抽出

## 全行，全カラムの抽出 [ SELECT * ]
```sql
SELECT * 
FROM sample_accesslog
```

|td_client_id|td_global_id             |td_title        |td_browser|td_host|td_path            |td_url    |td_referrer|td_ip|td_os     |td_language|time      |
|------------|-------------------------|----------------|----------|-------|-------------------|----------|-----------|-----|----------|-----------|----------|
|59523c90-2724-4d52-9157-1b93cd89f2a2|                         |リソース - Treasure Data|Chrome    |www.treasuredata.com|/jp/resources      |https://www.treasuredata.com/jp/resources|https://www.treasuredata.com/jp/|133.250.236.113|Windows 7 |ja         |1465977533|
|37c30508-18fa-422b-cdca-48ef68cbd29d|                         |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Mobile Safari|www.treasuredata.com|/jp/               |https://www.treasuredata.com/jp/|https://www.google.co.jp/|103.5.140.189|iOS       |ja-jp      |1465977276|
|e6d199fe-3367-4fb7-ab41-5422e30ca5c9|                         |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Mobile Safari|www.treasuredata.com|/jp/               |https://www.treasuredata.com/jp/?gclid=CMX_uou8qc0CFdgmvQodBrMD5w|           |106.133.82.194|iOS       |ja-jp      |1465974373|
|eb7339f6-5a55-4d60-f94d-56ddcb13138e|                         |Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|Chrome    |www.treasuredata.com|/jp/               |https://www.treasuredata.com/jp/|https://www.google.co.jp/|122.208.113.2|Windows 7 |ja         |1465974934|
|e244af8f-05b2-4aa0-8393-3fbb358f51a1|                         |サービス概要 - Treasure Data|Chrome    |www.treasuredata.com|/jp/service        |https://www.treasuredata.com/jp/service|https://www.treasuredata.com/jp/attribution|163.44.18.67|Windows 7 |ja         |1465975247|

## 全行，全カラムの抽出 [ SELECT col ]
```sql
SELECT time, td_client_id, td_title
FROM sample_accesslog
```
|time   |td_client_id             |td_title        |
|-------|-------------------------|----------------|
|1466410889|2705c701-3b7c-4ef3-d88d-50529a3d1060|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1466411086|a1265a65-70a0-4ee2-98fe-6671146666ee|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1466412618|83b53ce3-7c66-46aa-8c4d-cbec9e52bb86|Treasure Data - データ分析をクラウドで、シンプルに。 - Treasure Data|
|1466411214|7f47d05f-bd12-4553-e69c-763064738631|Fluentd Enterprise Edition Subscription - Treasure Data|
|1466412594|eb0b1d7e-0ddc-4297-ab4d-7100fda959f2|トレジャーデータサービスが、 株式会社MonotaRO（モノタロウ）のデータ分析基盤として導入 - プレスリリース - Treasure Data|

