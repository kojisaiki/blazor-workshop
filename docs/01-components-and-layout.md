# コンポーネントとレイアウト

このセッションでは、Blazor を使ってピザを注文できるアプリを開発します。アプリには、ピザのカスタマイズや注文カゴの機能、注文の実行や配送の状況確認行える機能を実装します。

## スターティングポイントとなるプロジェクト

このワークショップでは、開発するアプリのスターティングポイントとなるソリューションが提供されます。*save-points* フォルダにある、[スターティングポイント](https://github.com/dotnet-presentations/blazor-workshop/tree/master/save-points/00-Starting-point)フォルダーを確認してください。また各セッションの終了時点でのソリューションがそれぞれ提供されているため、途中でうまく動かない場合などに、完成したコードと比較することが出来ます。

> メモ: *save-points* フォルダから他のフォルダへコードをコピーする際は、ワークショップフォルダにある *Directory.Build.props* も一緒にコピーしてください。このファイルで必要なパッケージのリストアが行われます。

ソリューションは以下 4 つのプロジェクトで構成されています。

![image](https://user-images.githubusercontent.com/1874516/77238114-e2072780-6b8a-11ea-8e44-de6d7910183e.png)


- **BlazingPizza.Client**: UI を提供する Blazor WebAssembly プロジェクト
- **BlazingPizza.Server**: バックエンドを提供する ASP.NET Core プロジェクト
- **BlazingPizza.Shared**: モデルを含んだ共通プロジェクト
- **BlazingPizza.ComponentsLibrary**: Blazor コンポーネントのライブラリ及びヘルパーコードを含んだプロジェクト

実際に動かしてみましょう。

**BlazingPizza.Server** プロジェクトをスタートアッププロジェクトに設定してアプリを起動すると、シンプルなページが表示されます。

![image](https://user-images.githubusercontent.com/1874516/77238160-25fa2c80-6b8b-11ea-8145-e163a9f743fe.png)

**BlazingPizza.Client** プロジェクトの *Pages/Index.razor* ファイルのコードを確認します。

```
@page "/"

<h1>Blazing Pizzas</h1>
```

トップであるホームページは単一の Blazor コンポーネントで構成されていて、ページになるコンポーネントは、初めにルーティング情報でもある `@page` ディレクティブを指定します。ここではパスがルート (/) として設定されているため、ユーザーがルートにアクセスした際、index ページが描画されます。

## ピザの一覧を表示する

まず、ホームページであり `Index` コンポーネントにピザの一覧を表示する機能を追加します。

*Index.razor* に `@code` ブロックを追加して、ピザを保持するリスト型プロパティを指定します。

```csharp
@code {
    List<PizzaSpecial> specials;
}
```

`@code` ブロックに追加したコードはコンポーネントのクラスの一部として扱われます。リストに指定した `PizzaSpecial` 型は Shared プロジェクトで定義されています。

ピザの一覧を取得するには、バックエンドの API を呼び出します。Blazor では、バックエンドのベースアドレスが既に設定された `HttpClient` が Dependency Injection (DI) 経由で提供されます。DI を利用するために `@inject` ディレクティブに `HttpClient` を指定します。

```html
@page "/"
@inject HttpClient HttpClient
```

`@inject` ディレクティブには、最初にプロパティの型を指定し、続いてプロパティの名前を指定します。 オブジェクトは DI を通じてコンポーネントに渡されます。

`@code` ブロックの `OnInitializedAsync` メソッドで、ピザ一覧を取得します。このメソッドは Blazor コンポーネントのライフサイクルの一部で、コンポーネントが初期化される際に呼び出されます。詳細は [ASP.NET Core Blazor ライフサイクル](https://docs.microsoft.com/ja-jp/aspnet/core/blazor/lifecycle?view=aspnetcore-3.1) を参照してください。

`GetJsonAsync<T>()` メソッドで結果の JSON を任意の型にデシリアライズします。

```csharp
@code {
    List<PizzaSpecial> specials;

    protected override async Task OnInitializedAsync()
    {
        specials = await HttpClient.GetJsonAsync<List<PizzaSpecial>>("specials");
    }
}
```

`/specials` API はバックエンドプロジェクトの `SpecialsController` で定義されています。

コンポーネントの初期化が終わるとマークアップが描画されます。`Index` コンポーネントのマークアップを変更して、ピザ一覧を表示します:

```html
<div class="main">
    <ul class="pizza-cards">
        @if (specials != null)
        {
            @foreach (var special in specials)
            {
                <li style="background-image: url('@special.ImageUrl')">
                    <div class="pizza-info">
                        <span class="title">@special.Name</span>
                        @special.Description
                        <span class="price">@special.GetFormattedBasePrice()</span>
                    </div>
                </li>
            }
        }
    </ul>
</div>
```

`Ctrl-F5` を押下してアプリをデバッグなしで実行してください。以下のようにピザの一覧が表示されます。スタイリングは `wwwroot/css/site.css` に定義されています。この CSS はこれ以降開発するモジュール用のスタイリングも事前に全て定義されています。

![image](https://user-images.githubusercontent.com/1874516/77239386-6c558880-6b97-11ea-9a14-83933146ba68.png)


## レイアウトの作成

次にアプリのレイアウトを作成します。

Blazor では、レイアウトもコンポーネントの一種です。`LayoutComponentBase` を継承し、実際のコンテンツが描画される場所に `@Body` プロパティを配置します。今回のアプリでは、*Shared/MainLayout.razor* がレイアウトコンポーネントとなります。

```html
@inherits LayoutComponentBase

<div class="content">
    @Body
</div>
```

レイアウトとアプリの関連は、`App.razor` の `<Router>` で確認できます。`DefaultLayout` パラメーターに、既定で使用するレイアウトが指定されていて、独自にレイアウト定義を持たないページは、既定レイアウトが適用されます。

任意のページの先頭で `@layout SomeOtherLayout` を指定する事で、ページ単位でレイアウトを上書き出来ます。このアプリケーションでは、既定のレイアウトのみを使用します。

`MainLayout` コンポーネントにメニューバーを追加します。ブランドロゴおよびトップページへのリンクも追加しています:

```html
@inherits LayoutComponentBase

<div class="top-bar">
    <img class="logo" src="img/logo.svg" />

    <NavLink href="" class="nav-tab" Match="NavLinkMatch.All">
        <img src="img/pizza-slice.svg" />
        <div>Get Pizza</div>
    </NavLink>
</div>

<div class="content">
    @Body
</div>
```

`NavLink` は Blazor が提供するビルトインコンポーネントです。他のコンポーネントを呼び出す場合は、コンポーネント名を要素名として定義し、パラメータは属性として指定します。

`NavLink` コンポーネントは HTML のアンカータグとほぼ同じですが、現在のアドレスとリンクのアドレスが一致した場合 `active` クラスを付与するため、CSS でのスタリングが容易になります。`NavLinkMatch.All` は現在のアドレスとリンクのアドレスが完全一致した場合という条件です。`NavLink` については後ほど動作をしっかり見ていきます。

アプリを起動し、以下のようにメニューが出ることを確認してください。

![image](https://user-images.githubusercontent.com/1874516/77239419-aa52ac80-6b97-11ea-84ae-f880db776f5c.png)

# まとめ

このセッションでは、アプリの元となるソリューションの確認と、Dependency Injection の使い方、またコンポーネントの画面描画について見ていきました。

次のセッションは [ピザのカスタマイズ](02-customize-a-pizza.md) です。
