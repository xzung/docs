# API Authentication (Passport)

- [Introduction](#introduction)
- [Installation](#installation)
    - [Frontend Quickstart](#frontend-quickstart)
- [Configuration](#configuration)
    - [Token Lifetimes](#token-lifetimes)
- [Issuing Access Tokens](#issuing-access-tokens)
    - [Managing Clients](#managing-clients)
    - [Requesting Tokens](#requesting-tokens)
    - [Refreshing Tokens](#refreshing-tokens)
- [Password Grant Tokens](#password-grant-tokens)
    - [Creating A Password Grant Client](#creating-a-password-grant-client)
    - [Requesting Tokens](#requesting-password-grant-tokens)
    - [Requesting All Scopes](#requesting-all-scopes)
- [Implicit Grant Tokens](#implicit-grant-tokens)
- [Client Credentials Grant Tokens](#client-credentials-grant-tokens)
- [Personal Access Tokens](#personal-access-tokens)
    - [Creating A Personal Access Client](#creating-a-personal-access-client)
    - [Managing Personal Access Tokens](#managing-personal-access-tokens)
- [Protecting Routes](#protecting-routes)
    - [Via Middleware](#via-middleware)
    - [Passing The Access Token](#passing-the-access-token)
- [Token Scopes](#token-scopes)
    - [Defining Scopes](#defining-scopes)
    - [Assigning Scopes To Tokens](#assigning-scopes-to-tokens)
    - [Checking Scopes](#checking-scopes)
- [Consuming Your API With JavaScript](#consuming-your-api-with-javascript)
- [Events](#events)
- [Testing](#testing)

<a name="introduction"></a>
## Introduction

Laravel already makes it easy to perform authentication via traditional login forms, but what about APIs? APIs typically use tokens to authenticate users and do not maintain session state between requests. Laravel makes API authentication a breeze using Laravel Passport, which provides a full OAuth2 server implementation for your Laravel application in a matter of minutes. Passport is built on top of the [League OAuth2 server](https://github.com/thephpleague/oauth2-server) that is maintained by Alex Bilbie.

laravel 已经很容易去执行传统的登录表单认证， 但是APIs 呢？ APIs 通常是使用tokens 去认证用户，并且在请求之间不维护session状态。 使用laravel passport 使API 认证变得很容易。

> {note} This documentation assumes you are already familiar with OAuth2. If you do not know anything about OAuth2, consider familiarizing yourself with the general terminology and features of OAuth2 before continuing.

<a name="installation"></a>
## Installation

To get started, install Passport via the Composer package manager:

    composer require laravel/passport

Next, register the Passport service provider in the `providers` array of your `config/app.php` configuration file:

    Laravel\Passport\PassportServiceProvider::class,

The Passport service provider registers its own database migration directory with the framework, so you should migrate your database after registering the provider. The Passport migrations will create the tables your application needs to store clients and access tokens:

Passport 服务驱动会注册它自己的数据库migration 目录，在laravel 框架中； 因此，在注册完驱动后，你应该migrate数据库。 passport migrations 将创建表用于存储所有clients以及access tokens.

    php artisan migrate

> {note} If you are not going to use Passport's default migrations, you should call the `Passport::ignoreMigrations` method in the `register` method of your `AppServiceProvider`. You may export the default migrations using `php artisan vendor:publish --tag=passport-migrations`.

我们也可以不使用passport 提供的migrations , 而是自己设计数据表。这需要在方法 `AppServiceProvider->register()`中调用方法`Passport::ignoreMigrations()`. 当然 ，我们也可以手动发布passport 的migrations 到框架中。
ps: 手动发布后的migrations 我们是可以进行修改的。

Next, you should run the `passport:install` command. This command will create the encryption keys needed to generate secure access tokens. In addition, the command will create "personal access" and "password grant" clients which will be used to generate access tokens:
下一步，你应该运行 `passport:install` 命令。
    php artisan passport:install

After running this command, add the `Laravel\Passport\HasApiTokens` trait to your `App\User` model. This trait will provide a few helper methods to your model which allow you to inspect the authenticated user's token and scopes:

    <?php

    namespace App;

    use Laravel\Passport\HasApiTokens;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Foundation\Auth\User as Authenticatable;

    class User extends Authenticatable
    {
        use HasApiTokens, Notifiable;
    }

Next, you should call the `Passport::routes` method within the `boot` method of your `AuthServiceProvider`. This method will register the routes necessary to issue access tokens and revoke access tokens, clients, and personal access tokens:

    <?php

    namespace App\Providers;

    use Laravel\Passport\Passport;
    use Illuminate\Support\Facades\Gate;
    use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

    class AuthServiceProvider extends ServiceProvider
    {
        /**
         * The policy mappings for the application.
         *
         * @var array
         */
        protected $policies = [
            'App\Model' => 'App\Policies\ModelPolicy',
        ];

        /**
         * Register any authentication / authorization services.
         *
         * @return void
         */
        public function boot()
        {
            $this->registerPolicies();

            Passport::routes();
        }
    }

Finally, in your `config/auth.php` configuration file, you should set the `driver` option of the `api` authentication guard to `passport`. This will instruct your application to use Passport's `TokenGuard` when authenticating incoming API requests:
使用passport的‘TokenGuard’ 来做API请求中的身份认证。

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
        ],
    ],

<a name="frontend-quickstart"></a>
### Frontend Quickstart

> {note} In order to use the Passport Vue components, you must be using the [Vue](https://vuejs.org) JavaScript framework. These components also use the Bootstrap CSS framework. However, even if you are not using these tools, the components serve as a valuable reference for your own frontend implementation.

Passport ships with a JSON API that you may use to allow your users to create clients and personal access tokens. However, it can be time consuming to code a frontend to interact with these APIs. So, Passport also includes pre-built [Vue](https://vuejs.org) components you may use as an example implementation or starting point for your own implementation.

passport 提供了自己的vue 前端组件，用于创建客户端 和 私人访问令牌。 当然我们也可以自己用js来写。 但这个vue 的实现可以作为很好的参考。

To publish the Passport Vue components, use the `vendor:publish` Artisan command:

    php artisan vendor:publish --tag=passport-components

The published components will be placed in your `resources/assets/js/components` directory. Once the components have been published, you should register them in your `resources/assets/js/app.js` file:

    Vue.component(
        'passport-clients',
        require('./components/passport/Clients.vue')
    );

    Vue.component(
        'passport-authorized-clients',
        require('./components/passport/AuthorizedClients.vue')
    );

    Vue.component(
        'passport-personal-access-tokens',
        require('./components/passport/PersonalAccessTokens.vue')
    );

After registering the components, make sure to run `npm run dev` to recompile your assets. Once you have recompiled your assets, you may drop the components into one of your application's templates to get started creating clients and personal access tokens:

    <passport-clients></passport-clients>
    <passport-authorized-clients></passport-authorized-clients>
    <passport-personal-access-tokens></passport-personal-access-tokens>

<a name="configuration"></a>
## Configuration

<a name="token-lifetimes"></a>
### Token Lifetimes

By default, Passport issues long-lived access tokens that never need to be refreshed. If you would like to configure a shorter token lifetime, you may use the `tokensExpireIn` and `refreshTokensExpireIn` methods. These methods should be called from the `boot` method of your `AuthServiceProvider`:

    use Carbon\Carbon;

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::tokensExpireIn(Carbon::now()->addDays(15));

        Passport::refreshTokensExpireIn(Carbon::now()->addDays(30));
    }

<a name="issuing-access-tokens"></a>
## Issuing Access Tokens

Using OAuth2 with authorization codes is how most developers are familiar with OAuth2. When using authorization codes, a client application will redirect a user to your server where they will either approve or deny the request to issue an access token to the client.

<a name="managing-clients"></a>
### Managing Clients

First, developers building applications that need to interact with your application's API will need to register their application with yours by creating a "client". Typically, this consists of providing the name of their application and a URL that your application can redirect to after users approve their request for authorization.

注册一个client , 第三方应用程序需要提供一个它们app的名称，以及一个用于重定向的url,当用户的请求被批准后认证服务器会重定向到这个URL

#### The `passport:client` Command

The simplest way to create a client is using the `passport:client` Artisan command. This command may be used to create your own clients for testing your OAuth2 functionality. When you run the `client` command, Passport will prompt you for more information about your client and will provide you with a client ID and secret:

    php artisan passport:client

#### JSON API

Since your users will not be able to utilize the `client` command, Passport provides a JSON API that you may use to create clients. This saves you the trouble of having to manually code controllers for creating, updating, and deleting clients.

使用passport 的JSONAPI 创建（注册）client, 相比在自己编写一个controller 并在其中创建，更新，删除 client 要省下很多麻烦。

However, you will need to pair Passport's JSON API with your own frontend to provide a dashboard for your users to manage their clients. Below, we'll review all of the API endpoints for managing clients. For convenience, we'll use [Axios](https://github.com/mzabriskie/axios) to demonstrate making HTTP requests to the endpoints.

不管怎么样，你需要把passport的JSON API 整合到你自己的前端中； 这个前端是认证服务方提供的一个后台面板页面，它给第三方应用程序方使用，第三方应用程序方通过这个面板页面来管理它们的client.  还记得云通讯的后台页面吗？它就是云通讯提供给学学看项目的后台面板，用于让学学看来管理它的client.

ps: "your own frontend" 指的是谁的前端？  这里有两方：认证服务器（也就是安装了passport组件，提供了oauth服务的当前laravel项目）,使用 oauth服务的第三方应用程序。 它指的是认证服务方，由认证服务方提供的dashboard页面。

> {tip} If you don't want to implement the entire client management frontend yourself, you can use the [frontend quickstart](#frontend-quickstart) to have a fully functional frontend in a matter of minutes.

passport 有一个现成的后台管理实现，我们可以自己写，也可以参考它的做法。

#### `GET /oauth/clients`

This route returns all of the clients for the authenticated user. This is primarily useful for listing all of the user's clients so that they may edit or delete them:

列出一个用户的所有的clients

ps： 这里的用户是指一个注册了oauth 服务的用户；就是说一个注册了的用户，可以有创建多个clients ; 使用passport的laravel 项目，要看做一个认证服务器，就是说这个项目是提供授权认证的，它不是资源服务器。因此，这个项目的用户，就是那些申请了认证服务的用户，而不是资源拥有者。

    axios.get('/oauth/clients')
        .then(response => {
            console.log(response.data);
        });

#### `POST /oauth/clients`

This route is used to create new clients. It requires two pieces of data: the client's `name` and a `redirect` URL. The `redirect` URL is where the user will be redirected after approving or denying a request for authorization.

该路由用于创建一个新的客户端，它需要两个参数：name 和 redirect_uri.    redirect_uri 将用于给认证服务器做回调，通知第三方应用程序认证授权是否被同意。

When a client is created, it will be issued a client ID and client secret. These values will be used when requesting access tokens from your application. The client creation route will return the new client instance:

每个client 都会被分配一个client_id ,secret; 后面我们的第三方程序请求 access token 的时候，需要带上它们。

    const data = {
        name: 'Client Name',
        redirect: 'http://example.com/callback'
    };

    axios.post('/oauth/clients', data)
        .then(response => {
            console.log(response.data);
        })
        .catch (response => {
            // List errors on response...
        });

#### `PUT /oauth/clients/{client-id}`

This route is used to update clients. It requires two pieces of data: the client's `name` and a `redirect` URL. The `redirect` URL is where the user will be redirected after approving or denying a request for authorization. The route will return the updated client instance:

    const data = {
        name: 'New Client Name',
        redirect: 'http://example.com/callback'
    };

    axios.put('/oauth/clients/' + clientId, data)
        .then(response => {
            console.log(response.data);
        })
        .catch (response => {
            // List errors on response...
        });

#### `DELETE /oauth/clients/{client-id}`

This route is used to delete clients:

    axios.delete('/oauth/clients/' + clientId)
        .then(response => {
            //
        });

<a name="requesting-tokens"></a>
### Requesting Tokens

#### Redirecting For Authorization

Once a client has been created, developers may use their client ID and secret to request an authorization code and access token from your application. First, the consuming application should make a redirect request to your application's `/oauth/authorize` route like so:
一旦，一个client 被创建，开发者可能使用它们的client ID 和 secret 去向你的认证服务器请求一个 authorization code 和 access token 。
首先，正在消费的应用程序重定向跳转到认证服务器的 `/oauth/authorize` 路由上。

ps:'consuming application '一般情况下是指浏览器 。

    Route::get('/redirect', function () {
        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://example.com/callback',
            'response_type' => 'code',
            'scope' => '',
        ]);

        return redirect('http://your-app.com/oauth/authorize?'.$query);
    });

> {tip} Remember, the `/oauth/authorize` route is already defined by the `Passport::routes` method. You do not need to manually define this route.
ps: `/oauth/authorize` 已经在`Passport::routes` 中被定义，我们不必手动去设置它。


#### Approving The Request

When receiving authorization requests, Passport will automatically display a template to the user allowing them to approve or deny the authorization request. If they approve the request, they will be redirected back to the `redirect_uri` that was specified by the consuming application. The `redirect_uri` must match the `redirect` URL that was specified when the client was created.

当接收到一个认证请求时，Passport 将自动显示一个模板页面，让用户（资源拥有者）去同意或拒绝第三方应用程序的认证请求。如果用户同意请求，在消费应用程序中他们将被重定向到‘redirect_uri’ 指定的页面。ps: 消费应用程序就是浏览器

If you would like to customize the authorization approval screen, you may publish Passport's views using the `vendor:publish` Artisan command. The published views will be placed in `resources/views/vendor/passport`:

如果你想自定义认证同意界面， 你可以发布passport 的视图文件到 `resources/views/vendor/passport`中， 然后我们可以修改这个视图文件

    php artisan vendor:publish --tag=passport-views

#### Converting Authorization Codes To Access Tokens

If the user approves the authorization request, they will be redirected back to the consuming application. The consumer should then issue a `POST` request to your application to request an access token. The request should include the authorization code that was issued by your application when the user approved the authorization request. In this example, we'll use the Guzzle HTTP library to make the `POST` request:
如果用户（资源拥有者）同意认证请求， 它们将会被重定向会正在消费应用程序（浏览器）。消费者（第三方应用程序后台服务）应该已经发送一个post 请求到我们的认证服务器，为了请求一个access token. 这个请求应该包含 authorziation code .

    Route::get('/callback', function (Request $request) {
        $http = new GuzzleHttp\Client;

        $response = $http->post('http://your-app.com/oauth/token', [
            'form_params' => [
                'grant_type' => 'authorization_code',
                'client_id' => 'client-id',
                'client_secret' => 'client-secret',
                'redirect_uri' => 'http://example.com/callback',
                'code' => $request->code,
            ],
        ]);

        return json_decode((string) $response->getBody(), true);
    });

This `/oauth/token` route will return a JSON response containing `access_token`, `refresh_token`, and `expires_in` attributes. The `expires_in` attribute contains the number of seconds until the access token expires.

`/oauth/token` 将会返回一个json 响应，其中包含`access_token`, `refresh_token`, and `expires_in`。 expires_in 单位为秒

> {tip} Like the `/oauth/authorize` route, the `/oauth/token` route is defined for you by the `Passport::routes` method. There is no need to manually define this route.

注意： `/oauth/authorize` 和 `/oauth/token` 都是在 `Passport::routes` 中被定义的，不需要我们手动定义它们。

<a name="refreshing-tokens"></a>
### Refreshing Tokens

If your application issues short-lived access tokens, users will need to refresh their access tokens via the refresh token that was provided to them when the access token was issued. In this example, we'll use the Guzzle HTTP library to refresh the token:

access token过期，重新申请access token ,很简单，使用下面请求，就可以方便重新申请access token 了,  注意 grant_type 值是 refresh_token

    $http = new GuzzleHttp\Client;

    $response = $http->post('http://your-app.com/oauth/token', [
        'form_params' => [
            'grant_type' => 'refresh_token',
            'refresh_token' => 'the-refresh-token',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'scope' => '',
        ],
    ]);

    return json_decode((string) $response->getBody(), true);

This `/oauth/token` route will return a JSON response containing `access_token`, `refresh_token`, and `expires_in` attributes. The `expires_in` attribute contains the number of seconds until the access token expires.

<a name="password-grant-tokens"></a>
## Password Grant Tokens

The OAuth2 password grant allows your other first-party clients, such as a mobile application, to obtain an access token using an e-mail address / username and password. This allows you to issue access tokens securely to your first-party clients without requiring your users to go through the entire OAuth2 authorization code redirect flow.

OAuth2.0 密码授权允许你的其他的第一方客户端，如一个移动应用程序，使用一个email或用户以及密码来获取一个access token.这将允许你安全的颁发access token 给你的第一方客户端，而不必要求你的用户名去走完整的oauth 认证码重定向流程。

ps: first-party 可以理解为自家的，第一方的，就是说我有了OAuth2.0服务的api 项目，我还有了使用它的客户端程序

<a name="creating-a-password-grant-client"></a>
### Creating A Password Grant Client

Before your application can issue tokens via the password grant, you will need to create a password grant client. You may do this using the `passport:client` command with the `--password` option. If you have already run the `passport:install` command, you do not need to run this command:

在你的 application 能够通过密码模式发布token 之前，你需要先去创建一个密码模式client。你可以在oauth 服务端使用  `passport:client --password` 令来创建密码模式的客户端。 如果你已经运行过了  `passport:install`  令，你将不需要运行这个命令。


    php artisan passport:client --password

<a name="requesting-password-grant-tokens"></a>
### Requesting Tokens

Once you have created a password grant client, you may request an access token by issuing a `POST` request to the `/oauth/token` route with the user's email address and password. Remember, this route is already registered by the `Passport::routes` method so there is no need to define it manually. If the request is successful, you will receive an `access_token` and `refresh_token` in the JSON response from the server:

一旦你创建一个密码模式客户端，你可以使用一个post请求发送用户名以及密码到路由 /oauth/token 来获取一个access token .记住，这个路由已经被Passport:routes 方法注册，因此，不需要手动定义它。如果请求成功，你将接收到一个access_token 和 refresh_token .

    $http = new GuzzleHttp\Client;

    $response = $http->post('http://your-app.com/oauth/token', [
        'form_params' => [
            'grant_type' => 'password',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'username' => 'taylor@laravel.com',
            'password' => 'my-password',
            'scope' => '',
        ],
    ]);

    return json_decode((string) $response->getBody(), true);

> {tip} Remember, access tokens are long-lived by default. However, you are free to [configure your maximum access token lifetime](#configuration) if needed.
注意： access token 默认是长期有效的。你也可以去access token 的默认有效期。

<a name="requesting-all-scopes"></a>
### Requesting All Scopes

When using the password grant, you may wish to authorize the token for all of the scopes supported by your application. You can do this by requesting the `*` scope. If you request the `*` scope, the `can` method on the token instance will always return `true`. This scope may only be assigned to a token that is issued using the `password` grant:

当使用密码模式的时候，你可能希望在你的app支持的所有范围领域中去验证这个token； 你能够使用`*` 范围值。 如果你请求 范围值，在token 实例中的can方法将返回 true .这个scope值仅仅在使用密码模式发布的token 上才设置。

ps:your application 是指的提供资源的后台app吗？应该是的


    $response = $http->post('http://your-app.com/oauth/token', [
        'form_params' => [
            'grant_type' => 'password',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'username' => 'taylor@laravel.com',
            'password' => 'my-password',
            'scope' => '*',
        ],
    ]);

<a name="implicit-grant-tokens"></a>
## Implicit Grant Tokens

The implicit grant is similar to the authorization code grant; however, the token is returned to the client without exchanging an authorization code. This grant is most commonly used for JavaScript or mobile applications where the client credentials can't be securely stored. To enable the grant, call the `enableImplicitGrant` method in your `AuthServiceProvider`:

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();

        Passport::enableImplicitGrant();
    }

Once a grant has been enabled, developers may use their client ID to request an access token from your application. The consuming application should make a redirect request to your application's `/oauth/authorize` route like so:

    Route::get('/redirect', function () {
        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://example.com/callback',
            'response_type' => 'token',
            'scope' => '',
        ]);

        return redirect('http://your-app.com/oauth/authorize?'.$query);
    });

> {tip} Remember, the `/oauth/authorize` route is already defined by the `Passport::routes` method. You do not need to manually define this route.

<a name="client-credentials-grant-tokens"></a>
## Client Credentials Grant Tokens

The client credentials grant is suitable for machine-to-machine authentication. For example, you might use this grant in a scheduled job which is performing maintenance tasks over an API. To retrieve a token, make a request to the `oauth/token` endpoint:

    $guzzle = new GuzzleHttp\Client;

    $response = $guzzle->post('http://your-app.com/oauth/token', [
        'form_params' => [
            'grant_type' => 'client_credentials',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'scope' => 'your-scope',
        ],
    ]);

    echo json_decode((string) $response->getBody(), true);

<a name="personal-access-tokens"></a>
## Personal Access Tokens

Sometimes, your users may want to issue access tokens to themselves without going through the typical authorization code redirect flow. Allowing users to issue tokens to themselves via your application's UI can be useful for allowing users to experiment with your API or may serve as a simpler approach to issuing access tokens in general.

ps： your users 是谁？是注册了oauth 服务了的用户，还记得上面讲到，用户可以有多个client吗？ 一个用户有多个app 要调用api  ,则这个用户就可以注册多个client. 但如果，用户想测试一下我们的api , 又不想使用正式的注册了的client走正式的申请access token流程来获取一个access token ,这个时候就可以使用passport的私人访问令牌功能.它可以通过认证服务器提供的UI（应该是就是上面讲的认证服务器为它的用户提供的client管理后台）来为用户自己签发一个person access token. person access token 和从正式申请流程获取到access token 具有相同的效果，用户可以把它作为正式的access token 来使用，不同的是它的产生不是通过正式的access token 申请流程获得，它是passport 提供的格外功能，不是OAuth2.0 结构的一部分。

> {note} Personal access tokens are always long-lived. Their lifetime is not modified when using the `tokensExpireIn` or `refreshTokensExpireIn` methods.

注意：person access token 是永久有效的，这点和正式access token 是不一样的，毕竟它是做调试使用的。

<a name="creating-a-personal-access-client"></a>
### Creating A Personal Access Client

Before your application can issue personal access tokens, you will need to create a personal access client. You may do this using the `passport:client` command with the `--personal` option. If you have already run the `passport:install` command, you do not need to run this command:
使用person access token 之前，需要先在认证服务器创建一个 personal access client. 

ps: 之前已经讲过 php artisan passport:client 命令就是为了创建测试客户端的。

     php artisan passport:client --personal

<a name="managing-personal-access-tokens"></a>
### Managing Personal Access Tokens

Once you have created a personal access client, you may issue tokens for a given user using the `createToken` method on the `User` model instance. The `createToken` method accepts the name of the token as its first argument and an optional array of [scopes](#token-scopes) as its second argument:
一旦，你已经创建了一个personal access client, 你可以在User Model 实例中使用 createToken 方法为一个用户创建person access token . 该方法接收两个参数：token name , scopes

    $user = App\User::find(1);

    // Creating a token without scopes...
    $token = $user->createToken('Token Name')->accessToken;

    // Creating a token with scopes...
    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

#### JSON API

Passport also includes a JSON API for managing personal access tokens. You may pair this with your own frontend to offer your users a dashboard for managing personal access tokens. Below, we'll review all of the API endpoints for managing personal access tokens. For convenience, we'll use [Axios](https://github.com/mzabriskie/axios) to demonstrate making HTTP requests to the endpoints.

passport 也提供一个用于管理personal access token的JSON API. 你可以整合它到你的前端面板（认证服务器提供给用户的后台client管理页面,上面已经讲过它）中-- 做法和上面讲到的管理client 的JSON API 是一样的。 
> {tip} If you don't want to implement the personal access token frontend yourself, you can use the [frontend quickstart](#frontend-quickstart) to have a fully functional frontend in a matter of minutes.如果你不想去实现自己的用于管理personal access token 的前端， 你可以使用passport 现成的。

#### `GET /oauth/scopes`

This route returns all of the [scopes](#token-scopes) defined for your application. You may use this route to list the scopes a user may assign to a personal access token:

    axios.get('/oauth/scopes')
        .then(response => {
            console.log(response.data);
        });

#### `GET /oauth/personal-access-tokens`

This route returns all of the personal access tokens that the authenticated user has created. This is primarily useful for listing all of the user's token so that they may edit or delete them:

    axios.get('/oauth/personal-access-tokens')
        .then(response => {
            console.log(response.data);
        });

#### `POST /oauth/personal-access-tokens`

This route creates new personal access tokens. It requires two pieces of data: the token's `name` and the `scopes` that should be assigned to the token:

    const data = {
        name: 'Token Name',
        scopes: []
    };

    axios.post('/oauth/personal-access-tokens', data)
        .then(response => {
            console.log(response.data.accessToken);
        })
        .catch (response => {
            // List errors on response...
        });

#### `DELETE /oauth/personal-access-tokens/{token-id}`

This route may be used to delete personal access tokens:

    axios.delete('/oauth/personal-access-tokens/' + tokenId);

<a name="protecting-routes"></a>
## Protecting Routes
保护路由， 使用passport 保护路由，即验证http 请求的access token 是否合法。
到这里使用access token 的请求就不在是做oauth认证访问了，而是访问用户资源。
一般情况是去资源服务器访问资源，去认证服务器做oauth 权限申请。
但从本文档的描述来看，认证服务和资源访问服务（即数据api 服务）是写到一个项目里的。 
且passport 把对认证服务的访问都通过它提供的 JSON API ，以及指定路由（如：`/oauth/authorize` ， `/oauth/token`）从laravel 路由中隔离开了。
这样访问认证服务的http请求就不用受保护路由的干扰了。访问用户资源的http 请求走的常规框架路由，两类请求不会相互影响。

这里讲的protectiing routes 也是针对的访问用户资源的http请求走的路由。

<a name="via-middleware"></a>
### Via Middleware

Passport includes an [authentication guard](/docs/{{version}}/authentication#adding-custom-guards) that will validate access tokens on incoming requests. Once you have configured the `api` guard to use the `passport` driver, you only need to specify the `auth:api` middleware on any routes that require a valid access token:

passport 包含一个 authentication guard .它将验证在当前请求中的access token。一旦你使用了 'passport'  driver  配置了api guard, 你仅仅需要为路由指定一个中间件就可以验证access token 了。  -- config/auth.php 中配置

    Route::get('/user', function () {
        //
    })->middleware('auth:api');

<a name="passing-the-access-token"></a>
### Passing The Access Token

When calling routes that are protected by Passport, your application's API consumers should specify their access token as a `Bearer` token in the `Authorization` header of their request. For example, when using the Guzzle HTTP library:
当正在调用的路由已经被passport 保护的时候，我们的andriod/os/... 应用程序应该指定它们的access token 作为一个 ‘Bearer’ token ,在他们的请求的 `Authorization`消息头中。


    $response = $client->request('GET', '/api/user', [
        'headers' => [
            'Accept' => 'application/json',
            'Authorization' => 'Bearer '.$accessToken,
        ],
    ]);

<a name="token-scopes"></a>
## Token Scopes


<a name="defining-scopes"></a>
### Defining Scopes

Scopes allow your API clients to request a specific set of permissions when requesting authorization to access an account. For example, if you are building an e-commerce application, not all API consumers will need the ability to place orders. Instead, you may allow the consumers to only request authorization to access order shipment statuses. In other words, scopes allow your application's users to limit the actions a third-party application can perform on their behalf.

You may define your API's scopes using the `Passport::tokensCan` method in the `boot` method of your `AuthServiceProvider`. The `tokensCan` method accepts an array of scope names and scope descriptions. The scope description may be anything you wish and will be displayed to users on the authorization approval screen:

    use Laravel\Passport\Passport;

    Passport::tokensCan([
        'place-orders' => 'Place orders',
        'check-status' => 'Check order status',
    ]);

<a name="assigning-scopes-to-tokens"></a>
### Assigning Scopes To Tokens

#### When Requesting Authorization Codes

When requesting an access token using the authorization code grant, consumers should specify their desired scopes as the `scope` query string parameter. The `scope` parameter should be a space-delimited list of scopes:

    Route::get('/redirect', function () {
        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://example.com/callback',
            'response_type' => 'code',
            'scope' => 'place-orders check-status',
        ]);

        return redirect('http://your-app.com/oauth/authorize?'.$query);
    });

#### When Issuing Personal Access Tokens

If you are issuing personal access tokens using the `User` model's `createToken` method, you may pass the array of desired scopes as the second argument to the method:

    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="checking-scopes"></a>
### Checking Scopes

Passport includes two middleware that may be used to verify that an incoming request is authenticated with a token that has been granted a given scope. To get started, add the following middleware to the `$routeMiddleware` property of your `app/Http/Kernel.php` file:

    'scopes' => \Laravel\Passport\Http\Middleware\CheckScopes::class,
    'scope' => \Laravel\Passport\Http\Middleware\CheckForAnyScope::class,

#### Check For All Scopes

The `scopes` middleware may be assigned to a route to verify that the incoming request's access token has *all* of the listed scopes:

    Route::get('/orders', function () {
        // Access token has both "check-status" and "place-orders" scopes...
    })->middleware('scopes:check-status,place-orders');

#### Check For Any Scopes

The `scope` middleware may be assigned to a route to verify that the incoming request's access token has *at least one* of the listed scopes:

    Route::get('/orders', function () {
        // Access token has either "check-status" or "place-orders" scope...
    })->middleware('scope:check-status,place-orders');

#### Checking Scopes On A Token Instance

Once an access token authenticated request has entered your application, you may still check if the token has a given scope using the `tokenCan` method on the authenticated `User` instance:

    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        if ($request->user()->tokenCan('place-orders')) {
            //
        }
    });

<a name="consuming-your-api-with-javascript"></a>
## Consuming Your API With JavaScript

When building an API, it can be extremely useful to be able to consume your own API from your JavaScript application. This approach to API development allows your own application to consume the same API that you are sharing with the world. The same API may be consumed by your web application, mobile applications, third-party applications, and any SDKs that you may publish on various package managers.

Typically, if you want to consume your API from your JavaScript application, you would need to manually send an access token to the application and pass it with each request to your application. However, Passport includes a middleware that can handle this for you. All you need to do is add the `CreateFreshApiToken` middleware to your `web` middleware group:

    'web' => [
        // Other middleware...
        \Laravel\Passport\Http\Middleware\CreateFreshApiToken::class,
    ],

This Passport middleware will attach a `laravel_token` cookie to your outgoing responses. This cookie contains an encrypted JWT that Passport will use to authenticate API requests from your JavaScript application. Now, you may make requests to your application's API without explicitly passing an access token:

    axios.get('/user')
        .then(response => {
            console.log(response.data);
        });

When using this method of authentication, Axios will automatically send the `X-CSRF-TOKEN` header. In addition, the default Laravel JavaScript scaffolding instructs Axios to send the `X-Requested-With` header:

    window.axios.defaults.headers.common = {
        'X-Requested-With': 'XMLHttpRequest',
    };

> {note} If you are using a different JavaScript framework, you should make sure it is configured to send the `X-CSRF-TOKEN` and `X-Requested-With` headers with every outgoing request.


<a name="events"></a>
## Events

Passport raises events when issuing access tokens and refresh tokens. You may use these events to prune or revoke other access tokens in your database. You may attach listeners to these events in your application's `EventServiceProvider`:

```php
/**
 * The event listener mappings for the application.
 *
 * @var array
 */
protected $listen = [
    'Laravel\Passport\Events\AccessTokenCreated' => [
        'App\Listeners\RevokeOldTokens',
    ],

    'Laravel\Passport\Events\RefreshTokenCreated' => [
        'App\Listeners\PruneOldTokens',
    ],
];
```

<a name="testing"></a>
## Testing

Passport's `actingAs` method may be used to specify the currently authenticated user as well as its scopes. The first argument given to the `actingAs` method is the user instance and the second is an array of scopes that should be granted to the user's token:

    public function testServerCreation()
    {
        Passport::actingAs(
            factory(User::class)->create(),
            ['create-servers']
        );

        $response = $this->post('/api/create-server');

        $response->assertStatus(200);
    }
