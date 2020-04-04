# 発行と展開

このセッションでは、開発した Blazor アプリを Azure App Service に発行して、外部からも使えるようにします。

## Azure アカウントの作成

このセッションでは Azure アカウントが必要となります。お持ちでない場合、[無償利用版](https://azure.microsoft.com/Free) にサインアップしてください。

アカウントを取得したら、Visual Studio に同じアカウントでサインインし、Visual Studio 内から Azure にアクセスできるようにします。

## 新規 Azure App Service に発行

Azure App Service は ASP.NET Core アプリをクラウドで実行するサービスです。

バックエンドサーバープロジェクトを右クリックして、発行を選択します。このプロジェクトはクライアントプロジェクトをはじめ、必要なプロジェクトの参照を持つため、Blazor と依存関係も合わせて発行されます。

![Publish from VS](https://user-images.githubusercontent.com/1874516/51885818-2501ac80-2385-11e9-8025-4d1477083a8d.png)

"Pick a publish target" ダイアログで以下の操作をします。
- "App Service" を選択。
- "Create New" オプションを選択。
- "Create Profile" を選択してクリック。

![Pick a publish target](https://user-images.githubusercontent.com/1874516/51885912-7f027200-2385-11e9-8707-0e2f82b543fd.png)

"Create App Service" ダイアログで以下の操作をします。
- 正しい Azure アカウントが選択されていることを確認。
- ユニークな名前を指定。この名前は URL の一部となります。
- 使用する Azure サブスクリプションとリソースグループおよびホスティングプランを選択。
    - リソースグループは関連するリソースをまとめる便利な機能です。今回のアプリ専用のグループを作ってください。
    - ホスティングプランは無償のものが利用できます。

![Create App Service](https://user-images.githubusercontent.com/1874516/51886115-4e6f0800-2386-11e9-9da1-82cc910aad3b.png)

このタイミングで本番用のデータベースも作成できます。このアプリは SQLite を使っているため、別途データベースは必要ありませんが、実際のアプリでは必要となるでしょう。

作成をクリックます。この作業は数分かかります。App Service が作成されたら、発行ページでプロファイルが確認できます。

![Publish profile](https://user-images.githubusercontent.com/1874516/51886256-ee2c9600-2386-11e9-9da7-d80d2500b0ea.png)

実際に発行する前に、幾つか構成を行う必要があります。このアプリは Twitter 連携を行っているため、別途 Twitter 側でアプリを登録し、ID とシークレットを取得してください。開発中は `appsettings.Development.json` の定義を利用して動作していましたが、こちらは今回のワークショップ用に開発チームが取得したものです。

Twitter にアプリを登録するには、[Twitter 開発者コンソール](https://developer.twitter.com/apps) より開発者アカウントを取得します。

"Edit App Service Settings" リンクをクリックして、2 つの設定を追加します。 

- `Authentication:Twitter:ConsumerKey`
- `Authentication:Twitter:ConsumerSecret`

それぞれ正しい値を設定してください。

![Add app settings](https://user-images.githubusercontent.com/1874516/51886491-fc2ee680-2387-11e9-9c16-5f1fc47365fa.png)

準備が完了したら、発行を実行します。

![image](https://user-images.githubusercontent.com/1874516/51886932-a52a1100-2389-11e9-8b58-6ea3ae5a4291.png)

初めの発行は数分かかりますが、完了すると画面が表示されます。

![Published app](https://user-images.githubusercontent.com/1874516/51886593-5a5bc980-2388-11e9-9329-7e015901e45d.png)

以上でワークショップは全て完了です。お疲れ様でした。