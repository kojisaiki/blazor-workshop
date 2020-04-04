# JavaScript interop

このセッションでは *JavaScript interop* を使って、ユーザーがリアルタイムにピザの配送状況をマップで確認できるようにします。

## Map コンポーネント

ComponentsLibrary プロジェクトの `Map` コンポーネントは、現在の位置をリアルタイムにマップに表示するコンポーネントです。アプリでもこのコンポーネントを使って配達状況を表示しますが、まずは `Map` コンポーネントを見ていきます。

`Map.razor` を開いてコードを確認します:

```csharp
@using Microsoft.JSInterop
@inject IJSRuntime JSRuntime

<div id="@elementId" style="height: 100%; width: 100%;"></div>

@code {
    string elementId = $"map-{Guid.NewGuid().ToString("D")}";
    
    [Parameter] double Zoom { get; set; }
    [Parameter] List<Marker> Markers { get; set; }

    protected async override Task OnAfterRenderAsync(bool firstRender)
    {
        await JSRuntime.InvokeVoidAsync(
            "deliveryMap.showOrUpdate",
            elementId,
            Markers);
    }
}
```

`Map` コンポーネントは `@inject` ディレクティブで `IJSRuntime` インスタンスを受け取るように指定しています。このサービスはブラウザー API や既存の JavaScript ライブラリをの関数を `InvokeVoidAsync` と `InvokeAsync<TResult>` メソッドで呼び出せます。第 1 パラメーターに JavaScript のモジュールのパスと関数名を指定します。パスはルートの `window` オブジェクトからのパスとなります。残りのパラメーターは呼び出す関数に引き渡すパラメータとなります。パラメーターは JSON 形式でシリアライズされます。

`Map` コンポーネントはユニークな ID をもつ　`div` を描画し、`deliveryMap.showOrUpdate` 関数にその ID と位置を示す `Markers` を渡しています。処理は `OnAfterRenderAsync` コンポーネントライフサイクルイベントで実行されるため、`div` の描画が終わってから関数が実行される事が保障されます。`deliveryMap.showOrUpdate` 関数は `wwwroot/deliveryMap.js` ファイルに定義されていて、[leaflet.js](http://leafletjs.com) と [OpenStreetMap](https://www.openstreetmap.org/) を使い、マップを描画します。このコードの詳細より、どのように JavaScript の関数が呼び出されるかが重要であるため、この実装パターンを覚えて置いてください。

`Sdk="Microsoft.NET.Sdk.Razor"` を使う Blazor ライブラリプロジェクトは、コンパイル時に`wwwroot/` フォルダ配下にあるファイルがバンドルされます。バックエンドのサーバープロジェクトは、これらのファイルを静的ファイルとしてクライアントに配布します。

最終的に `.js` や `.css` ファイルへのリンクは `index.html` に含まれます。例えば既に紹介した `localStorage` の場合、`_content/BlazingPizza.ComponentsLibrary/localStorage.js` という相対パスが指定されます。このように Blazor ライブラリが持つファイルへのパスは `_content/<library name>/<file path>` となります。

クライアントプロジェクトのコンポーネントで `Map` をタイプした場合、エディターでコンポーネントとして認識されません。これは `@using` を使った名前空間の指定が存在しないからです。`Map` は `BlazingPizza.ComponentsLibrary.Map` 名前空間を使います。これを各コンポーネントに追加しても動作しますが、複数のコンポーネントで利用する場合はプロジェクトのルートにある `_Imports.razor` に指定できます。以下のようにライブラリの名前空間を追加します:

```html
@using System.Net.Http
@using Microsoft.AspNetCore.Authorization
@using Microsoft.AspNetCore.Components.Authorization
@using Microsoft.AspNetCore.Components.Forms
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.JSInterop
@using BlazingPizza.Client
@using BlazingPizza.Client.Shared
@using BlazingPizza.ComponentsLibrary
@using BlazingPizza.ComponentsLibrary.Map
```

その後 `OrderDetails` コンポーネントで `Map` を使うと、エディターで認識されます。`track-order-details` の `div` 直下に以下マークアップを追加します:

```html
<div class="track-order-map">
    <Map Zoom="13" Markers="orderWithStatus.MapMarkers" />
</div>
```

実はこれまで @using を明示的に指定せずに認識されていたコンポーネントは、`_Imports.razor` で `@using` が指定されているからでした。

`OrderDetails` コンポーネントが注文のステータスをポーリングで取得する度に、配送の最新の位置情報が取得でき、その結果、地図上のマーカーがリアルタイムに移動します。

![Real-time pizza map](https://user-images.githubusercontent.com/1874516/51807322-6018b880-227d-11e9-89e5-ef75f03466b9.gif)

## 確認ダイアログの実装

`Map` コンポーネントでは初めから *JavaScript interop* が構成されていましたが、次は新しくコンポーネントを作ってみます。

もしユーザーが間違えてピザを注文カゴから消してしまった場合、そのまま購入してくれないかもしれません。そこで削除時に確認ダイアログを出せるようにします。

クライアントプロジェクトに `JSRuntimeExtensions.cs` ファイルを追加し、静的クラスを作ります。`IJSRuntime` クラスに拡張メソッドとして `Confirm` メソッドを追加します。メソッド内では、 JavaScript ビルトイン関数である `confirm` 関数を呼び出します。

```csharp
public static class JSRuntimeExtensions
{
    public static ValueTask<bool> Confirm(this IJSRuntime jsRuntime, string message)
    {
        return jsRuntime.InvokeAsync<bool>("confirm", message);
    }
}
```

`Index` コンポーネントでも *JavaScript interop* を使えるよう、`IJSRuntime` サービスをインジェクトします:

```html
@page "/"
@inject HttpClient HttpClient
@inject OrderState OrderState
@inject NavigationManager NavigationManager
@inject IJSRuntime JS
```

`RemovePizza` メソッドで先ほど作成した `Confirm` メソッドの呼び出しを追加し、ピザを注文から削除した際に、確認画面が出るようにします。また確認の結果が true であった場合、`OrderState` の `RemoveConfiguredPizza` メソッドを実行します:

```csharp
async Task RemovePizza(Pizza configuredPizza)
{
    if (await JS.Confirm($"Remove {configuredPizza.Special.Name} pizza from the order?"))
    {
        OrderState.RemoveConfiguredPizza(configuredPizza);
    }
}
```

次に `index` コンポーネントの `ConfiguredPizzaItems` で現在 `RemoveConfiguredPizza` メソッドを実行している箇所を、定義した `RemovePizza` メソッドを実行するように変更します: 

```csharp
@foreach (var configuredPizza in OrderState.Order.Pizzas)
{
    <ConfiguredPizzaItem Pizza="configuredPizza" OnRemoved="@(() => RemovePizza(configuredPizza))" />
}
```

アプリを実行して、注文を作成した後、注文からピザを削除してみてください。

![Confirm pizza removal](https://user-images.githubusercontent.com/1874516/77243688-34b40400-6bca-11ea-9d1c-331fecc8e307.png)

`ConfiguredPizzaItem.OnRemoved` の呼び出しで非同期処理をサポートしていませんが、これは `EventCallback` の特性で、イベントハンドラーは同期と非同期の両方をサポートしています。

次のセッションは - [テンプレート コンポーネント](08-templated-components.md) です。
