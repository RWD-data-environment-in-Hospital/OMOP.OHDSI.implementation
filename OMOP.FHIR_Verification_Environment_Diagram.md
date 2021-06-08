# OMOP／FHIR検証環境の構成図





------

## １．目的

OMOP／FHIR検証の環境をご説明いたします。



------
## ２．システムとネットワーク構成図

![image-20210403070796524](../images/OMOP.FHIR_Verification_Environment_Diagram/image-20210518133402167.png)

------
## ３．開発環境

### ３．１．OMOP／FHIR

### ①Windows10 Pro (EPSON)

![image-20210403070796524](../images/OMOP.FHIR_Verification_Environment_Diagram/image-20210518133474635.png)

| 用途                                  | 名称                 | バージョン      |
| ------------------------------------- | -------------------- | --------------- |
| More ActionsFHIR/OMOPプラットフォーム | Docker Engine CentOS | 20.10.3         |
| FHIRサーバー                          | Hapi FHIR JPA Server | 5.2.0           |
| Achiless / CohortMethod               | HADES                |                 |
| OMOP分析ツール1                       | Achiless             | 1.6.3           |
| OMOP分析ツール2                       | CohortMethod         | 4.1.0           |
| OMOPデータ表示ツール                  | Broadsea             | 2.8.0           |
| R開発環境                             | R Studio             | 1.4             |
| R Shiny用Webサーバー                  | Shiny Server         | 1.5.16.958      |
| ID変換ツール                          | Python               | 3.7             |
| 匿名化ツール                          | NESTGate一式         |                 |
| データベース                          | PostgreSQL           | 10              |
| VMソフト                              | VMware-player        | 16.1.0-17198959 |
| ファイル転送ソフト                    | WinSCP               |                 |
| DB接続ソフト                          | A5:SQL Mk-2          | 2.15.2_x64      |
| ターミナルソフト                      | teraterm             | 4.105           |
| エディタ                              | sakura               |                 |

------
