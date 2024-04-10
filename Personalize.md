「Users」、「Items」、「interaction」データについてそれぞれ「schema」を指定する必要がある。

「schema」は、各データの定義情報。

例えば、下記は「Users」データの「schema」(データ構造)の例。

```jsx
{
  "type": "record",
  "name": "Users",
  "namespace": "com.amazonaws.personalize.schema",
  "fields": [
      {
          "name": "USER_ID",
          "type": "string"
      },
      {
          "name": "AGE",
          "type": "int"
      },
      {
          "name": "GENDER",
          "type": "string",
          "categorical": true
      }
  ],
  "version": "1.0"
}
```

参照:[Recording Events](https://docs.aws.amazon.com/personalize/latest/dg/recording-events.html)

「Users」、「Items」データについては、「USER_ID」、「ITEM_ID」以外の文字列は全て「"categorical":true」と指定する必要がある。

### solution(personalizeの実行結果)

「学習済みのモデル」のことを「Amazon Personalize」では「solution」と呼ぶ。

### recipe

「recipe」は「solution」を作成するために利用される。

「recipe」は「アルゴリズム（ハイパーパラメータも含む）、「特徴量の変換（前処理方法）」の2点からなる。

### Campaign

「solution（学習済モデル）」を、実際にAPIとして利用できるようにデプロイする必要がある。

「Amazon Personalize」では、「Campaign」を作成することで「solution」をデプロイすることになります。レコメンド実行[API](http://d.hatena.ne.jp/keyword/API)のエンドポイント

参照 : https://blog.capilano-fw.com/?p=8807

https://qiita.com/ikegam1/items/22ee984313808e91d64f

<img width="620" alt="スクリーンショット 2024-01-17 18 49 05" src="https://github.com/Ryo-0912/AWS/assets/82032550/9f54f3b1-82db-4d1f-8f7b-80a7b7fe267d">



レコメンドが実行される手順

1. **事前準備1: RoleやS3バケットを作成する**

1. **事前準備2: 使うデータをダウンロードする**

1. **データセットグループを作成する**

      **データセットグループ** =>  Users(任意), Items(任意), Interactions(必須)。これら全て**schema設定必須**。

| Users | レコメンドを受ける対象 ユーザ。「最低限指定する必要があるカラム」、「事前予約されたカラム」以外のカラムは1〜5カラムまで利用することができる。 |
| --- | --- |
| Items | レコメンドの対象となるデータ(マスタデータ) |
| Interactions | User が Item に対して実施した過去の行動データ。「最低1,000レコード以上」、「2回以上出現するユーザーが25人以上」といった最低要件がある点に注意 |

1. **データセットに学習データのインポートを行う(以下3STEP)**

       1.   データセットのSchemaを作成する(データセットにデータを読み込むにはschemaを作成する必要あり)

       2.   **データをSchemaに合わせて加工し**、S3にアップロードする

       3.   データセットを作成し、S3からデータセットに読みこむjobを作成する

　　　  ⚠️**このとき、personalizeがbucketにアクセスできるように権限を付与しないといけない**

              ⇒ 以下の画像の「ブロックパブリックアクセス」でブロック解除し、参照先のバケットへpersonalizeがアクセスできるようロールを編集する。

<img width="629" alt="スクリーンショット 2024-01-17 18 50 09" src="https://github.com/Ryo-0912/AWS/assets/82032550/d9fc4484-75f4-4d2e-b1c4-1711d9d8a623">


ロールの設定

```jsx
{
    "Version": "2012-10-17",
    "Id": "PersonalizeS3BucketAccessPolicy",
    "Statement": [
        {
            "Sid": "PersonalizeS3BucketAccessPolicy",
            "Effect": "Allow",
            "Principal": {
                "Service": "personalize.amazonaws.com"
            },
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::ando-s3-bucket",
                "arn:aws:s3:::ando-s3-bucket/*"
            ]
        }
    ]
```

1. **ソリューション(=モデルを学習させるパート)を作成する**

　   「ソリューションのバージョンを作成する = モデルを学習させる」ことにあたる

         ⚠️**ソリューションの作成には時間がかかる(ソリューションを作成しないとキャンペーンの作成はできない)**

<img width="635" alt="スクリーンショット 2024-01-17 18 50 53" src="https://github.com/Ryo-0912/AWS/assets/82032550/d5074f61-53d8-4447-92e4-c033f65873ff">


1. **キャンペーンを作成する(これを行うことで、ソリューションをデプロイできる)**

<img width="637" alt="スクリーンショット 2024-01-17 18 51 10" src="https://github.com/Ryo-0912/AWS/assets/82032550/0853c12d-b72e-45d5-99b4-1da5d65a0409">


**Solution** : 5で作成したソリューション。5で作成できていない状態で選択してもキャンペーンの作成時にエラー出る。

**Minimum provisioned transactions per second** : 1秒間に最低でも何回レコメンドを実施するか。デフォルト値でいいかと。

1. **レコメンドを行う**

キャンペーン一覧からキャンペーン詳細を開くと次のような画面が表示される。

<img width="628" alt="スクリーンショット 2024-01-17 18 51 35" src="https://github.com/Ryo-0912/AWS/assets/82032550/76f0465d-4f4a-4bdb-b84c-f24ae26248e9">

<img width="635" alt="スクリーンショット 2024-01-17 18 51 50" src="https://github.com/Ryo-0912/AWS/assets/82032550/bc9c6324-3212-4937-953d-4fadc0915adb">

Item IDを任意の数値を入れて、「Get Recommendations」をクリックすると、レコメンド結果を25データ返してくれる。
