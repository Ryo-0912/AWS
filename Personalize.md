「Users」、「Items」、「interaction」データについてそれぞれ「schema」を指定する必要がある。

「schema」は、各データの定義情報です。

例えば、下記は「Users」データの「schema」(データ構造)の例です。

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

参照:**[Recording Events](https://docs.aws.amazon.com/personalize/latest/dg/recording-events.html)**

「Users」、「Items」データについては、「USER_ID」、「ITEM_ID」以外の文字列は全て「"categorical":true」と指定する必要があります。

### solution(personalizeの実行結果)

「学習済みのモデル」のことを「Amazon Personalize」では「solution」と呼んでいます。

### recipe

「recipe」は「solution」を作成するために利用される。

「recipe」は「アルゴリズム（ハイパーパラメータも含む）、「特徴量の変換（前処理方法）」の2点からなるようです。

### Campaign

「solution（学習済モデル）」を、実際にAPIとして利用できるようにデプロイする必要がある。

「Amazon Personalize」では、「Campaign」を作成することで「solution」をデプロイすることになります。レコメンド実行[API](http://d.hatena.ne.jp/keyword/API)のエンドポイント

参照 : https://blog.capilano-fw.com/?p=8807

https://qiita.com/ikegam1/items/22ee984313808e91d64f

![スクリーンショット 2024-01-12 12.03.24.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/3fa88b57-9dc5-4cee-b7c0-84521fe7acdf/15dbd981-b490-4927-bfcd-a9cb15bb739d/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2024-01-12_12.03.24.png)

レコメンドが実行される手順

1. **事前準備1: RoleやS3バケットを作成する**

1. **事前準備2: 使うデータをダウンロードする**

1. **データセットグループを作成する**

      **データセットグループ** =  Users(任意), Items(任意), Interactions(必須)。これら全て**schema設定必須**。

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

![スクリーンショット 2024-01-12 15.51.56.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/3fa88b57-9dc5-4cee-b7c0-84521fe7acdf/e4e1c86a-60a3-430a-b2a2-4b59d602e399/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2024-01-12_15.51.56.png)

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

![スクリーンショット 2024-01-17 17.06.07.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/3fa88b57-9dc5-4cee-b7c0-84521fe7acdf/39bf08e3-217c-41dc-b94b-2aacae8116f4/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2024-01-17_17.06.07.png)

1. **キャンペーンを作成する(これを行うことで、ソリューションをデプロイできる)**

![スクリーンショット 2024-01-17 17.08.20.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/3fa88b57-9dc5-4cee-b7c0-84521fe7acdf/2e845a05-17b6-4c79-a4a8-586e6f59556c/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2024-01-17_17.08.20.png)

**Solution** : 5で作成したソリューション。5で作成できていない状態で選択してもキャンペーンの作成時にエラー出る。

**Minimum provisioned transactions per second** : 1秒間に最低でも何回レコメンドを実施するか。デフォルト値でいいかと。

1. **レコメンドを行う**

キャンペーン一覧からキャンペーン詳細を開くと次のような画面が表示される。

![スクリーンショット 2024-01-17 17.26.54.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/3fa88b57-9dc5-4cee-b7c0-84521fe7acdf/4f031131-9329-4e59-8071-907497072048/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2024-01-17_17.26.54.png)

![スクリーンショット 2024-01-17 17.27.13.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/3fa88b57-9dc5-4cee-b7c0-84521fe7acdf/21e3c768-85a5-4a85-83a7-c34b4a8c37bb/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2024-01-17_17.27.13.png)

Item IDを任意の数値を入れて、「Get Recommendations」をクリックすると、レコメンド結果を25データ返してくれる。
