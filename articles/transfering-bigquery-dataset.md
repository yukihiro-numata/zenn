---
title: "異なるプロジェクト間でBigQueryのデータ転送を行う"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GoogleCloud", "BigQuery"]
published: true
---

# 概要
別プロジェクトのBigQueryに存在するデータを転送する手順について備忘録です。

# 動機
Firebaseは環境ごとにプロジェクトを分けているのですが、サーバーやDB、分析基盤で使用しているプロジェクトは1つという運用をしています。
FirebaseCloudMessaging（FCM）のデータを使ってプッシュ通知の送信数を分析するにあたり、今回の対応が必要になりました。
FCMはFirebaseと同じプロジェクトのBigQueryにしかデータ転送ができない仕様のため、分析基盤のプロジェクトからデータ参照ができず、データ転送が必要になったという経緯です。

# 具体的な手順
1. BigQueryコンソールから「データ転送」を選択します
![](https://storage.googleapis.com/zenn-user-upload/545eca84355b-20250307.png)

2. 「転送を作成」を選択します
![](https://storage.googleapis.com/zenn-user-upload/6f11ad09e1fd-20250307.png)

3. ソースに「Dataset Copy」を選択します
![](https://storage.googleapis.com/zenn-user-upload/bf46e37f25bc-20250307.png)

4. 転送設定を登録します
- 転送構成名
作成するデータ転送の名前です。任意の値を設定します。
- スケジュールオプション
転送タイミングを設定します。
FCMは[ドキュメント](https://firebase.google.com/docs/cloud-messaging/understand-delivery?hl=ja&_gl=1*1hlp5bd*_up*MQ..*_ga*MTQ1NTQ3MjQ0Mi4xNzQxMzQxNTkx*_ga_CW55HF8NVT*MTc0MTM0MTU5MS4xLjAuMTc0MTM0MTU5MS4wLjAuMA..&platform=ios#bigquery-data-export)によると、「太平洋時間の午前4時にデータの転送処理が始まり、24時間以内に完了する」とありますが、完了時間は日によってまちまちなので、決め打ちで設定するしかないと思います。
- 転送先の設定
転送してきたデータを格納するBigQueryデータセットを指定します。ここで新規にデータセットを作成することもできます。
- データソースの詳細
データの転送元を指定します。
FCMの場合、データセット名は通常であれば`firebase_messaging`となっています。
  
![](https://storage.googleapis.com/zenn-user-upload/87d933b8f22b-20250307.png)

※詳細は下記を参照すると良いです
https://cloud.google.com/bigquery/docs/managing-datasets#copy-datasets
