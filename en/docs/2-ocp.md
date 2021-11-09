# 2 - Open-closed Principle

The 'Open Closed Principle' reflects to a part of the code that needs to be implemented, but without to
change the code that is already written.

The idea is to let the code functions more generic, applying interfaces, with predefined functions where the polymorphism become the main attractive to develop with this principle.

<center>
    "Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification."
</center>

Let's say that you have three routes to OAuth authentication (3rd party platforms), where you want your
app have the possibility to auth with Twitch, Github and Spotify.

Inside the router, you'll create three different routes, one to each type of authentication (since they go to
different services).


```php
// routes/web.php
Route::get('auth/oauth/discord', [AuthController:: class,'getDiscordAuth']);
Route::get('auth/oauth/twitch', [AuthController:: class,'getTwitchAuth']);
Route::get('auth/oauth/github', [AuthController:: class,'getGithubAuth']);
```


```php
// app/Http/Controllers/AuthController.php
class AuthController {

    private $repository;

    public function __construct(AuthRepository $repository)
    {
        $this->repository = $repository;
    }

    public function getDiscordAuth(Request $request)
    {
        try {
            $result = $this->repository->discordAuth($request->input('code'));
            return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }

    public function getTwitchAuth(Request $request)
    {
        try {
            $result = $this->repository->twitchAuth($request->input('code'));
            return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }

    public function getGithubAuth(Request $request)
    {
        try {
            $result = $this->repository->githubAuth($request->input('code'));
            return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }
}
```

```php
class AuthRepository {

    public function discordAuth(string $code)
    {
        $service = new DiscordService();
        $authData = $service->authWithDiscord($code);

        $response = $authService->getDiscordUser($authData['access_token']);
        $authUser = $this->findOrCreate('discord',$response);

        Auth::user($authUser);
        return true;
    }

    public function twitchAuth(string $code)
    {
        $service = new TwitchService();
        $authData = $service->authWithTwitch($code);

        $response = $authService->getTwitchUser($authData['access_token']);
        $authUser = $this->findOrCreate('twitch',$response);

        Auth::user($authUser);
        return true;
    }

    public function githubAuth(string $code)
    {
        $service = new GithubService();
        $authData = $service->authWithGithub($code);

        $response = $authService->getGithubUser($authData['access_token']);
        $authUser = $this->findOrCreate('github',$response);

        Auth::user($authUser);
        return true;
    }

    public function findOrCreate(string $provider, $providerData): User
    {
        $auth = User::where('email', $providerData['email'])->first();
        if (!$auth) {
            return User::create([
                'name' => $providerData['name'],
                'email' => $providerData['email'],
                $provider . "_id" => $providerData['id'],
            ]);
        }

        if (empty($auth->{$provider . "_id"})) {
            $auth->update([
                $provider . "_id" => $providerData['id']
            ]);
            return $auth;
        }

        if ($auth->{$provider . "_id"} == $providerData['id']) {
            return $auth;
        }

        throw new \Exception('deu ruim');
    }

}
```

If you read the snippets above, you will notice that has a pattern that we can follow to improve the code.
Like, the OAuth itself is a pattern, you have the same requests and types of responses but our code doesn't understand that YET.

The way that was written WORKS but is pretty hard to maintain this code. Let's rewrite all that applying OCP.
Let's summarize these three routes in one, and it should look like this:

```php
// routes/web.php
Route::get('auth/oauth/{provider}', [AuthController:: class, 'getOAuth']);
```

Only changing this route prefix, you can already understant that we're going to make things more generic having in sight that has a pattern. Now we're going to change our controller to support these changes:

```php
// app/Http/Controllers/AuthController.php
class AuthController {

    private $repository;

    public function __construct(AuthRepository $repository)
    {
        $this->repository = $repository;
    }

    public function getOAuth(Request $request, string $provider)
    {
        try {
            $result = $this->repository->authenticateOAuth($provider,$request->input('code'));
            return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }
}
```

We pass the provider that we want to consume into our repository and now we have a issue to make the repository understand which one of the 3/N it needs to call. 

Now let's analyze the functions/methods from service that is being called in the Repository:

```php
// GithubService
$service = new GithubService();
$authData = $service->authWithGithub($code);
$response = $authService->getGithubUser($authData['access_token']);

// DiscordService
$service = new DiscordService();
$authData = $service->authWithDiscord($code);
$response = $authService->getDiscordUser($authData['access_token']);

// TwitchService
$service = new TwitchService();
$authData = $service->authWithTwitch($code);
$response = $authService->getTwitchUser($authData['access_token']);
```

As we can see there's a pattern, but the names of the functions is intuitives but not generics. Now, if we stop and create an **INTERFACE**, it changes completely.

Let's name our interface as OAuthContract with the following methods:

```php
interface OAuthContract {

    public function auth(string $code);

    public function getAuthenticatedUser(string $accessToken);
}
```

If we can standardize the methods, all we have to do is to find a way to call a Service that have this Interface, because we will guarantee that the methods are implemented. Look at this:


```php
// GithubService
$service = new GithubService();
$authData = $service->auth($code);
$response = $authService->getAuthenticatedUser($authData['access_token']);

// DiscordService
$service = new DiscordService();
$authData = $service->auth($code);
$response = $authService->getAuthenticatedUser($authData['access_token']);

// TwitchService
$service = new TwitchService();
$authData = $service->auth($code);
$response = $authService->getAuthenticatedUser($authData['access_token']);
```

Now to finish, we need to tell to our Repository that has a polymorphic method/class trying to be called, and the correct way to do it is typing the return of this function with the **INTERFACE**. Then you will return: 

```php

public function getProvider(string $provider): OAuthContract
{
    return match($provider) {
        'discord' => new DiscordService(),
        'twitch' => new TwitchService(),
        'github' => new GithubService()
    };
}
```

Just keep in mind that you can return class/interfaces/types. You can understand that you will return a class that have the Interface OAuthContract implemented that will force those generic methods being implemented. If you try to pass other class that doesn't have this interface implemented, it not going to work.


Now, lets refactor the Repository to receive this generic change inside our project:

```php
class AuthRepository {

    public function authenticateOAuth(string $provider, string $code): bool
    {
        $service = $this->getProvider($provider);
        $authData = $service->auth($code);

        $response = $authService->getAuthenticatedUser($authData['access_token']);
        $authUser = $this->findOrCreate($provider, $response);

        Auth::user($authUser);
        return true;
    }

    public function findOrCreate(string $provider, $providerData): User
    {
        $auth = User::where('email', $providerData['email'])->first();
        if (!$auth) {
            return User::create([
                'name' => $providerData['name'],
                'email' => $providerData['email'],
                $provider . "_id" => $providerData['id'],
            ]);
        }

        if (empty($auth->{$provider . "_id"})) {
            $auth->update([
                $provider . "_id" => $providerData['id']
            ]);
            return $auth;
        }

        if ($auth->{$provider . "_id"} == $providerData['id']) {
            return $auth;
        }

        throw new \Exception('Something Wrong');
    }

    public function getProvider(string $provider): OAuthContract
    {
        return match($provider) {
            'discord' => new DiscordService(),
            'twitch' => new TwitchService(),
            'github' => new GithubService()
        };
    }
}
```

Your software is opened to extend more OAuth Services, but closed for modification! Congratz you finished this principle.

If your Services are working, you'll not need to modify it. But in the case that you want to implement a new  OAuth provider, you'll need to create a new Service Class like **GoogleService** and implement the **OAuthInterface** and add it to the match expression on the function **getProvider()** inside your repository and that's it. Open for extension but close for modification.

[3. Go to 'Liskov's Substitution Principle'](3-lsp.md)
