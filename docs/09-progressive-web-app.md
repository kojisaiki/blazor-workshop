# プログレッシブ Web アプリ(PWA)

*プログレッシブ Web アプリ (PWA)* とはモダンブラウザの API を駆使し、ネイティブアプリのように動作するアプリケーションの事で、デスクトップやモバイル OS と連携して機能します:

 * OS のタスクバーやホームスクリーンにインストールされる
 * オフラインで機能する
 * プッシュ通知機能がある

Blazor もブラウザで動作するアプリのため、他のモダン Web フレームワークと同様、高度なブラウザの機能が利用できます。

## サービスワーカーの追加

PWA の重要な機能として、*サービスワーカー* があります。これは通常の JavaScript ファイルで、ブラウザがアプリケーションのコンテキスト外で実行できるイベントハンドラを提供します。このハンドラはプッシュ通知を受けた時などにトリガーすることができます。詳細については Google の [Service Worker の紹介](https://developers.google.com/web/fundamentals/primers/service-workers)を参照してください。

Blazor は .NET で開発しますが、サービスワーカーについてはアプリケーションのコンテキスト外のため、 JavaScript で実装します。技術的には Mono WebAssembly を使ってサービスワーカーを作り、.NET コードを実行することもできますが、開発工数に見合わないため通常は数行の JavaScript コードを使います。

サービスワーカーとして `wwwroot` に `service-worker.js` ファイルを作成して、以下のコードを貼り付けます:

```js
self.addEventListener('install', async event => {
    console.log('Installing service worker...');
    self.skipWaiting();
});

self.addEventListener('fetch', event => {
    // You can add custom logic here for controlling whether to use cached data if offline, etc.
    // The following line opts out, so requests go directly to the network as usual.
    return null;
});
```

上記コードは、サービスワーカーのインストールの実行と、他サービスがアプリケーションに対して `fetch` イベントを発生させた場合に応答するリスナーを作成します。必要に応じてオフラインサポート等も追加できますが、今のところ変更はしません。

次に `index.html` の `<script>` タグに以下のコードを追加します:

```html
<script>navigator.serviceWorker.register('service-worker.js');</script>
```

アプリを起動してブラウザーの開発者ツールを開くと、以下のメッセージがコンソールに表示されます。

```
Installing service worker...
```

インストールは `service-worker.js` ファイル変更後にページをロードした際に 1 度だけ実行されるため、再起動しても毎回インストールはされません。ファイルの一部を変更して再読み込みするなどして試してみてください。

この時点で機能はありませんが、全てのサービスワーカーはこのコードが必要となるため、テンプレートとして覚えておいてください。

## アプリをインストール可能にする

次に Blazor アプリを OS にインストールできるようにします。Windows/Mac/Linux OS の場合は Chrome/Edge ベータ版、iOS/Android は Safari/Chrome の機能を使うため、Firefox など他のブラウザでは機能しない可能性があります。

`wwwroot` 直下に `manifest.json` ファイルを作成して、以下設定を追加します:

```json
{
  "short_name": "Blazing Pizza",
  "name": "Blazing Pizza",
  "icons": [
    {
      "src": "img/icon-512.png",
      "type": "image/png",
      "sizes": "512x512"
    }
  ],
  "start_url": "/",
  "background_color": "#860000",
  "display": "standalone",
  "scope": "/",
  "theme_color": "#860000"
}
```

この情報で OS に対してアプリをインストールします。色や名前など好きに変えてください。

次にこのファイルのパスを `index.html` の `<head>` セクションで指定します:

```html
<link rel="manifest" href="manifest.json" />
```

この状態でアプリを起動すると、アドレスバーにインストール用の [+] ボタンが表示さえれるので、クリックするとインストールが行えます。

![image](https://user-images.githubusercontent.com/1101362/66352975-d1eee900-e958-11e9-9042-85ea4ac0c56b.png)

モバイルデバイスの場合は *Add to home screen* 機能から実行します。

インストールされたアプリは、スタンドアロンのネイティブアプリのように表示されます:

![image](https://user-images.githubusercontent.com/1101362/66356174-0024f680-e962-11e9-9218-3f1ca657a7a7.png)

Windows ユーザーの場合、インストールしたアプリはスタートメニューにも追加され、タスクバーにピンすることもできます。macOS でも似た機能が提供されます。

## プッシュ通知の送信

PWA の重要な機能として、バックエンドサーバーからの*プッシュ通知*があります。サービスワーカーが受け取るため、アプリを起動している必要はありません。

 * 重要なイベントがある場合にユーザーに通知してアプリを開いてもらう。
 * ユーザーが次回アプリを開いた際、オフラインであっても最新の情報が取得できる様にニュースフィードなどアプリ内のデータを更新する。

今回のアプリではピザの配達が始まったタイミングや配達のステータスが変わったタイミングで通知を行います。

### イベントのサブスクリプション

プッシュ通知を受け取る為にユーザーの同意を得る必要があります。ユーザーが同意するとブラウザはサブスクリプションを作成して通知を受け取れます。通常は *Send me updates* のようなボタンを用意しますが、今回は単純にするためにチェックアウト画面に移動したタイミングで同意の確認をします。

`Checkout.razor` の `OnInitializedAsync` メソッドに以下を追加します:

```csharp
// In the background, ask if they want to be notified about order updates
_ = RequestNotificationSubscriptionAsync();
```

次に `RequestNotificationSubscriptionAsync` を `@code` ブロックに追加します:

```csharp
async Task RequestNotificationSubscriptionAsync()
{
    var subscription = await JSRuntime.InvokeAsync<NotificationSubscription>("blazorPushNotifications.requestSubscription");
    if (subscription != null)
    {
        await HttpClient.PutJsonAsync<object>("notifications/subscribe", subscription);
    }
}
```

このコードでは `BlazingPizza.ComponentsLibrary/wwwroot/pushNotifications.js` にある `pushManager.subscribe` API 経由で結果を .NET 側に伝えます。

ユーザーがプッシュ通知に同意した場合、バックエンドにトークンが送信され、プッシュ通知時に使用されます。

アプリを起動して、注文を行ってみてください。

![image](https://user-images.githubusercontent.com/1101362/66354176-eed8eb80-e95b-11e9-9799-b4eba6410971.png)

開発者ツールのコンソールを開き、*Allow* を選択した際にエラーが出ないことを確認します。バックエンドでデバッグをしたい場合は、`NotificationsController` の `Subscribe` メソッドにブレークポイントを設定します。ブラウザより、エンドポイント URL やトークンが送られてくることが分かります。

同意してもしなくても、ユーザーに対するプロンプトは 1 度しか行われません。テストのため同意情報をリセットしたい場合、ブラウザの情報アイコン [!] をクリックして、通知を *確認する* に変更してください。
![image](https://user-images.githubusercontent.com/1101362/66354317-58f19080-e95c-11e9-8c24-dfa2d19b45f6.png)

### プッシュ通知の送信

プッシュ通知を行う際、セキュリティのために複雑な暗号化処理が行われます。しかしこのような処理は 3rd パーティーの NuGet パッケージが行ってくれます。

`BlazingPizza.Server` プロジェクトに `WebPush` Nuget を追加します。以下手順はバージョン `1.0.11` で検証したものです。

次に `OrdersController` を開き、`TrackAndSendNotificationsAsync` メソッドを確認します。このメソッドでは配送のステップをシミュレートし、`SendNotificationAsync` を実行します。

`SendNotificationAsync` は先ほど取得したサブスクリプションを利用して、通知を行います。次のコードは `WebPush` API を使ってプッシュ通知をおこなっています:

```csharp
private static async Task SendNotificationAsync(Order order, NotificationSubscription subscription, string message)
{
    // For a real application, generate your own
    var publicKey = "BLC8GOevpcpjQiLkO7JmVClQjycvTCYWm6Cq_a7wJZlstGTVZvwGFFHMYfXt6Njyvgx_GlXJeo5cSiZ1y4JOx1o";
    var privateKey = "OrubzSz3yWACscZXjFQrrtDwCKg-TGFuWhluQ2wLXDo";

    var pushSubscription = new PushSubscription(subscription.Url, subscription.P256dh, subscription.Auth);
    var vapidDetails = new VapidDetails("mailto:<someone@example.com>", publicKey, privateKey);
    var webPushClient = new WebPushClient();
    try
    {
        var payload = JsonSerializer.Serialize(new
        {
            message,
            url = $"myorders/{order.OrderId}",
        });
        await webPushClient.SendNotificationAsync(pushSubscription, payload, vapidDetails);
    }
    catch (Exception ex)
    {
        Console.Error.WriteLine("Error sending push notification: " + ex.Message);
    }
}
```

暗号キーはローカルでも https://tools.reactpwa.com/vapid のようなツールを使ってオンラインでも生成できます。キーを更新した場合は、`pushNotifications.js` のキーも併せて更新してください。また`someone@example.com` サンプルアドレスも変更してください。

サービスワーカーは通知を受け取りますが、処理を特に実施しないため、この状態でアプリを起動しても UI 上には変化がありません。

ブラウザーの開発者ツールを開くと、注文を行った 10 秒程度後に通知が届くことが確認できます。*Application* タブを開き、 *Push Messaging* セクションを確認してください。*Start recording* をクリックして記録を行えます。

![image](https://user-images.githubusercontent.com/1101362/66354962-690a6f80-e95e-11e9-9b2c-c254c36e49b4.png)

### 通知の表示

受け取った通知を `service-worker.js` で処理して表示できるよう、以下の関数を使いします:

```js
self.addEventListener('push', event => {
    const payload = event.data.json();
    event.waitUntil(
        self.registration.showNotification('Blazing Pizza', {
            body: payload.message,
            icon: 'img/icon-512.png',
            vibrate: [100, 50, 100],
            data: { url: payload.url }
        })
    );
});
```

`service-worker.js` を変更したため、次回ブラウザでアプリをロードした際、`Installing service worker...` の表示と共にサービスワーカーが更新されます。手動でアップデートする場合は開発者ツールの *アプリケーション* タブにある *サービスワーカー* より *更新* または *登録解除* から強制的に行います。

アプリを起動して注文を行うと、ステータスが *Out for delivery* になったタイミングで通知が表示されます。

![image](https://user-images.githubusercontent.com/1101362/66355395-0bc2ee00-e95f-11e9-898d-23be0a17829f.png)

Chrome または Edge ベータ版を使っている場合、アプリと異なるページを見ていても通知が来ますが、ブラウザが起動している必要があります。アプリをインストール場合は、アプリを起動していなくても通知が処理されます。

## 通知をクリックした際の処理

ユーザーが通知をクリックした際に、注文のページが開くようにします。バックエンドからは通知に `url` が送られているため、こちらのアドレスを開きます。`service-worker.js` に以下のコードを追加します:

```javascript
self.addEventListener('notificationclick', event => {
    event.notification.close();
    event.waitUntil(clients.openWindow(event.notification.data.url));
});
```

サービスワーカーが更新された後、再度通知を試します。今回は届いた通知をクリックすることで、注文画面が開きます。アプリケーションとしてインストールされている場合は、PWA が開きます。

このセッションでは Blazor が .NET だけでなく、モダンブラウザーや JavaScript の恩恵を全て受けられることを見てきました。通常の Web アプリケーションのように、常に最新バージョンを使えるメリットに加え、ネイティブアプリのように動作し、OS にインストールできます。

もし PWA をさらに突き詰めたい場合、是非オフラインサポートも試してください。基本的な動作は非常にシンプルです。[The Offline Cookbook](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook) には多くのサービスワーカーの例があります。今回のアプリはピザの注文用途のため、バックエンドなしではあまり意味がありませんでした。それでもオフライン時に注文を見れるなど幾つかのシナリオはあるでしょう。

次のセッションは - [発行と展開](10-publish-and-deploy.md) です。
