# 認証

アプリケーションは期待した通りに動作しています。ユーザーはピザを選択して注文できますし、その後の追跡もできています。しかし現状複数のユーザーを区別できないという大きな問題があります。"My orders" ページでは *全ての* ユーザーの注文が表示され、誰でも観ることができます。

解決策はユーザーログインを使った*認証*を組み込むことです。また*認可*によってログインしたユーザーが何ができるかを制御する必要もあります。

## サーバーサイドでログインを強制

最も重要な点は、本当のセキュリティはサーバーサイドで強制することです。クライアントサイドは*行儀の良い*ユーザーは制御できますが、少しでも知識があると挙動を変更することは難しくありません。よってまずはサーバーサイドでの認証と認可を実装していきます。

`BlazorPizza.Server` プロジェクトの `OrdersController.cs` を開きます。このクラスで `/orders` と `/orders/{orderId}` エンドポイントの処理を行っています。コントローラーを保護するために、`[Authorize]` 属性を追加します:

```csharp
[Route("orders")]
[ApiController]
[Authorize]
public class OrdersController : Controller
{
}
```

`AuthorizeAttribute` クラスは `Microsoft.AspNetCore.Authorization` 名前空間にあります。

この変更だけでアプリからは注文の実行も、既に作成した注文の取得もできなくなります。ブラウザの開発者ツールのネットワークを確認すると、バックエンドより HTTP 302 が返っていて、ログイン URL にリダイレクトされていますが、まだログインページがないためそこで止まっています。

![Secure orders](https://user-images.githubusercontent.com/1874516/77242788-a9ce0c00-6bbf-11ea-98e6-c92e8f7c5cfe.png)

## 認証しているか確認する

クライアントサイドではユーザーが認証された状態か、またそれがどのユーザーか確認する必要があります。Blazor はこの用途で使える `AuthenticationStateProvider` を提供します。`AuthenticationStateProvider` は DI として機能するため扱いが容易です。

Blazor サーバーは `AuthenticationStateProvider` をサーバーサイドの認証と連携してログインユーザーを確認できますが、今回のアプリは Blazor WebAssembly アプリのため、クライアント側で `AuthenticationStateProvider` を実装する必要があります。

まず `ServerAuthenticationStateProvider` クラスを `BlazingPizza.Client` プロジェクトのルートに追加します:

```csharp
using System.Security.Claims;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Components.Authorization;

namespace BlazingPizza.Client
{
    public class ServerAuthenticationStateProvider : AuthenticationStateProvider
    {
        public override async Task<AuthenticationState> GetAuthenticationStateAsync()
        {
            // Currently, this returns fake data
            // In a moment, we'll get real data from the server
            var claim = new Claim(ClaimTypes.Name, "Fake user");
            var identity = new ClaimsIdentity(new[] { claim }, "serverauth");
            return new AuthenticationState(new ClaimsPrincipal(identity));
        }
    }
}
```

今は `GetAuthenticationStateAsync` メソッドで常に `Fake user` がログインしている状態を返しています。作成したクラスを `Program.cs` で DI サービスに追加します:

```csharp
public static async Task Main(string[] args)
{
    var builder = WebAssemblyHostBuilder.CreateDefault(args);
    
    builder.RootComponents.Add<App>("app");

    builder.Services.AddBaseAddressHttpClient();
    builder.Services.AddScoped<OrderState>();

    // Add auth services
    builder.Services.AddOptions();
    builder.Services.AddAuthorizationCore();
    builder.Services.AddScoped<AuthenticationStateProvider, ServerAuthenticationStateProvider>();

    await builder.Build().RunAsync();
}
```

認証ステートをクライアントサイドに渡すため、もう一つコンポーネントを追加します。`App.razor` で `<Router>` を `<CascadingAuthenticationState>` で囲みます:

```html
<CascadingAuthenticationState>
    <Router AppAssembly="typeof(Program).Assembly" Context="routeData">
        ...
    </Router>
</CascadingAuthenticationState>
```

挙動に変化が無いように見えるかもしれませんが、これで直下のコンポーネントだけでなく、依存する全てのコンポーネントに*パラメーターのカスケード*が行えるようになり、認証ステートも渡せるようになります。

最後に UI をカスタマイズしましょう。

## ログイン状態の表示

`LoginDisplay` コンポーネントを `Shared` フォルダに追加します:

```html
<div class="user-info">
    <AuthorizeView>
        <Authorizing>
            <text>...</text>
        </Authorizing>
        <Authorized>
            <img src="img/user.svg" />
            <div>
                <span class="username">@context.User.Identity.Name</span>
                <a class="sign-out" href="user/signout">Sign out</a>
            </div>
        </Authorized>
        <NotAuthorized>
            <a class="sign-in" href="user/signin">Sign in</a>
        </NotAuthorized>
    </AuthorizeView>
</div>
```

`<AuthorizeView>` は Blazor のビルトインコンポーネントで、認証ステートによって表示する画面を変えられます。また認可ステートの条件も指定できますが、ここでは特別な条件を指定していません。既定ではユーザーがログインしていると認可もされているとみなします。

`<AuthorizeView>` は画面のどこでも利用が可能です。ロールベースでメニューを切り替えたり出来ますが、ここでは単純にログインしているかどうかだけを切り分けていて、ログイン/サインアウトを表示します。

`MainLayout` を開いて `<div class="top-bar">` の最後に `LoginDisplay` を追加します:

```html
<div class="top-bar">
    (... leave existing content in place ...)

    <LoginDisplay />
</div>
```

現在は `GetAuthenticationStateAsync` で常にログインしている状態を返すため、アプリでも `Fake User` でログインしている状態として表示されます。このため "sign out" リンクも機能しません:

![Fake user](https://user-images.githubusercontent.com/1874516/77243292-a6d61a00-6bc5-11ea-8250-d841988b6dda.png)

しかしサーバーサイドではこの偽ログインは有効でないため、注文情報は依然として取得できません。

## Twitter 連携でログインする

アプリでクッキーベースの認証を実装します。クッキーベース認証は以下のように機能します:

1. クライアントからサーバーサイドにユーザーがログインしているか確認。
1. サーバーサイドは ASP.NET Core のビルトインクッキーベース認証を使ってログインを追跡しているため、クライアントにユーザー名を含めた認証情報を返せる
1. クライアントがサーバーサイドに認証フローを依頼した場合、サーバーサイドは ASP.NET Core のビルトイン OAuth 認証を使って Twitter のログインページにリダイレクト。Twitter 以外にも Google や Facebook ログインもサポートしているため簡単に構成出来るうえ、独自データベースで認証する *Identity* システムもサポート。
1. ユーザーがログインしたら認証クッキーを付与。このクッキーにはユーザー名も含まれる。
1. クライアントアプリが再起動し、サーバーより返ったユーザー名を表示。
1. 以降の `OrdersController` に対する HTTP 要求にクッキーを含め、サーバーサイドで認可できるようになる。
1. サインアウトを要求した場合、サーバーサイドで認証クッキーを削除。

`LoginDisplay` の "sign in" と "sign out" リンクは `BlazingPizza.Server` プロジェクトの `UserController` へのリンクとなっています。 コードを確認してどのように認証をハンドルしているか確認してください。

次にクライアントからサーバーに対してログイン状態の確認を行います。`ServerAuthenticationStateProvider` に戻ってロジックを以下のように変更します:

```cs
using System.Net.Http;
using System.Security.Claims;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Components;
using Microsoft.AspNetCore.Components.Authorization;

namespace BlazingPizza.Client
{
    public class ServerAuthenticationStateProvider : AuthenticationStateProvider
    {
        private readonly HttpClient _httpClient;

        public ServerAuthenticationStateProvider(HttpClient httpClient)
        {
            _httpClient = httpClient;
        }

        public override async Task<AuthenticationState> GetAuthenticationStateAsync()
        {
            var userInfo = await _httpClient.GetJsonAsync<UserInfo>("user");

            var identity = userInfo.IsAuthenticated
                ? new ClaimsIdentity(new[] { new Claim(ClaimTypes.Name, userInfo.Name) }, "serverauth")
                : new ClaimsIdentity();

            return new AuthenticationState(new ClaimsPrincipal(identity));
        }
    }
}
```

アプリを起動すると `/user` エンドポイントはログアウト状態であることを返します。そのため、UI では *Sign in* リンクが表示されます。

"sign in" をクリックすると Twitter ログインへリダイレクトされ、ログインが実行されます。ログインが完了すると画面にユーザー名が表示されます。

> Tip: ログインがうまく出来ない場合、バックエンドサーバーのポートが `64589` か `64590` でない可能性があります。`BlazingPizza.Server/Properties/launchSettings.json` の設定を見直してポートを直してください。

![Signed in](https://user-images.githubusercontent.com/1874516/77243353-5d39ff00-6bc6-11ea-8962-8ed862c50c4b.png)

OAuth フローが正常に終了するには、バックエンドサーバーが `http(s)://localhost:64589` か `http(s)://localhost:64590` で起動している必要があります。異なるポートで起動する場合は、`BlazingPizza.Server/Properties/launchSettings.json` を見直してください。Twitter との連携情報は `appsettings.Development.json` に保存されています。独自に連携させたい場合は、[Twitter 開発者コンソール](https://developer.twitter.com/apps) よりアプリケーションを登録して、アプリケーション ID とシークレットを取得することもできます。この場合、任意のコールバック URL を指定する事ができます。

認証の情報はバックエンドでクッキーに保存されるため、アプリケーションを再読み込みしてもログイン状態は維持されます。*Sign out* をクリックして認証クッキーを削除するとログアウト状態に戻ります。

## 注文を行う前に認証を強制する

ログインした状態であれば、注文の実行も既存の注文の取得もできるようになりました。しかしログアウトした状態では引き続きそれらの操作は出来ませんが、UI にはエラーが表示されないため、ユーザー視点ではアプリが動作しない理由が分かりません。

この問題を解消するために、必要に応じてサインイン画面にリダイレクトできるようにします。

`Checkout` コンポーネントに `OnInitializedAsync` を追加して、ユーザーの認証ステートを確認します。もしログインしていない場合は、*Sign in* ページにリダイレクトします。

```csharp
@code {
    [CascadingParameter] public Task<AuthenticationState> AuthenticationStateTask { get; set; }

    protected override async Task OnInitializedAsync()
    {
        var authState = await AuthenticationStateTask;
        if (!authState.User.Identity.IsAuthenticated)
        {
            // The server won't accept orders from unauthenticated users, so avoid
            // an error by making them log in at this point
            NavigationManager.NavigateTo("user/signin?redirectUri=/checkout", true);
        }
    }

    // Leave PlaceOrder unchanged here
}
```

これでアプリを起動してログアウト後、チェックアウト画面まで来ると、*Sign in* 画面にリダイレクトされます。`[CascadingParameter]` は `App` コンポーネントで指定した `<CascadingAuthenticationState>` 経由で `AuthenticationStateProvider` が渡されたものです。

しかし、注意深く画面を見ると、Twitter のログインページが出る前に、一瞬チェックアウト画面が表示されます。この動作を解消したい場合、`<AuthorizeView>` を使用して画面を分けることが可能です。`Checkout.razor` のマークアップを変更します:

```html
<div class="main">
    <AuthorizeView Context="authContext">
        <NotAuthorized>
            <h2>Redirecting you...</h2>
        </NotAuthorized>
        <Authorized>
            [the whole EditForm and contents remains here]
        </Authorized>
    </AuthorizeView>
</div>
```

これで認証されていない場合には "Redirecting you..." が表示され、チェックアウト画面は見えません。

## 注文の情報を画面遷移時にも保持する

Blazor はクライアントサイドの SPA である為、注文などのアプリケーションステートはクライアントサイドにキャッシュされています。そのため、Twitter のログイン画面などに遷移するとステート情報が失われ、結果として注文情報が失われます。

この問題は以下の手順で再現できます。

1. アプリを起動して、ログアウト状態にする。
1. ピザを選択して注文を作る。
1. チェックアウト画面に遷移して Twitter ログインをする。
1. アプリに戻ってくると注文の中身が消失する。

これは SPA 共通の課題ですが、解決方法は簡単で、ステートをブラウザの `localStorage` に保存することです。 `localStorage` は JavaScript API のため、*JavaScript interop* 機能を使います。`Checkout.razor` に戻って `IJSRuntime` インスタンスをインジェクトします:

```csharp
@inject IJSRuntime JSRuntime
```

`OnInitializedAsync` メソッドの `NavigationManager.NavigateTo` の前に以下のコードを追加します:

```csharp
await LocalStorage.SetAsync(JSRuntime, "currentorder", OrderState.Order);
```

*JavaScript interop* については後のセッションでも扱うためここでは詳細は説明しませんが、この方法で JavaScript API が利用できる事だけ覚えておいてください。実装は `BlazingPizza.ComponentsLibrary` プロジェクトの `LocalStorage.cs` と `localStorage.js` にあります。

これで画面遷移の直前にアプリケーションステートが JSON 形式で保存されます。ブラウザのコンソールでその情報を確認できます。

![image](https://user-images.githubusercontent.com/1101362/59276103-90258e80-8c55-11e9-9489-5625f424880f.png)

データの保存を行ったので、次はデータの取り出しを行います。`Checkout.razor` の `OnInitializedAsync` メソッドに以下コードを追加します:

```csharp
// Try to recover any temporary saved order
if (!OrderState.Order.Pizzas.Any())
{
    var savedOrder = await LocalStorage.GetAsync<Order>(JSRuntime, "currentorder");
    if (savedOrder != null)
    {
        OrderState.ReplaceOrder(savedOrder);
        await LocalStorage.DeleteAsync(JSRuntime, "currentorder");
    }
    else
    {
        // There's nothing check out - go to home
        NavigationManager.NavigateTo("");
    }
}
```

また `OrderState` クラスで読み込まれた注文を受け取れるようにします:

```csharp
public void ReplaceOrder(Order order)
{
    Order = order;
}
```

上記の実装で、画面遷移時にアプリケーションステートが失われる問題は解決しました。アプリを起動して再現試験を行ってください。

## 非ログインユーザーの "My orders" 画面制御

ログインしていない状態で "My orders" ページを開くと、`/order` エンドポイントに拒否されるため、クライアントサイドで例外処理が発生します。ユーザーにログインするように促す方が意味がある為、改修していきます。

コンポーネント内で認証/認可をハンドルする基本的な方法は 3 種類あります。既に以下の 2 種類は見てきました:

 * `<AuthorizeView>` の利用: 認証ステートや認可の条件によって UI を切り替えることができます。
 * `[CascadingParameter]` 経由で `Task<AuthenticationState>` を受け取る: イベントハンドラーのロジックなどで `AuthenticationState` を利用する際に利用でます。

ここでは新たに `@page` ディレクティブに `[Authorize]` 属性を指定する方法を使います。この方法はページ自体のアクセスを制御する場合に便利です。

`MyOrders` コンポーネントの `@page` ディレクティブの下に以下ラインを追加します:

```csharp
@attribute [Authorize]
```

`[Authorize]` 属性を指定するとルーティングシステムの一部として機能します。また `App.razor` で `<RouteView ../>`  `<AuthorizeRouteView .../>` に変更する必要があります:

```html
<CascadingAuthenticationState>
    <Router AppAssembly="typeof(Program).Assembly" Context="routeData">
        <Found>
            <AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)" />
        </Found>
        ...
    </Router>
</CascadingAuthenticationState>
```

`AuthorizeRouteView` コンポーネントは `RouteView` と同等の機能があり、さらに `[Authorize]` 属性と連動します。

これでログインしたユーザーは *My orders* ページにアクセスできますが、ログインしていないユーザーはページにアクセスすると *Not authorized* というメッセージが表示されます。

最後にメッセージを少しカスタマイズしてみます。単に *Not authorized* と表示する代わりに *Sign in* へのリンクを表示します。`App.razor` の `<NotAuthorized>` セクションと `<Authorizing>` セクションを以下のように変更します:

```html
<AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)">
    <NotAuthorized>
        <div class="main">
            <h2>You're signed out</h2>
            <p>To continue, please sign in.</p>
            <a class="btn btn-danger" href="user/signin">Sign in</a>
        </div>
    </NotAuthorized>
    <Authorizing>
        <div class="main">Please wait...</div>
    </Authorizing>
</AuthorizeRouteView>
```

アプリを起動して、ログアウトした状態で *My orders* へ移動すると、既定よりは良い画面が表示されます:

![image](https://user-images.githubusercontent.com/1101362/51807840-11225180-2284-11e9-81ed-ea9caacb79ef.png)

## 非ログインユーザーの注文詳細画面制御

続いて注文詳細の画面です。ログインしていない状態で `/myorders/1` を開くと、以下の画面が表示されます。

![image](https://user-images.githubusercontent.com/1101362/51807869-5f375500-2284-11e9-8417-dcd572cd028d.png)

先ほど同様、ログインしていない状態ではバックエンドで処理を拒否される為です。

こちらも同じく、`OrderDetails.razor` で `[Authorize]` を指定することで対処できます。

## データアクセスの認可

バックエンドで認証 (Authentication) を必須にしましたが、ユーザーの識別 (Authorization) まではしていないため、全ての注文が見える状況は変わっていません。

バックエンドでもユーザーを識別することは簡単です。ここでは `OrdersController` を改修して、自分自身の注文だけを取得できるようにします。まずはクライアントサイドから注文を作成する際に、ユーザー Id を指定します。`PlaceOrder` メソッドでコメントアウトされている以下コードを有効にします:

```csharp
order.UserId = GetUserId();
```

これで各注文にユーザー ID の情報が付与されます。

次に、`OrderController` の `GetOrders` および `GetOrderWithStatus` メソッドでコメントアウトされている `.Where` コードを有効にします。この変更により、クエリでユーザー ID が利用されます:

```csharp
.Where(o => o.UserId == GetUserId())
```

アプリを起動して 'My Orders' に行くと何も注文が表示されません。原因は過去の注文データにはユーザー ID が含まれていないからです。新規に注文を作成して、意図したとおりに動作するか確認してください。また別の Twitter アカウントでログインした場合に、他ユーザーの注文が取得できない事も確認してみてください。

次のセッションは - [JavaScript interop](07-javascript-interop.md) です。
