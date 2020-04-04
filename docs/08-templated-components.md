# テンプレート コンポーネント

幾つかの既存コンポーネントを改修して、再利用可能なコンポーネントにしていきます。また新しいコンポーネント用にソリューションも作成します。

新しいプロジェクトは Razor クラスライブラリプロジェクトテンプレートを使います。

## コンポーネントライブラリの作成 (Visual Studio)

Visual Studio のソリューションエクスプローラーより、ソリューションを右クリックしてプロジェクトを追加します。

テンプレートで `Razor Class Library` を選択します。

![image](https://user-images.githubusercontent.com/1430011/65823337-17990c80-e209-11e9-9096-de4cb0d720ba.png)

名前を `BlazingComponents` として作成します。

## コンポーネントライブラリの作成 (コマンドライン)

コマンドラインからも **dotnet** コマンドでプロジェクトの作成ができます。以下のコマンドを実行します。

```shell
dotnet new razorclasslib -o BlazingComponents
dotnet sln add BlazingComponents
```

これで `BlazingComponents` プロジェクトが作成されます。

## ライブラリプロジェクトを理解する

ソリューションエクスプローラーより *BlazingComponents* をダブルクリックして開きます:

```xml
<Project Sdk="Microsoft.NET.Sdk.Razor">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <RazorLangVersion>3.0</RazorLangVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Components" Version="3.1.2" />
    <PackageReference Include="Microsoft.AspNetCore.Components.Web" Version="3.1.2" />
  </ItemGroup>

</Project>
```

幾つか重要な点があります。

パッケージは `netstandard2.0` をターゲットにしています。Blazor サーバーは `netcoreapp3.1` ベースで、Blazor WebAssembly は `netstandard2.1` ベースのため、`netstandard2.0` であればどちらでも利用が可能です。

`<RazorLangVersion>3.0</RazorLangVersion>` は Razor のバージョンを指定しています。バージョン 3 は `.razor` 拡張子のファイルを扱えます。

最後に `<PackageReference />` で必要な Blazor コンポーネントパッケージを参照しています。

## テンプレートダイアログの開発

現在の `index` コンポーネントを振り返り、幾つかの要素を疎結合なコンポーネントへとリファクタリングします。

*再利用可能なダイアログ*を考えると、表示/非表示の制御の他、独自のスタイリングをサポートする必要があります。また真に再利用可能である為には、ヘッダーやコンテンツなどをパラメーターとして渡せることが必要です。コンテンツをパラメータとして渡せるコンポーネントのことを*テンプレート コンポーネント*と呼びます。

Blazor はこの機能を提供しており、レイアウトも似た機能を提供します。レイアウトには `Body` パラメーターがあり、レイアウトの実装は `Body` の周りで行われます。レイアウトでは `Body` パラメーターは `RenderFragment` 型です。

どのコンポーネントでも `RenderFragment` をパラメーターで受け取ることができます。`App.razor` はこの機能を使っていて、ルーティングコンポーネントや認証コンポーネントもテンプレートコンポーネントです。

新しく `TemplatedDialog.razor` を `BlazingComponents` プロジェクトに作成して、以下マークアップを追加します:

```html
<div class="dialog-container">
    <div class="dialog">

    </div>
</div>
```

ここでは 2 つのことを実現します。

1. コンテンツをダイアログのパラメーターとして受けとる
2. 条件に応じてダイアログの表示/非表示を切り替える

まず `RenderFragment` 型の `ChildContent` を追加します。`ChildContent` は特別な名前で、コンテンツが 1 つだけ渡される場合に利用できます:

```csharp
@code {
    [Parameter] public RenderFragment ChildContent { get; set; }
}
```

つぎに `dialog` で作成した `ChildContent` を指定します:

```html
<div class="dialog-container">
    <div class="dialog">
        @ChildContent
    </div>
</div>
```

`@ChildContent` の書き方に違和感がある場合、レイアウトで `@Body` と記述したことを思い出してください。`RenderFragment` は直接実行されることはなく、フレームワークが描画します。

次に `bool` 型の `Show` プロパティを追加して、`@if (Show) { ... }` シンタックスで表示の制御を行います:

```html
@if (Show)
{
    <div class="dialog-container">
        <div class="dialog">
            @ChildContent
        </div>
    </div>
}

@code {
    [Parameter] public RenderFragment ChildContent { get; set; }
    [Parameter] public bool Show { get; set; }
}
```

ソリューションをビルドしてエラーが無いことを確認します。

## 参照の追加

コンポーネントを `BlazingPizza.Client` で使えるように、`BlazingComponents` をプロジェクト参照追加します。

追加後 `_Imports.razor` ファイルに `@using` を追加します:

```html
@using BlazingComponents
```

念のため再度ソリューションをビルドしてエラーがないことを確認します。

## ConfigurePizzaDialog のリファクタリング

`ConfigurePizzaDialog` のダイアログを `TemplatedDialog` で使うように変えていきます。現在 `ConfigurePizzaDialog` は `div` を複数含んでいます。:

```html
<div class="dialog-container">
    <div class="dialog">
        <div class="dialog-title">
        ...
        </div>
        <form class="dialog-body">
        ...
        </form>

        <div class="dialog-buttons">
        ...
        </div>
    </div>
</div>
```

外側 2 つの `div` は `TemplatedDialog` に存在するため、削除します:

```html
<div class="dialog-title">
...
</div>
<form class="dialog-body">
...
</form>

<div class="dialog-buttons">
...
</div>
```

## 新しいダイアログを使う

新しいテンプレートコンポーネントを `Index.razor` で使います。`Index.razor` を開いて、以下のマークアップを見つけます。

```html
@if (OrderState.ShowingConfigureDialog)
{
    <ConfigurePizzaDialog
        Pizza="@OrderState.ConfiguringPizza"
        OnConfirm="@OrderState.ConfirmConfigurePizzaDialog"
        OnCancel="@OrderState.CancelConfigurePizzaDialog" />
}
```

このマークアップを、`TemplatedDialog` に置き換えます。

```html
<TemplatedDialog Show="OrderState.ShowingConfigureDialog">
    <ConfigurePizzaDialog 
        Pizza="OrderState.ConfiguringPizza" 
        OnCancel="OrderState.CancelConfigurePizzaDialog" 
        OnConfirm="OrderState.ConfirmConfigurePizzaDialog" />
</TemplatedDialog>
```

`TemplatedDialog` コンポーネントの `Show` 属性に渡した `OrderState.ShowingConfigureDialog` で表示/非表示を切り替えます。`ChildContent` は特別な名前のため明示的に指定する必要なく、`<TemplatedDialog> </TemplatedDialog>` の間に渡されたマークアップを `RenderFragment` パラメーターとして扱います。

> メモ: 複数の `RenderFragment` を持つテンプレート ダイアログを作成することもできます。ここでは単位のコンテンツを使う場合に、`RenderFragment` を渡す例をしましましたが、後で複数コンテンツのダイアログも作成します。詳細は [ASP.NET Core Blazor テンプレート コンポーネント](https://docs.microsoft.com/ja-jp/aspnet/core/blazor/templated-components?view=aspnetcore-3.1)を参照してください。

アプリを実行して、これまでと同じ挙動であることを確認します。

## より高度なテンプレートコンポーネント

基本的なテンプレートコンポーネントを作ったので、次はより高度なテンプレートを作ります。

`MyOrders.razor` ページでは複数の注文が表示できますが、3 つの状態 (ロード中、データがない場合、データがある場合)で表示を切り替えています。これを再利用可能なテンプレート コンポーネントにします。

`TemplatedList.razor` を `BlazingComponents` に追加します。このテンプレートコンポーネントは以下の機能を持たせます。

1. 非同期でのデータロード
2. 3 つの状態それぞれの描画ロジック

非同期のデータロードに関しては、デリゲート型 `Func<Task<List<?>>>` を受け取ることで実現できます。**?** にはロードするデータの型を指定します。今回任意の型をサポートするため、ジェネリック型を指定します。ジェネリック型は `@typeparam` ディレクティブを使います。`TemplatedList.razor` の先頭に以下コードを追加します:

```html
@typeparam TItem
```

`@typeparam` は便利な Razor シンタックスで C# のジェネリックと同様に機能します。

>メモ: 現時点では型パラメーターの制約はサポートされていませんが、将来的にはサポートされる予定です。

ジェネリック型のパラメータを指定したので、デリゲートを受け取って処理するコードを追加します:

```csharp
@code {
    List<TItem> items;

    [Parameter] public Func<Task<List<TItem>>> Loader { get; set; }

    protected override async Task OnParametersSetAsync()
    {
        items = await Loader();
    }
}
```

データの取得ができたので、画面を描画します:

```html
@if (items == null)
{

}
else if (items.Count == 0)
{
}
else
{
    <div class="list-group">
        @foreach (var item in items)
        {
            <div class="list-group-item">
                
            </div>
        }
    </div>
}
```

コンポーネントパラメーターとして 3 つの `RenderFragment` を受け取るため、先ほど使用した `ChildContent` が使えません。よってそれぞれ別の名前を付けます。またジェネリック型のパラメーターを受け取る Item は `RenderFragment<T>` 形式でしています。

以下のコードを追加します:

```csharp
    [Parameter] public RenderFragment Loading{ get; set; }
    [Parameter] public RenderFragment Empty { get; set; }
    [Parameter] public RenderFragment<TItem> Item { get; set; }
```

受け取った `RenderFragment` を描画します:

```html
@if (items == null)
{
    @Loading
}
else if (items.Count == 0)
{
    @Empty
}
else
{
    <div class="list-group">
        @foreach (var item in items)
        {
            <div class="list-group-item">
                @Item(item)
            </div>
        }
    </div>
}
```

`Item` はパラメーターを関数の引数のように受け取れます。`RenderFragment<T>` は他の `RenderFragment` 同様に処理されます。

最後に、`MyOrders.razor` では `<div class="list-group">` で `list-group` 以外に追加のクラスを指定し、 CSS でスタイリングしています。それに対応するため `string` 型のパラメーターを追加します:

```csharp
    [Parameter] public string ListGroupClass { get; set; }
}
```

`<div class="list-group">` マークアップを `<div class="list-group @ListGroupClass">` に更新します。最終的に `TemplatedList.razor` は以下のようになります:

```html
@typeparam TItem

@if (items == null)
{
    @Loading
}
else if (items.Count == 0)
{
    @Empty
}
else
{
    <div class="list-group @ListGroupClass">
        @foreach (var item in items)
        {
            <div class="list-group-item">
                @Item(item)
            </div>
        }
    </div>
}

@code {
    List<TItem> items;

    [Parameter] public Func<Task<List<TItem>>> Loader { get; set; }
    [Parameter] public RenderFragment Loading { get; set; }
    [Parameter] public RenderFragment Empty { get; set; }
    [Parameter] public RenderFragment<TItem> Item { get; set; }
    [Parameter] public string ListGroupClass { get; set; }

    protected override async Task OnParametersSetAsync()
    {
        items = await Loader();
    }
}
```

## TemplatedList の利用

早速追加した `TemplatedList` を `MyOrders.razor` で使います。

まず `TemplatedList` に対してデータを非同期でロードするためのメソッドを `MyOrders.OnParametersSetAsync` を変更して作成します:

```csharp
@code {
    Task<List<OrderWithStatus>> LoadOrders()
    {
        return HttpClient.GetJsonAsync<List<OrderWithStatus>>("orders");
    }
}
```

これでシグネチャーが `Loader` の期待する `Func<Task<List<?>>>` となり **?** は `OrderWithStatus` 型指定されています。次に `TemplatedList` を使うように変更します:

```html
<div class="main">
    <TemplatedList>
    </TemplatedList>
</div>
```

コンパイラーは `TemplatedList` がジェネリック型のパラメーターを受け取ることを知っているため、エラーを表示します。そこで引数を `Loader` に指定します:

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders">
    </TemplatedList>
</div>
```

> メモ: ジェネリック型パラメーターを受け取るコンポーネントには、以下のようにパラメーターのタイプ(ここでは `TItem`) に直接型を指定することもできます。

```html
<div class="main">
    <TemplatedList TItem="OrderWithStatus">
    </TemplatedList>
</div>
```

ここでは `Loader` 経由で型情報が渡る為必要ありません。


次に複数の `RenderFragment` パラメーターを `TemplatedDialog` に渡します。単一のパラメーターの場合 `[Parameter] RenderFragment ChildContent` が利用できましたが、ここでは明示的に渡します:

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders">
        <Loading>Hi there!</Loading>
        <Empty>
            How are you?
        </Empty>
        <Item>
            Are you enjoying Blazor?
        </Item>
    </TemplatedList>
</div>
```

ここで `Item` パラメータは `RenderFragment<T>` 型であり、既定でパラメーターは `context` と呼ばれますが、明示的に`Context` 属性に値を渡すことができます:

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders">
        <Loading>Hi there!</Loading>
        <Empty>
            How are you?
        </Empty>
        <Item Context="item">
            Are you enjoying Blazor?
        </Item>
    </TemplatedList>
</div>
```

最後に既存の `MyOrders.razor` よりコンテンツを取ってきて渡します:

```html
<div class="main">
    <TemplatedList Loader="@LoadOrders" ListGroupClass="orders-list">
        <Loading>Loading...</Loading>
        <Empty>
            <h2>No orders placed</h2>
            <a class="btn btn-success" href="">Order some pizza</a>
        </Empty>
        <Item Context="item">
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
        </Item>
    </TemplatedList>
</div>
```

`TemplatedList` コンポーネントでは `Loader` 以外に `ListGroupClass` 属性に `MyOrders.razor` を渡して、元々と同じ状態を作っています。

実際に機能するか確認するには、以下の手順で操作します。

1. `pizza.db` ファイルを `Blazor.Server` プロジェクトより削除。
2. `LoadOrders` メソッドを `async` に変更して、`await Task.Delay(3000);` 追加。これによりロード中の画面が確認可能。

このセッションでは以下のことを学びました。

1. *content* をパラメーターをとして受け取るコンポーネントを作れる
2. テンプレートコンポーネントを使うとダイアログやデータロードなどを抽象化できる
3. コンポーネントはジェネリック型を受け取れ、より汎用性を上げられる

次のセッションは - [プログレッシブ Web アプリ(PWA)](09-progressive-web-app.md) です。
