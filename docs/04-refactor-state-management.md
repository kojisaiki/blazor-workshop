# ステート管理のリファクタリング

このセッションでは、既に実装したコードを振り返り、リファクタリングしていきます。

## 現在の問題

既に気づいているかもしれませんが、このアプリには不具合があります。注文の情報を `index` コンポーネントで保持しているため、ユーザーが画面遷移をすると、注文データが失われます。事象を確認するために、ピザを選択して注文カゴにいれた状態で、`My Orders` ページへ遷移して戻ってみてください。カゴに入れたデータが消失していることが分かります。

## 解決策

この問題は *アプリケーションステート パターン* と呼ばれる手法で解決できます。アプリケーションステートを保持するオブジェクトを別に用意して、DI 経由で利用することで、コンポーネントからステート管理を切り離せます。これにより複数コンポーネントを跨いだアプリケーションステート管理を可能にします。さらにこのパターンを使うことで、UI レイヤーとアプリケーションステートに関わるビジネスロジックを分割できます。

## ステート管理の実装

クライアントプロジェクトのルートに `OrderState` クラスを追加して、スコープサービスとして DI に登録します。Blazor WebAssembly アプリケーションでは、`Program` クラスの `Main` メソッドで IoC を構成します:

```csharp
public static async Task Main(string[] args)
{
    var builder = WebAssemblyHostBuilder.CreateDefault(args);
    builder.RootComponents.Add<App>("app");

    builder.Services.AddBaseAddressHttpClient();
    builder.Services.AddScoped<OrderState>();

    await builder.Build().RunAsync();
}
```

> メモ: 今回 *Singleton* ではなく *Scoped* でサービスを登録した理由は、サーバーサイドのコンポーネントと合わせるためです。シングルトンは *全ユーザー向け* の場合に利用し、スコープは *現在の作業* 単位となります。ただし、現時点で WebAssembly の *Scoped* は *Singleton* と同じ振る舞いをします。Blazor サーバーアプリの場合は *Scoped* は接続単位で動作します。詳細は [サービスの有効期間](https://docs.microsoft.com/ja-jp/aspnet/core/blazor/dependency-injection?view=aspnetcore-3.1#service-lifetime)を参照してください。

## Index の更新

DI への登録が完了したので、`@inject` を使って `Index` ページでステートオブジェクトを利用します。

```razor
@page "/"
@inject HttpClient HttpClient
@inject OrderState OrderState
@inject NavigationManager NavigationManager
```

以前にも紹介しましたが、 `@inject` ディレクティブでは型とプロパティ名を指定できます。

登録されていない型を挿入しようとすると例外が発生して、アプリケーションの起動に失敗します。

次に `OrderState` に注文のステータスや追加されたピザを管理するプロパティとメソッドを追加します。

`configuringPizza`、`showingConfigureDialog` および `order` フィールドを `OrderState` のクラスプロパティとして移動します。`private set` を指定して、内部メソッドからのみ値を操作できるようにしています:

```csharp
public class OrderState
{
    public bool ShowingConfigureDialog { get; private set; }

    public Pizza ConfiguringPizza { get; private set; }

    public Order Order { get; private set; } = new Order();
}
```

次に `Index` から `OrderState` にメソッドを移動します。`PlaceOrder` メソッドは画面遷移の機能があるため残しますが、代わりに`ResetOrder` メソッドを追加しました:

```csharp
public void ShowConfigurePizzaDialog(PizzaSpecial special)
{
    ConfiguringPizza = new Pizza()
    {
        Special = special,
        SpecialId = special.Id,
        Size = Pizza.DefaultSize,
        Toppings = new List<PizzaTopping>(),
    };

    ShowingConfigureDialog = true;
}

public void CancelConfigurePizzaDialog()
{
    ConfiguringPizza = null;

    ShowingConfigureDialog = false;
}

public void ConfirmConfigurePizzaDialog()
{
    Order.Pizzas.Add(ConfiguringPizza);
    ConfiguringPizza = null;

    ShowingConfigureDialog = false;
}

public void ResetOrder()
{
    Order = new Order();
}

public void RemoveConfiguredPizza(Pizza pizza)
{
    Order.Pizzas.Remove(pizza);
}
```

`order`、`configuringPizza` および `showingConfigureDialog` をはじめ移動したメソッドも `Index.razor` から消し忘れないように注意してください。これらのプロパティやメソッドは全て `OrderState` のものを利用します。

`Index` コンポーネントにある古い参照を、`OrderState` のものに差し替えるとコンパイルも通ります。例えば、`Index.razor` に残した `PlaceOrder` メソッドは以下のように変更します:

```csharp
async Task PlaceOrder()
{
    var newOrderId = await HttpClient.PostJsonAsync<int>("orders", OrderState.Order);
    OrderState.ResetOrder();
    NavigationManager.NavigateTo($"myorders/{newOrderId}");
}
```

`OrderState.Order` と書くのが長いと感じる場合は、プロパティを作成することもできます。

```csharp
Order Order => OrderState.Order;
```

コードを変更した後でアプリケーションを起動して、全て機能するか確認してください。特に既知の不具合が解消されているかは、再現手順を実施して、注文データが失われないか確認してください。

## ステート変更

この機会に、Blazor でどのようにステートが変わるかと、`EventCallback` が解決する問題を見ていきます。

通常 `EventCallback` は、イベントハンドラーを定義しているコンポーネントに、イベントハンドラーの実行と画面の再描画を行うよう通知をします。今回はイベントハンドラーが Blazor コンポーネントではない `OrderState` に定義されている為、イベントハンドラーとイベントを紐づけしている `index` コンポーネントに通知が行われます。

## 結論

*アプリケーションステート パターン* の採用で実施したことをまとめました:
- アプリケーションで共通のステートを `OrderState` に移動
- ステートの変更が実行されるメソッドがコンポーネント経由で実行
- `EventCallback` が適切なコンポーネントに変更通知を実施

このセッションでは以下のように多くの内容をカバーしました:
- パラメーターが変わった時やイベントを受け取った際のコンポーネントの再描画処理
- イベントハンドラーの実行に伴う、イベントのディスパッチの動作
- `EventCallback` を使う事で、容易にイベントのディスパッチが行えること

次のセッションは - [チェックアウトとデータ検証](05-checkout-with-validation.md) です。
