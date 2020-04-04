# ピザのカスタマイズ

このセッションでは、ユーザーが注文するピザのカスタマイズを行えるようにします。

## イベントハンドリング

ユーザーがピザを選択した場合、ピザのカスタマイズを行ってから注文カゴに入れれるようにします。Blazor アプリで クリックイベントなどの DOM イベントをハンドルするには、ハンドルしたいイベントの指定と、処理として呼ばれる C# のデリゲートが必要となります。

*Pages/Index.razor* を開き、`@onclick` をリストアイテムに追加します:

```html
@foreach (var special in specials)
{
    <li @onclick="@(() => Console.WriteLine(special.Name))" style="background-image: url('@special.ImageUrl')">
        <div class="pizza-info">
            <span class="title">@special.Name</span>
            @special.Description
            <span class="price">@special.GetFormattedBasePrice()</span>
        </div>
    </li>
}
```
アプリを実行して、ピザをクリックしてみます。画面上の反応はありませんが、ブラウザの開発者ツールを開くと、コンソールにクリックしたピザの名前が表示される様子が見て取れます。

![@onclick-event](https://user-images.githubusercontent.com/1874516/77239615-f56dbf00-6b99-11ea-8535-ddcc8bc0d8ae.png)


Razor ファイルでは `@` シンボルで始まり括弧で括られた範囲は、C# コードとして認識されます。 

次に *Index.razor* の `@code` ブロックを更新して、選択されたピザとカスタマイズダイアログを表示するか決めるブール値を作成します:

```csharp
List<PizzaSpecial> specials;
Pizza configuringPizza;
bool showingConfigureDialog;
```

続いて、ピザをクリックした際の処理として `ShowConfigurePizzaDialog` メソッドを追加します:

```csharp
void ShowConfigurePizzaDialog(PizzaSpecial special)
{
    configuringPizza = new Pizza()
    {
        Special = special,
        SpecialId = special.Id,
        Size = Pizza.DefaultSize,
        Toppings = new List<PizzaTopping>(),
    };

    showingConfigureDialog = true;
}
```

最後に `@onclick` ハンドラーで `Console.WriteLine` の代わりに、追加した `ShowConfigurePizzaDialog` を呼び出すよう変更します:

```html
<li @onclick="@(() => ShowConfigurePizzaDialog(special))" style="background-image: url('@special.ImageUrl')">
```

## ピザのカスタマイズダイアログの実装

ピザのカスタマイズダイアログは、ピザのサイズやピザに乗せるトッピングを指定できます。カスタマイズに応じた金額を表示し、注文カゴに追加する機能も必要です。

*ConfigurePizzaDialog.razor* ファイルを *Shared* フォルダに追加します。このコンポーネントはページではないため、ファイルの先頭に `@page` ディレクティブは必要ありません。このようなコンポーネントは今後も *Shared* フォルダに配置します。

> メモ: Visual Studio では、*Shared* フォルダを右クリックして、新しいアイテムより *Razor Component* を追加できます。

`ConfigurePizzaDialog` に選択されたピザを受け取る `Pizza` パラメーターを指定します。コンポーネントのパラメータプロパティには `[Parameter]` 属性を指定します。`@code` ブロックに `Pizza` を追加します:

```csharp
@code {
    [Parameter] public Pizza Pizza { get; set; }
}
```

> メモ: コンポーネントパラメータは、フレームワークで値をセットできるよう、`public` のセッターが必要です。しかしフレームワークだけが値をセットすべきで、コードから直接値を変更しないでください。コード内から値を変更した場合、描画される値とフレームワークで管理するステートの値の同期が出来なくなります。

`ConfigurePizzaDialog` に以下のマークアップを追加します:

```html
<div class="dialog-container">
    <div class="dialog">
        <div class="dialog-title">
            <h2>@Pizza.Special.Name</h2>
            @Pizza.Special.Description
        </div>
        <form class="dialog-body"></form>
        <div class="dialog-buttons">
            <button class="btn btn-secondary mr-auto">Cancel</button>
            <span class="mr-center">
                Price: <span class="price">@(Pizza.GetFormattedTotalPrice())</span>
            </span>
            <button class="btn btn-success ml-auto">Order ></button>
        </div>
    </div>
</div>
```

*Pages/Index.razor* で、ピザが選択された際に `ConfigurePizzaDialog` を表示するよう実装します。`ConfigurePizzaDialog` は現在のページのオーバーレイ表示するため、要素は任意の場所に記述可能です:

```html
@if (showingConfigureDialog)
{
    <ConfigurePizzaDialog Pizza="configuringPizza" />
}
```

アプリを起動してピザを選択してください。`ConfigurePizzaDialog` が表示されます。

![initial-pizza-dialog](https://user-images.githubusercontent.com/1874516/77239685-e3d8e700-6b9a-11ea-8adf-5ee8a69f08ae.png)


この時点ではまだ基本的な情報しか表示されませんが、これから機能を追加していきます。

## データバインディング

まずピザのサイズを指定できるようにします。`ConfigurePizzaDialog` にスライダーを追加して、サイズを選択できるようにしましょう。以下のマークアップを既存の `<form class="dialog-body"></form>` と入れ替えます:

```html
<form class="dialog-body">
    <div>
        <label>Size:</label>
        <input type="range" min="@Pizza.MinimumSize" max="@Pizza.MaximumSize" step="1" />
        <span class="size-label">
            @(Pizza.Size)" (£@(Pizza.GetFormattedTotalPrice()))
        </span>
    </div>
</form>
```

これでダイアログはスライダーを表示するようになりました。この時点ではまだスライダーを動かしても何も起こりません。

![Slider](https://user-images.githubusercontent.com/1430011/57576985-eff40400-7421-11e9-9a1b-b22d96c06bcb.png)

スライダーの値を `Pizza.Size` プロパティに反映する必要があります。またダイアログを開いた際に、プロパティの値をスライダ-に反映する必要もあります。この場合*双方向バインディング*が利用できます。

もしスクラッチで双方向バインディングを実装すると、以下のように value と @onchange イベントを組み合わせる非超があります。しかし Blazor ではより簡単な方法が提供されています。

```html
<input 
    type="range" 
    min="@Pizza.MinimumSize" 
    max="@Pizza.MaximumSize" 
    step="1" 
    value="@Pizza.Size"
    @onchange="@((ChangeEventArgs e) => Pizza.Size = int.Parse((string) e.Value))" />
```

Blazor の `@bind` ディレクティブを使う事で、双方向バインドが可能となります。`@bind` を使ってスライダーを更新します:

```html
<input type="range" min="@Pizza.MinimumSize" max="@Pizza.MaximumSize" step="1" @bind="Pizza.Size"  />
```

しかし `@bind` でデータバインドしただけでは、期待した通りの動作になりません。アプリを起動してスライダーを動かしてみます。値の更新はスライダーを離したタイミングでしか行われません。

![Slider with default bind](https://user-images.githubusercontent.com/1874516/51804870-acec9700-225d-11e9-8e89-7761c9008909.gif)

この問題を解決するには、Blazor の `@bind:<eventname>` シンタックスを使います。これは任意のイベント発生時にデータバインドを実行することが出来る機能であり、今回は `oninput` イベントが発生するたびにデータを更新するようにします。マークアップを以下のように書き換えます:

```html
<input type="range" min="@Pizza.MinimumSize" max="@Pizza.MaximumSize" step="1" @bind="Pizza.Size" @bind:event="oninput" />
```

アプリを起動して動作を確認してください。

![Slider bound to oninput](https://user-images.githubusercontent.com/1874516/51804899-28e6df00-225e-11e9-9148-caf2dd269ce0.gif)

## トッピングを追加する

`ConfigurePizzaDialog` ではトッピングの追加もサポートします。追加可能なトッピングは、バックエンドの `/toppings` API から取得するよう以下のコードを追加します:

```csharp
@inject HttpClient HttpClient

<div class="dialog-container">
...
</div>

@code {
    List<Topping> toppings;

    [Parameter] public Pizza Pizza { get; set; }

    protected async override Task OnInitializedAsync()
    {
        toppings = await HttpClient.GetJsonAsync<List<Topping>>("toppings");
    }
}
```

取得したトッピングを表示するマークアップを追加します。トッピングは最大 6 つまで追加できるようにしています。マークアップは `<form class="dialog-body">` 内、既存の `<div>` の下に追加します:

```html
<div>
    <label>Extra Toppings:</label>
    @if (toppings == null)
    {
        <select class="custom-select" disabled>
            <option>(loading...)</option>
        </select>
    }
    else if (Pizza.Toppings.Count >= 6)
    {
        <div>(maximum reached)</div>
    }
    else
    {
        <select class="custom-select" @onchange="ToppingSelected">
            <option value="-1" disabled selected>(select)</option>
            @for (var i = 0; i < toppings.Count; i++)
            {
                <option value="@i">@toppings[i].Name - (£@(toppings[i].GetFormattedPrice()))</option>
            }
        </select>
    }
</div>

<div class="toppings">
    @foreach (var topping in Pizza.Toppings)
    {
        <div class="topping">
            @topping.Topping.Name
            <span class="topping-price">@topping.Topping.GetFormattedPrice()</span>
            <button type="button" class="delete-topping" @onclick="@(() => RemoveTopping(topping.Topping))">x</button>
        </div>
    }
</div>
```

トッピングの追加と削除を行うイベントハンドラーを実装します:

```csharp
void ToppingSelected(ChangeEventArgs e)
{
    if (int.TryParse((string)e.Value, out var index) && index >= 0)
    {
        AddTopping(toppings[index]);
    }
}

void AddTopping(Topping topping)
{
    if (Pizza.Toppings.Find(pt => pt.Topping == topping) == null)
    {
        Pizza.Toppings.Add(new PizzaTopping() { Topping = topping });
    }
}

void RemoveTopping(Topping topping)
{
    Pizza.Toppings.RemoveAll(pt => pt.Topping == topping);
}
```

アプリを起動して、トッピングの追加と削除を試してください。

![Add and remove toppings](https://user-images.githubusercontent.com/1874516/77239789-c0626c00-6b9b-11ea-9030-0bcccdee6da7.png)


## コンポーネントイベント

ダイアログの `Cancel` と `Order` ボタンをクリックしても何も起きません、ダイアログ表示のコントロールは `Index` コンポーネントで行うため、ユーザーがどちらのボタンをクリックしたかを、`index` ダイアログに伝える必要があります。コンポーネントイベントは、親コンポーネントがサブスクライブできるコールバックパラメーターであり、このようなシナリオで利用できます。

`ConfigurePizzaDialog` コンポーネントに `OnCancel` と `OnConfirm` を `EventCallback` 型として追加します:

```csharp
[Parameter] public EventCallback OnCancel { get; set; }
[Parameter] public EventCallback OnConfirm { get; set; }
```

`ConfigurePizzaDialog` の `@onclick` イベントでそれぞれのイベントをトリガーします:

```html
<div class="dialog-buttons">
    <button class="btn btn-secondary mr-auto" @onclick="OnCancel">Cancel</button>
    <span class="mr-center">
        Price: <span class="price">@(Pizza.GetFormattedTotalPrice())</span>
    </span>
    <button class="btn btn-success ml-auto" @onclick="OnConfirm">Order ></button>
</div>
```

`Index` コンポーネントに `OnCancel` イベントハンドラーを実装し、`ConfigurePizzaDialog` ダイアログの OnCancel イベントに関連付けます:

```html
<ConfigurePizzaDialog Pizza="configuringPizza" OnCancel="CancelConfigurePizzaDialog" />
```

```csharp
void CancelConfigurePizzaDialog()
{
    configuringPizza = null;
    showingConfigureDialog = false;
}
```

これでユーザーがキャンセルボタンをクリックした場合、`Index.CancelConfigurePizzaDialog` が実行され、`Index` コンポーネントの再描画処理が行われます。`showingConfigureDialog` が `false` にセットされているため、ダイアログは表示されません。

通常ボタンをクリックするなどしてイベントをトリガーした場合、イベントハンドラーのデリゲートを定義しているコンポーネントで再描画処理が行われます。イベントは `Action` や `Func<string, Task>` など任意のデリゲートで定義することができます。コンポーネントに定義されていないデリゲートをイベントハンドラーで使いたい場合もありますが、通常のデリゲート型を定義しているとコンポーネントでは何も起きません。

`EventCallback` は特別な型でこれらの課題を解決します。コンパイラーにイベントハンドラーのロジックを定義しているコンポーネントにイベントがディスパッチするよう指示します。`EventCallback` は他にもいくつかの便利な機能がありますが、ここでは正しい場所にイベントをディスパッチする方法であると覚えておいてください。

アプリを起動してピザを選択し、キャンセルボタンをクリックすることでダイアログが消えるか確認してください。

`OnConfirm` イベントがトリガーされた場合は、`Order` を注文カゴに追加します。`Index` で注文を追跡するプロパティを追加します:

```csharp
List<PizzaSpecial> specials;
Pizza configuringPizza;
bool showingConfigureDialog;
Order order = new Order();
```

`Index` コンポーネントで `OnConfirm` のイベントハンドラーを実装して、`ConfigurePizzaDialog` に紐づけます:

```html
<ConfigurePizzaDialog 
    Pizza="configuringPizza" 
    OnCancel="CancelConfigurePizzaDialog"  
    OnConfirm="ConfirmConfigurePizzaDialog" />
```

```csharp
void ConfirmConfigurePizzaDialog()
{
    order.Pizzas.Add(configuringPizza);
    configuringPizza = null;

    showingConfigureDialog = false;
}
```

アプリを起動して動作を確認します。`Order` をクリックした際にダイアログが消えれば成功です。また UI 上は注文が注文カゴに追加されたか確認できないため、次にこの機能を追加します。

## 注文カゴの表示

次に注文カゴに追加したピザの詳細を表示します。また追加したピザの合計金額も計算して画面に表示します。

`ConfiguredPizzaItem` コンポーネントを作成します。このコンポーネントは追加されたピザと、ピザを削除する `EventCallback` の ２つのパラメーターを持ちます:

```html
<div class="cart-item">
    <a @onclick="OnRemoved" class="delete-item">x</a>
    <div class="title">@(Pizza.Size)" @Pizza.Special.Name</div>
    <ul>
        @foreach (var topping in Pizza.Toppings)
        {
        <li>+ @topping.Topping.Name</li>
        }
    </ul>
    <div class="item-price">
        @Pizza.GetFormattedTotalPrice()
    </div>
</div>

@code {
    [Parameter] public Pizza Pizza { get; set; }
    [Parameter] public EventCallback OnRemoved { get; set; }
}
```

`Index` コンポーネントのメイン `div` の下に、以下のマークアップを追加して、注文カゴを表示できるようにします:

```html
<div class="sidebar">
    @if (order.Pizzas.Any())
    {
        <div class="order-contents">
            <h2>Your order</h2>

            @foreach (var configuredPizza in order.Pizzas)
            {
                <ConfiguredPizzaItem Pizza="configuredPizza" OnRemoved="@(() => RemoveConfiguredPizza(configuredPizza))" />
            }
        </div>
    }
    else
    {
        <div class="empty-cart">Choose a pizza<br>to get started</div>
    }

    <div class="order-total @(order.Pizzas.Any() ? "" : "hidden")">
        Total:
        <span class="total-price">@order.GetFormattedTotalPrice()</span>
        <button class="btn btn-warning" disabled="@(order.Pizzas.Count == 0)" @onclick="PlaceOrder">
            Order >
        </button>
    </div>
</div>
```

また、注文カゴからピザを削除するイベントハンドラーと注文を確定するメソッドを実装します:

```csharp
void RemoveConfiguredPizza(Pizza pizza)
{
    order.Pizzas.Remove(pizza);
}

async Task PlaceOrder()
{
    await HttpClient.PostJsonAsync("orders", order);
    order = new Order();
}
```

アプリを実行してピザを追加すると、注文カゴの詳細が確認できます。

![Order list pane](https://user-images.githubusercontent.com/1874516/77239878-b55c0b80-6b9c-11ea-905f-0b2558ede63d.png)


注文は `PlaceOrder` メソッドによりバックエンドサーバーに渡され、データベースに保存されますが、UI から確認する方法がありません。次のセッションで新しくページを追加します。

次のセッションは  [注文ステータスの表示](03-show-order-status.md) です。
