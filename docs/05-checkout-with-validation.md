# チェックアウトとデータ検証

`BlazingPizza.Shared` プロジェクトの `Order` クラスには `Address` 型 `DeliveryAddress` プロパティがあります。現在の実装ではアドレスを指定する機能がないため、全ての注文は住所が無い状態で作成されています。

このセッションでは住所を入力するための "チェックアウト" 画面の実装と住所データの検証を行います。

## チェックアウト画面の実装

まずは `Checkout.razor` を追加して、`@page` ディレクティブに `/checkout` を指定します。始めに `OrderReview` コンポーネントを再利用して注文の詳細を表示します:

```html
<div class="main">
    <div class="checkout-cols">
        <div class="checkout-order-details">
            <h4>Review order</h4>
            <OrderReview Order="OrderState.Order" />
        </div>
    </div>

    <button class="checkout-button btn btn-warning" @onclick="PlaceOrder">
        Place order
    </button>
</div>
```

`PlaceOrder` を `Index.razor` から `Checkout.razor` に移します:

```csharp
@code {
    async Task PlaceOrder()
    {
        var newOrderId = await HttpClient.PostJsonAsync<int>("orders", OrderState.Order);
        OrderState.ResetOrder();
        NavigationManager.NavigateTo($"myorders/{newOrderId}");
    }
}
```

これまで通り、`OrderState`、`HttpClient` および `NavigationManager` を使うために、`@inject` ディレクティブも追加してください。

次に注文した際に、チェックアウト画面に遷移させます。`Index.razor` で `PlaceOrder` メソッドを消したことを確認してから、注文ボタンをアンカータグに書き換え、リンク先を `/checkout` にします:

```html
<a href="checkout" class="@(Order.Pizzas.Count == 0 ? "btn btn-warning disabled" : "btn btn-warning")">
    Order >
</a>
```

アンカータグには `disabled` 属性が無いため、代わりに CSS で制御しています。

アプリを実行して注文を行うと、チェックアウトページが表示されるはずです。

![Confirm order](https://user-images.githubusercontent.com/1874516/77242251-d2530780-6bb9-11ea-8535-1c41decf3fcc.png)

## 住所の取得

チェックアウト画面はユーザーに住所を入れてもらうには良い場所です。アドレス入力は再利用の機会もありそうなので、再利用可能なコンポーネントとして実装します。
`BlazingPizza.Client` プロジェクトの `Shared` フォルダーに `AddressEditor.razor` を追加します。`Address` オブジェクトの編集ができるよう、パラメーターとして定義します:

```csharp
@code {
    [Parameter] public Address Address { get; set; }
}
```

マークアップでは `Address` クラスの各フィールドに対応するインプットを作成します:

```html
<div class="form-field">
    <label>Name:</label>
    <div>
        <input @bind="Address.Name" />
    </div>
</div>

<div class="form-field">
    <label>Line 1:</label>
    <div>
        <input @bind="Address.Line1" />
    </div>
</div>

<div class="form-field">
    <label>Line 2:</label>
    <div>
        <input @bind="Address.Line2" />
    </div>
</div>

<div class="form-field">
    <label>City:</label>
    <div>
        <input @bind="Address.City" />
    </div>
</div>

<div class="form-field">
    <label>Region:</label>
    <div>
        <input @bind="Address.Region" />
    </div>
</div>

<div class="form-field">
    <label>Postal code:</label>
    <div>
        <input @bind="Address.PostalCode" />
    </div>
</div>

@code {
    [Parameter] public Address Address { get; set; }
}
```

最後に `AddressEditor` コンポーネントを `Checkout.razor` に定義します:

```html
<div class="checkout-cols">
    <div class="checkout-order-details">
        ... leave this div unchanged ...
    </div>

    <div class="checkout-delivery-address">
        <h4>Deliver to...</h4>
        <AddressEditor Address="OrderState.Order.DeliveryAddress" />
    </div>
</div>
```

これでチェックアウト画面で住所の入力ができるようになりました。

![Address editor](https://user-images.githubusercontent.com/1874516/77242320-79d03a00-6bba-11ea-9e40-4bf747d4dcdc.png)

注文を保存すると、チェックアウト画面で入力した住所は `Order` オブジェクトの一部としてシリアライズされ、バックエンドで保存されます。これはチェックアウト画面に渡した `Address` オブジェクトが `OrderState.Order.DeliveryAddress` のためです。

もしデータベースの中身を直接見たい場合は、[DB Browser for SQLite](https://sqlitebrowser.org/) などの SQLite のデータベースを開けるツールを使い、 `pizza.db` ファイルを見てください。

もしくは、`BlazingPizza.Server` プロジェクトの `OrderController.PlaceOrder` メソッドにブレークポイントを置き、デバッグしてもデータは確認できます。

## サーバーサイドでのデータ検証

まだ住所データは一切検証していないため、値がブランクのままでも保存が可能です。検証をアプリケーションに追加する場合、通常はサーバーサイド/クライアントサイドの両方に実装します。

 * クライアントサイドの検証は即座に画面に出せるため、UX が向上します。しかし簡単にバイパスする事もできるため、データの保証にはなりません。
 * サーバーサイドの検証は強制力があります。

このため、クライアント側で何があっても検証が行えるように、サーバーサイドから実装することが多いです。`BlazingPizza.Server` プロジェクトの `OrdersController.cs` を確認すると、`[ApiController]` 属性が設定されています:

```csharp
[Route("orders")]
[ApiController]
public class OrdersController : Controller
{
    // ...
}
```

`[ApiController]` 属性を指定することで様々な機能が利用できます。データの検証もその 1 つで、`DataAnnotations` をモデルに指定するだけで検証が機能します。詳細は[ASP.NET Core MVC および Razor Pages でのモデルの検証](https://docs.microsoft.com/ja-jp/aspnet/core/mvc/models/validation?view=aspnetcore-3.1) を参照してください。

`BlazingPizza.Shared` プロジェクトの `Address.cs` クラスを開き、`[Required]` 属性を各プロパティに指定します。尚、自動生成される `Id` と必須入力にしない `Line2` には指定しないでください。他には `[MaxLength]` 属性でデータの最大長を設定することもできます:


```csharp
using System.ComponentModel.DataAnnotations;

namespace BlazingPizza
{
    public class Address
    {
        public int Id { get; set; }

        [Required, MaxLength(100)]
        public string Name { get; set; }

        [Required, MaxLength(100)]
        public string Line1 { get; set; }

        [MaxLength(100)]
        public string Line2 { get; set; }

        [Required, MaxLength(50)]
        public string City { get; set; }

        [Required, MaxLength(20)]
        public string Region { get; set; }

        [Required, MaxLength(20)]
        public string PostalCode { get; set; }
    }
}
```

アプリを再起動して注文を試してください。チェックアウト画面で住所をブランクのまま登録すると、サーバーサイドで検証が失敗して先に進まなくなります。画面にエラーが出ないため分かりずらいですが、ブラウザの開発者ツールを開き、ネットワークタブを確認してください。HTTP 400 ("Bad Request") エラーが確認できます。

![Server validation](https://user-images.githubusercontent.com/1874516/77242384-067af800-6bbb-11ea-8dd0-74f457d15afd.png)

住所を入力すると、これまで通り注文が作成できます。

## クライアントサイドでのデータ検証

Blazor はフォームデータの検証機能を提供します。サーバーサイドで利用している `DataAnnotations` ルールをクライアントサイドでも利用していきます。

Blazor のフォームデータ検証は `EditContext` を使って実行されます。`EditContext` は編集中のデータを追跡しているため、どのフィールドが編集されたか、どんな値が入力されたか、またフィールドの値に問題がないかなど確認できます。様々なビルトインコンポーネントが `EditContext` を連動し、検証エラーのメッセージを表示したり、ユーザーが入力したデータを表示したり出来ます。

### EditForm の利用

`EditForm` はビルトインコンポーネントで、フォームコントロールとして機能します。コンパイルすると HTML の `<form>` タグを出力し、`EditContext` と紐づけます。`Checkout.razor` コンポーネントで `main` div 内で `EditForm` を指定します:

```html
<div class="main">
    <EditForm Model="OrderState.Order.DeliveryAddress">
        <div class="checkout-cols">
            ... leave unchanged ...
        </div>

        <button class="checkout-button btn btn-warning" @onclick="PlaceOrder">
            Place order
        </button>
    </EditForm>
</div>
```

尚、コンポーネント内で複数の `EditForm` を利用できますが、入れ子には出来ません。これは HTML の `<form>` が入れ子にできないためです。

`Model` 属性に `EditContext` でトラックするものを指定します。ここでは `OrderState.Order.DeliveryAddress` を指定しているため、`Address` クラスの定義を使って追跡されます。

あまり見た目はよくありませんが、検証結果を基本的な機能を使って表示します。`EditForm` 内の一番下に以下のコンポーネントを追加します:

```html
<DataAnnotationsValidator />
<ValidationSummary />
```

`DataAnnotationsValidator` コンポーネントは `EditContext` と連携して `DataAnnotations` のルールを実行します。`DataAnnotations` の機能が十分でない場合は、`DataAnnotationsValidator` を別のものに変更してください。

`ValidationSummary` コンポーネントは `EditContext` の検証メッセージを HTML の `<ul>` としてリスト表示します。

### フォーム登録のハンドリング

この状態では、住所がブランクのままでも登録ボタンがクリックできます。これは `<button>` が明示的に `submit` ボタンであると指定されていないためです。`button` に `type="submit"` を追加して、`@onclick` 属性を削除します。

次に `PlaceOrder` をボタンから実行する代わりに、`EditForm` から実行するように変更します。`OnValidSubmit` 属性を `EditForm` に追加してください:

```html
<EditForm Model="OrderState.Order.DeliveryAddress" OnValidSubmit="PlaceOrder">
```

これで `<button>` は `PlaceOrder` メソッドを直接実行せず、フォームの登録のみを行います。その後 `EditForm` がデータの検証を行った後、`PlaceOrder` を実行するか判定します。アプリを実行して動作を確認してみてください。

![Validation summary](https://user-images.githubusercontent.com/1874516/77242430-9d47b480-6bbb-11ea-96ef-8865468375fb.png)

### ValidationMessage でメッセージをカスタマイズ

フォーム下部にまとめてエラーを表示しても UX は向上しないため、それぞれのインプットの近くにメッセージを移します。

まず `<ValidationSummary>` コンポーネントを削除します。そして `AddressEditor.razor` で `<ValidationMessage>` コンポーネントをそれぞれのフィールドに追加します。

```html
<div class="form-field">
    <label>Name:</label>
    <div>
        <input @bind="Address.Name" />
        <ValidationMessage For="@(() => Address.Name)" />
    </div>
</div>
```

全てのフィールドで同じ作業を繰り返します。尚、`@(() => Address.Name)` のラムダ式で、どのプロパティを検証するかを指定しています。

アプリを起動して、実際の動作を確認します。だいぶ分かりやすくなりました。

![Validation messages](https://user-images.githubusercontent.com/1874516/77242484-03ccd280-6bbc-11ea-8dd1-5d723b043ee2.png)

メッセージの変更も簡単です。例えば *The City field is required* の表示を変更する場合、`Address.cs` を書き換えます:

```csharp
[Required(ErrorMessage = "How do you expect to receive the pizza if we don't even know what city you're in?"), MaxLength(50)]
public string City { get; set; }
```

### ビルトインコンポーネントの検証機能で UX を向上する

検証結果の表示場所は良くなりましたが、エラーが出た場合、ユーザーが入力を変更しても、すぐにエラーは消えません。この問題を解消する為に HTML の input エレメントを Blazor のビルトインコンポーネントに差し替えます。 これらのコンポーネントはより柔軟に `EditContext` を連携します。

* 値が編集された場合、すぐに `EditContext` に通知して検証結果を変更できる。
* `EditContext` から検証結果を受け取れるため、値が有効か無効かを判断して画面を変更できる。

再度 `AddressEditor.razor` に戻って、`<input>` エレメントを `<InputText>` に変更します:

```html
<div class="form-field">
    <label>Name:</label>
    <div>
        <InputText @bind-Value="Address.Name" />
        <ValidationMessage For="@(() => Address.Name)" />
    </div>
</div>
```

同じ変更を他のプロパティにも行います。これでユーザーが入力を変更すると即座に UI が変更されるようになりました。また CSS で検証結果が分かるようにハイライトを変えています。

![Input components](https://user-images.githubusercontent.com/1874516/77242542-ba30b780-6bbc-11ea-8018-be022d6cac0b.png)

ハイライトの方法を変えたい場合は CSS を変更してみてください。

`InputText` 以外にも、`InputCheckbox`、`InputDate`、`InputSelect` などのビルトインコンポーネントが提供されています。詳細は前述した [ASP.NET Core Blazor のフォームと検証](https://docs.microsoft.com/ja-jp/aspnet/core/blazor/forms-validation?view=aspnetcore-3.1) を参照してください。

次のセッションは [認証と認可](06-authentication-and-authorization.md) です。
