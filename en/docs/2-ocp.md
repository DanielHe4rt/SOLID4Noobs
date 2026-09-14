# 2 - Open-Closed Principle

The Open-Closed Principle says that you should be able to add new behavior to your code without changing the code that is already written.

The idea is to make the code generic by using interfaces. Each interface defines a set of methods, and polymorphism becomes the star of the show.

> "Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification."

Let's say your platform has OAuth login (third-party platforms) with Discord, Twitch, and GitHub.

To do that, you create three different routes, one for each provider (since each one talks to a different service):

```php
// routes/web.php
Route::get('auth/oauth/discord', [AuthController::class, 'discord']);
Route::get('auth/oauth/twitch', [AuthController::class, 'twitch']);
Route::get('auth/oauth/github', [AuthController::class, 'github']);
```

```php
// app/Http/Controllers/AuthController.php
final class AuthController extends Controller
{
    public function __construct(
        private readonly AuthService $authService,
    ) {}

    public function discord(Request $request): JsonResponse
    {
        $user = $this->authService->loginWithDiscord($request->query('code'));

        return response()->json($user);
    }

    public function twitch(Request $request): JsonResponse
    {
        $user = $this->authService->loginWithTwitch($request->query('code'));

        return response()->json($user);
    }

    public function github(Request $request): JsonResponse
    {
        $user = $this->authService->loginWithGithub($request->query('code'));

        return response()->json($user);
    }
}
```

```php
// app/Services/AuthService.php
final class AuthService
{
    public function loginWithDiscord(string $code): User
    {
        $client = new DiscordClient();
        $accessToken = $client->authWithDiscord($code);
        $discordUser = $client->getDiscordUser($accessToken);

        $user = $this->findOrCreateUser('discord', $discordUser);
        Auth::login($user);

        return $user;
    }

    public function loginWithTwitch(string $code): User
    {
        $client = new TwitchClient();
        $accessToken = $client->authWithTwitch($code);
        $twitchUser = $client->getTwitchUser($accessToken);

        $user = $this->findOrCreateUser('twitch', $twitchUser);
        Auth::login($user);

        return $user;
    }

    public function loginWithGithub(string $code): User
    {
        $client = new GithubClient();
        $accessToken = $client->authWithGithub($code);
        $githubUser = $client->getGithubUser($accessToken);

        $user = $this->findOrCreateUser('github', $githubUser);
        Auth::login($user);

        return $user;
    }

    private function findOrCreateUser(string $provider, array $providerUser): User
    {
        $providerColumn = "{$provider}_id";
        $user = User::query()->firstWhere('email', $providerUser['email']);

        if ($user === null) {
            return User::query()->create([
                'name' => $providerUser['name'],
                'email' => $providerUser['email'],
                $providerColumn => $providerUser['id'],
            ]);
        }

        if ($user->{$providerColumn} === null) {
            $user->update([$providerColumn => $providerUser['id']]);

            return $user;
        }

        $isSameAccount = (string) $user->{$providerColumn} === (string) $providerUser['id'];

        if (! $isSameAccount) {
            throw new AuthenticationException('This email is already linked to another account.');
        }

        return $user;
    }
}
```

If you read the snippets above, you'll notice a pattern we can use to improve the code. The OAuth flow is **generic**: every provider exchanges a `code` for an access token and then returns the user data. But our code doesn't understand that YET.

The way it's written, it WORKS. But for every new provider, you have to touch the routes, the controller, and the service. Let's rewrite it applying OCP.

Let's merge these three routes into one:

```php
// routes/web.php
Route::get('auth/oauth/{provider}', [AuthController::class, 'login']);
```

Just by changing this route, you can already tell that we're going to make things more generic, since there is a pattern. Now let's update our controller to support this change:

```php
// app/Http/Controllers/AuthController.php
final class AuthController extends Controller
{
    public function __construct(
        private readonly AuthService $authService,
    ) {}

    public function login(Request $request, string $provider): JsonResponse
    {
        $user = $this->authService->login($provider, $request->query('code'));

        return response()->json($user);
    }
}
```

The controller passes the provider to the service, and now the service has to figure out which of the 3 (or N) clients it needs to call.

Now let's analyze the client methods that the service calls:

```php
// DiscordClient
$accessToken = $client->authWithDiscord($code);
$providerUser = $client->getDiscordUser($accessToken);

// TwitchClient
$accessToken = $client->authWithTwitch($code);
$providerUser = $client->getTwitchUser($accessToken);

// GithubClient
$accessToken = $client->authWithGithub($code);
$providerUser = $client->getGithubUser($accessToken);
```

There's a pattern, but the method names, although intuitive, are not generic. Now, if we stop and create an **INTERFACE**, everything changes.

Let's name our interface `OAuthContract`, with the following methods:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): array;
}
```

If every client implements this interface, PHP will GUARANTEE that the methods exist. Then it doesn't matter which client shows up. Look at this:

```php
final class DiscordClient implements OAuthContract { /* ... */ }
final class TwitchClient implements OAuthContract { /* ... */ }
final class GithubClient implements OAuthContract { /* ... */ }

$accessToken = $client->getAccessToken($code);
$providerUser = $client->getAuthenticatedUser($accessToken);
```

Now we need to tell the service which client to use. The right way to do it is to type the method's return with the **INTERFACE**:

```php
private function resolveClient(string $provider): OAuthContract
{
    return match ($provider) {
        'discord' => new DiscordClient(),
        'twitch' => new TwitchClient(),
        'github' => new GithubClient(),
        default => throw new InvalidArgumentException("Unsupported OAuth provider: {$provider}"),
    };
}
```

Since the return type is `OAuthContract`, the method can only return a class that implements this interface, and that's what forces those generic methods to exist. If you try to return a class without the interface, it will blow up (with a nice `TypeError`).

Now let's refactor the service to receive this change:

```php
// app/Services/AuthService.php
final class AuthService
{
    public function login(string $provider, string $code): User
    {
        $client = $this->resolveClient($provider);

        $accessToken = $client->getAccessToken($code);
        $providerUser = $client->getAuthenticatedUser($accessToken);

        $user = $this->findOrCreateUser($provider, $providerUser);
        Auth::login($user);

        return $user;
    }

    private function resolveClient(string $provider): OAuthContract
    {
        return match ($provider) {
            'discord' => new DiscordClient(),
            'twitch' => new TwitchClient(),
            'github' => new GithubClient(),
            default => throw new InvalidArgumentException("Unsupported OAuth provider: {$provider}"),
        };
    }

    private function findOrCreateUser(string $provider, array $providerUser): User
    {
        // same as the previous example
    }
}
```

## Closing it for modification for real

Notice one detail: to add Google, you would still have to open `AuthService` and change the `match`. In other words, it isn't 100% closed for modification yet.

To fix that, we move the list of clients out of the service and let the Laravel container hand it over ready to use:

```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Services\AuthService;
use App\Services\OAuth\DiscordClient;
use App\Services\OAuth\GithubClient;
use App\Services\OAuth\TwitchClient;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;

final class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->when(AuthService::class)
            ->needs('$clients')
            ->give(fn (Application $app): array => [
                'discord' => $app->make(DiscordClient::class),
                'twitch' => $app->make(TwitchClient::class),
                'github' => $app->make(GithubClient::class),
            ]);
    }
}
```

```php
// app/Services/AuthService.php
namespace App\Services;

use App\Models\User;
use App\Services\OAuth\OAuthContract;
use Illuminate\Support\Facades\Auth;
use InvalidArgumentException;

final class AuthService
{
    /**
     * @param array<string, OAuthContract> $clients
     */
    public function __construct(
        private readonly array $clients,
    ) {}

    public function login(string $provider, string $code): User
    {
        $client = $this->clients[$provider]
            ?? throw new InvalidArgumentException("Unsupported OAuth provider: {$provider}");

        $accessToken = $client->getAccessToken($code);
        $providerUser = $client->getAuthenticatedUser($accessToken);

        $user = $this->findOrCreateUser($provider, $providerUser);
        Auth::login($user);

        return $user;
    }

    private function findOrCreateUser(string $provider, array $providerUser): User
    {
        // same as the previous example
    }
}
```

Your software is now open for extension, but closed for modification! Congratz, you finished this principle.

If your clients are working, you won't need to touch them. And if you want to add a new provider, like Google, all you need to do is:

1. Create a `GoogleClient` class that implements `OAuthContract`;
2. Register `'google'` in the `AppServiceProvider`.

Not a single line of `AuthService`, the controller, or the routes needs to change.

> **In real life:** [Laravel Socialite](https://laravel.com/docs/socialite) already handles OAuth login using this exact idea: one driver per provider, all of them with the same interface.

---

## Navigation

[← Introduction](0-introduction.md) • [1 – Single Responsibility Principle](1-srp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md) • [5 – Dependency Inversion Principle](5-dip.md)
