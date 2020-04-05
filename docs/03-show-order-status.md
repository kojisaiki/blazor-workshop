# 注文ステータスの表示

ユーザーはピザを注文できますが、注文のステータスが確認できません。このセッションでは "My orders" ページを追加して複数の注文を確認できるようにします。また "Order details" で注文の詳細も表示できるようにします。

## ナビゲーションリンクの追加

`Shared/MainLayout.razor` を開きます。実験として HTML の `<a>` タグをメニューに追加して `myorders` へのリンクを作成します:

```html
<div class="top-bar">
    (leave existing content in place)

    <a href="myorders" class="nav-tab">
        <img src="img/bike.svg" />
        <div>My Orders</div>
    </a>
</div>
```

> ここではリンクのアドレスが `/` から始まっていません。もし `/myorders` にリンクした場合、一見同じ動作をしますが、将来的にアプリをルートではない場所に配置した場合、リンクをすべて直す必要があります。Blazor は `index.html` にある `<base href="/">` でベースのアドレスを指定でき、`/` から始まらないアドレスは、このベースが補完されます。

アプリを起動してメニューを確認します。

![My orders link](https://user-images.githubusercontent.com/1874516/77241321-a03ba880-6bad-11ea-9a46-c73be397cb5e.png)

現時点では `<NavLink>` を使いたい理由はまだ分かりませんが、後で詳細を見ていきます。

## "My Orders" ページの追加

"My Orders" リンクをクリックすると "Page not found" が表示されます。これは当然、`myorders` のアドレスに一致するページが無いためですが、動作をよく見るとクライアントサイドのナビゲーションだけでなく、画面全体のリロードが発生していることが分かります。

以下に何が起きているかの詳細を説明します。

1. `myorders` リンクをクリック
2. Blazor はクライアントサイドのコンポーネントで `@page` ディレクティブに一致するアドレスがあるか確認
3. 見つからない場合、サーバーサイドのページであると推測してページ全体をリロード
4. サーバー側でも該当のページが無い場合、Blazor はフォールバックオプションに従って画面を描画
5. 今回の場合、クライアントサイドにもサーバーサイドにも一致するページが無いため、`App.razor` に定義されている `NotFound` を描画

つまり、必要に応じて `App.razor` の `NotFound` 内容を変更することでエラー画面を変更できます。

では実際にページを追加します。`MyOrders.razor` を `Pages` フォルダに追加し、`@page` ディレクティブにパスを定義します:

```html
@page "/myorders"

<div class="main">
    My orders will go here
</div>
```

アプリを起動して動作を確認します。

![My orders blank page](https://user-images.githubusercontent.com/1874516/77241343-fc9ec800-6bad-11ea-8176-febf614ed4ad.png)

今回はクライアントサイドでページが見つかるため、SPA 内でナビゲーションが完結し、ページ全体がリロードされることはありません。

## ナビゲーションメニューのハイライト

メニューバーをよく見ると、"My orders" を選択してもハイライト表示がされていません、`<a>` タグの代わりに `<NavLink>` を使いたい理由はここにあります。`<NavLink>` は指定されているリンクとアプリのアドレスが一致した場合、`active` クラスを自動で追加するため、CSS でのスタイリングが容易になります。

`MainLayout` にある `<a>` タグを `<NavLink>` に変更します:

```html
<NavLink href="myorders" class="nav-tab">
    <img src="img/bike.svg" />
    <div>My Orders</div>
</NavLink>
```

アプリを起動してハイライトを確認します。スタイリングは site.css の active を参照してください。

![My orders nav link](https://user-images.githubusercontent.com/1874516/77241358-412a6380-6bae-11ea-88da-424434d34393.png)

## 注文の一覧表示

`MyOrders` に戻り、`HttpClient` を DI 経由で取得してバックエンド API を呼び出します。`@page` ディレクティブの下に以下コードを追加します。:

```html
@inject HttpClient HttpClient
```

続いて `@code` に注文一覧を保持するプロパティを追加します:

```csharp
@code {
    List<OrderWithStatus> ordersWithStatus;

    protected override async Task OnParametersSetAsync()
    {
        ordersWithStatus = await HttpClient.GetJsonAsync<List<OrderWithStatus>>("orders");
    }
}
```

UI は 3 つの場合で表示を切り替えます

 1. データのロード中
 2. 注文が無い場合
 3. 注文がある場合

Razor の `@if/else` を使って場合分けを行います:

```html
<div class="main">
    @if (ordersWithStatus == null)
    {
        <text>Loading...</text>
    }
    else if (ordersWithStatus.Count == 0)
    {
        <h2>No orders placed</h2>
        <a class="btn btn-success" href="">Order some pizza</a>
    }
    else
    {
        <text>TODO: show orders</text>
    }
</div>
```
何か所は新しい記述があるため以下で説明します。

### 1. `<text>` エレメント

`<text>` は HTML エレメントではなく、Blazor のコンポーネントでもありません。実際コンパイルされた際には削除されます。`<text>` は Razor コンパイラに対して、そのブロックが C# コードではなく文字列であることを明示的に指定します。シンタックス上コードが文字列が不明瞭な場合に指定できます。

### 2. href="" の意味

通常は `<a href="">` のように記述することはありませんが、先述した通り、Blazor では index.html の `<base href="/">` をベースアドレスとして、`/` を持たないアドレスに自動で付与する機能があります。

### 3. どのように描画されるか

コンポーネントで非同期処理がある場合、ページの読み込み直後に一度画面は描画され、非同期処理が終わった際に再描画されます。

### 4. データのリセット方法

データベースのデータを削除して、注文が無い状態を試したい場合は、バックエンドプロジェクトにある `pizza.db` を削除してください。

![My orders empty list](https://user-images.githubusercontent.com/1874516/77241390-a4b49100-6bae-11ea-8dd4-e59afdd8f710.png)

## 注文をグリッドで表示

データの取得が完了したため、HTML グリッドに結果を表示します。

`<text>TODO: show orders</text>` を以下のコードに変更します:

```html
<div class="list-group orders-list">
    @foreach (var item in ordersWithStatus)
    {
        <div class="list-group-item">
            <div class="col">
                <h5>@item.Order.CreatedTime.ToLongDateString()</h5>
                Items:
                <strong>@item.Order.Pizzas.Count()</strong>;
                Total price:
                <strong>£@item.Order.GetFormattedTotalPrice()</strong>
            </div>
            <div class="col">
                Status: <strong>@item.StatusText</strong>
            </div>
            <div class="col flex-grow-0">
                <a href="myorders/@item.Order.OrderId" class="btn btn-success">
                    Track &gt;
                </a>
            </div>
        </div>
    }
</div>
```

多くのコードがあるように見えますが、目新しいものはありません。`ordersWithStatus` を `@foreach` でループ処理して、`<div>` に結果を表示しているだけです。アプリを起動して結果を確認します。

![My orders grid](https://user-images.githubusercontent.com/1874516/77241415-feb55680-6bae-11ea-89ba-f8367ef6a96c.png)

## 注文詳細画面

注文一覧グリッドで `Track` をクリックすると、`myorders/<id>` (例 `http://example.com/myorders/37`) にナビゲートされます。現在は一致するアドレスが無いため、"Page not found" が表示されます。

注文詳細用のページコンポーネント追加していきます。`Pages` フォルダに `OrderDetails.razor` を追加します:

```html
@page "/myorders/{orderId:int}"

<div class="main">
    TODO: Show details for order @OrderId
</div>

@code {
    [Parameter] public int OrderId { get; set; }
}
```

`@page` ディレクティブを上記のように記述すると、パラメーターを受け取れます。パスで指定したプロパティは `@code` ブロックにも同じ名前で追加してください。`{parameterName}` の名前は大文字小文字を区別しません。パラメータを数値型で受け取りたい場合は、制約として `{parameterName:int}` と記述できます。`:int` は数値型を制約としますが、他にも様々な制約を指定できます。詳細は[ルート制約](https://docs.microsoft.com/ja-jp/aspnet/core/blazor/routing?view=aspnetcore-3.1#route-constraints)を参照してください。

![Order details empty](https://user-images.githubusercontent.com/1874516/77241434-391ef380-6baf-11ea-9803-9e7e65a4ea2b.png)

ルーティングは以下の順番で機能します。

1. アプリ起動時に `Startup.cs` よりルートコンポーネントが `App` であることを確認する。
2. `App.razor` に定義した `App` コンポーネントに `<Router>` があることを確認。`Router` はビルトインコンポーネントの 1 つでブラウザーのナビゲーション API と連携してクライアントサイドのナビゲーションを制御する。
3. `Router` は `@page` に定義されている URL　のパターンを確認して `{parameter}` トークンよりパラメーターとして必要な値いを渡す。また `:int` のように制約がある場合は型もチェックする。
   * マッチするコンポーネントがあると `Router` がコンポーネントを読み込む
   * マッチするコンポーネントが無い場合はページをフルロードしてサーバーサイドで処理をする
   * サーバーが Blazor アプリの再起動を行った場合は、Blazor 側はサーバーサイドにも一致するページが存在しなかったと判断して`NotFound` を描画する。

## データのポーリング

`OrderDetails` のデータ取得ロジックは `MyOrders` とは異なります。コンポーネントの初期化時にデータを一度だけ取得するのではなく、定期的に最新の情報を取得できるよう、サーバーをポーリングして、画面を更新します。これによりほぼリアルタイムで注文のステータスや配達状況を画面に表示できます。

さらに `OrderId` についても有効性を検証します。これは以下のシナリオを想定しているためです。

* 一致する注文が無い場合
* 後ほど認証を実装した場合、そのオーダーがログインしているユーザーのものでは無い場合

ポーリングを実装する前に、`OrderDetails.razor` の `@page` ディレクティブの下に以下コードを追加します:

```html
@using System.Threading
@inject HttpClient HttpClient
```

`@inject HttpClient HttpClient` はすでに使った事がりますが、`@using` ディレクティブは始めて出てきます。これは C# のコードと同じく名前空間を読み込む機能です。Visual Studio はまだ `@using` を自動で追加しないため、手動で追加する必要があります。

では `@code` にポーリングのロジックを追加します:

```csharp
@code {
    [Parameter] public int OrderId { get; set; }

    OrderWithStatus orderWithStatus;
    bool invalidOrder;
    CancellationTokenSource pollingCancellationToken;

    protected override void OnParametersSet()
    {
        // If we were already polling for a different order, stop doing so
        pollingCancellationToken?.Cancel();

        // Start a new poll loop
        PollForUpdates();
    }

    private async void PollForUpdates()
    {
        pollingCancellationToken = new CancellationTokenSource();
        while (!pollingCancellationToken.IsCancellationRequested)
        {
            try
            {
                invalidOrder = false;
                orderWithStatus = await HttpClient.GetJsonAsync<OrderWithStatus>($"orders/{OrderId}");
            }
            catch (Exception ex)
            {
                invalidOrder = true;
                pollingCancellationToken.Cancel();
                Console.Error.WriteLine(ex);
            }

            StateHasChanged();

            await Task.Delay(4000);
        }
    }
}
```

少しコードが長いため、処理内容をしっかり確認してください。

* `OnInitialized` や `OnInitializedAsync` ではなく、`OnParametersSet` をオーバーライドしています。`OnParametersSet` はコンポーネントライフサイクルイベントの 1 つで、コンポーネントの初期化時と、パラメーターの値が変わる度に実行されます。ユーザーが `myorders/2` から `myorders/3` に直接移動した場合、フレームワークは最適化の機能のため `OrderDetails` インスタンスを破棄せず `OrderId` のみを変更します。
  * 上記シナリオを避けるため、アプリでは注文詳細から他の詳細へ移動するリンクは提供しませんが、いつ変更されるかは分からないため、適切なライフサイクルイベントを使用するよう心がけてください。
* `async void` メソッドでポーリングを実施しています。処理にエラーが発生した場合、呼び出し元は既に完了しているため `try/catch` を使って例外を投げるようにしてください。
* `CancellationTokenSource` を使ってポーリング処理を中断できるようにしています。現時点では例外が発生した場合にのみポーリングがキャンセルされます。
* `StateHasChanged` メソッドを実行することで Blazor アプリで再描画を実行します。他の方法ではフレームワークはいつ画面を描画するべきか分かりません。

## 注文詳細の表示

注文詳細データをポーリングで取得できるようになったため、次は画面を描画します。`<div class="main">` を以下のマークアップに変更します:

```html
<div class="main">
    @if (invalidOrder)
    {
        <h2>Nope</h2>
        <p>Sorry, this order could not be loaded.</p>
    }
    else if (orderWithStatus == null)
    {
        <text>Loading...</text>
    }
    else
    {
        <div class="track-order">
            <div class="track-order-title">
                <h2>
                    Order placed @orderWithStatus.Order.CreatedTime.ToLongDateString()
                </h2>
                <p class="ml-auto mb-0">
                    Status: <strong>@orderWithStatus.StatusText</strong>
                </p>
            </div>
            <div class="track-order-body">
                TODO: show more details
            </div>
        </div>
    }
</div>
```

このマークアップで 3 種類の内容を切り替えます。

1. `OrderId` が無効な場合 (例: サーバーがエラーを返した)
2. データ読み込み中
3. 表示するデータがある場合

![Order details status](https://user-images.githubusercontent.com/1874516/77241460-a7fc4c80-6baf-11ea-80c1-3286374e9e29.png)


最後に注文のデータを表示しますが、ここでは再利用可能なコンポーネントを作成します。

`OrderReview.razor` を `Shared` フォルダに追加します。コンポーネントは `Order` パラメータを受け取り描画します:

```html
@foreach (var pizza in Order.Pizzas)
{
    <p>
        <strong>
            @(pizza.Size)"
            @pizza.Special.Name
            (£@pizza.GetFormattedTotalPrice())
        </strong>
    </p>

    <ul>
        @foreach (var topping in pizza.Toppings)
        {
            <li>+ @topping.Topping.Name</li>
        }
    </ul>
}

<p>
    <strong>
        Total price:
        £@Order.GetFormattedTotalPrice()
    </strong>
</p>

@code {
    [Parameter] public Order Order { get; set; }
}
```

`OrderDetails.razor` に戻り、`TODO: show more details` の代わりに `OrderReview` を表示します:

```html
<div class="track-order-body">
    <div class="track-order-details">
        <OrderReview Order="orderWithStatus.Order" />
    </div>
</div>
```

`track-order-details` クラスを持った `div` は CSS でのスタイリングに使います。

アプリを起動して動作を確認します。

![Order details](https://user-images.githubusercontent.com/1874516/77241512-2e189300-6bb0-11ea-9740-fe778e0ce622.png)


## 画面のリアルタイム更新を確認する

バックエンドサーバーには注文のステータス変更をシミュレーションする機能があります。リアルタイムに表示が変わることを確認するため、注文を行ったらすぐに注文詳細に移動します。

始めはステータスが *Preparing* ですが、その後 10-15 秒程度で *Out for delivery* に代わり、60 秒程度で *Delivered* となります。

## Dispose の実装

アプリをこのまま本番環境に展開すると、深刻な問題が発生します。`OrderDetails` のロジックはポーリングを開始しますが、終了することはありません。もしユーザーが異なる注文詳細を何度も表示すると、複数の `OrderDetails` インスタンスが作成され、その分ポーリングも増加しますが、最後に表示した UI に対してしかポーリングの意味がありません。

この挙動は以下の手順で確認できます。

1. "my orders" を開く。
2. "Track" をクリックしてして詳細画面へ遷移。
3. 戻るボタンで "my orders" に戻る。
4. 20 回ほど繰り返す。
5. ブラウザの開発者ツールより、ネットワークタブを開くと、詳細画面を開いた分だけの HTTP 通信が残っていることが確認できる

この問題を解決するには `OrderDetails` が画面から消えたタイミングでポーリングを停止します。これは `IDisposable` を継承することで実現可能です。

`OrderDetails.razor` に戻って以下のコードを追加します:

```html
@implements IDisposable
```

この時点でコンパイルするとエラーとなります。

```
error CS0535: 'OrderDetails' does not implement interface member 'IDisposable.Dispose()'
```

`@code` ブロックに Dispose を追加します:

```cs
void IDisposable.Dispose()
{
    pollingCancellationToken?.Cancel();
}
```

これでフレームワークが、コンポーネント破棄のタイミングで `Dispose` メソッドを実行してくれます。再度アプリを起動して、問題が解消したか確認します。

## 注文詳細へ自動でナビゲーションする

現在は注文を行った際、`Index` コンポーネントでステートがリセットされるだけであり、手動で注文一覧に移動する必要があります。これは良い UX とは言えないため改善しましょう。

`Index` コンポーネントに戻り、`NavigationManager` をインジェクトします:

```html
@inject NavigationManager NavigationManager
```

`NavigationManager` を使うと、ナビゲーションの実行ができる他、現在の URI を取得したりできます。詳細は [URI およびナビゲーション状態ヘルパー](https://docs.microsoft.com/ja-jp/aspnet/core/blazor/routing?view=aspnetcore-3.1#uri-and-navigation-state-helpers) を参照してください。

`PlaceOrder` メソッドに `NavigationManager.NavigateTo` を追加します:

```csharp
async Task PlaceOrder()
{
    var newOrderId = await HttpClient.PostJsonAsync<int>("orders", order);
    order = new Order();
    NavigationManager.NavigateTo($"myorders/{newOrderId}");
}
```

アプリを起動して、動作を確認します。

次のセッションは [ステート管理のリファクタリング](04-refactor-state-management.md) です。