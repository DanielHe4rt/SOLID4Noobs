# 4 - Interface Segregation Principle

In my opinion, this principle is straightforward to understand and powerful in practice. First, let's see how it works in theory, and then we go to the code.

> "Clients should not be forced to depend upon interfaces that they do not use."

ISP is about splitting interfaces by specific responsibilities. Remember the first principle, Single Responsibility? It's the same idea here, but applied to interfaces: it's better to have several small interfaces than one giant interface that does everything.

Didn't get it? Let's go to the example. Our `OAuthContract` grew and got a method to renew the access token when it expires:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): OAuthUser;

    public function refreshAccessToken(string $refreshToken): string;
}
```

Is it wrong? Not necessarily. But when we're talking about ISP, it probably is. Why exactly?

Let's build a scenario:

- Our chat lets you sign in with Spotify, Twitch, and GitHub;
- Spotify and Twitch issue tokens that expire. To keep using their APIs, you have to renew them with a **refresh token**;
- GitHub (in an OAuth App) issues a token that doesn't expire. In other words: there's no refresh token to renew.

Now look at what happens to `GithubClient`:

```php
final class GithubClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }

    public function refreshAccessToken(string $refreshToken): string
    {
        throw new LogicException('GitHub does not use refresh tokens.');
    }
}
```

The interface forced `GithubClient` to implement a method it doesn't use. The result: a method that only exists to throw an exception. And if someone calls `refreshAccessToken()` on any `OAuthContract`, the code blows up in production. Remember LSP? Yep, we broke it too.

How can we split these responsibilities into interfaces? Look:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): OAuthUser;
}

interface RefreshableOAuthContract
{
    public function refreshAccessToken(string $refreshToken): string;
}
```

We split the methods by responsibility. What does our application look like after that?

```php
final class SpotifyClient implements OAuthContract, RefreshableOAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }

    public function refreshAccessToken(string $refreshToken): string
    {
        return Http::asForm()
            ->withBasicAuth(config('services.spotify.client_id'), config('services.spotify.client_secret'))
            ->post('https://accounts.spotify.com/api/token', [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
            ])
            ->json('access_token');
    }
}

final class TwitchClient implements OAuthContract, RefreshableOAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }

    public function refreshAccessToken(string $refreshToken): string
    {
        return Http::asForm()
            ->post('https://id.twitch.tv/oauth2/token', [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
                'client_id' => config('services.twitch.client_id'),
                'client_secret' => config('services.twitch.client_secret'),
            ])
            ->json('access_token');
    }
}

final class GithubClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }
}
```

Now the type tells EXACTLY what each client can do. Whoever needs to renew a token asks for a `RefreshableOAuthContract`, and PHP won't accept a client that can't do it:

```php
function renewAccessToken(RefreshableOAuthContract $client, string $refreshToken): string
{
    return $client->refreshAccessToken($refreshToken);
}

renewAccessToken(new TwitchClient(), $refreshToken);
// OK

renewAccessToken(new GithubClient(), $refreshToken);
// ❌ TypeError: renewAccessToken(): Argument #1 ($client) must be of type RefreshableOAuthContract, GithubClient given
```

See? All three providers can sign in, but only two of them can renew tokens. And none of them carries a method it doesn't use.

The idea is to not write unnecessary code and to tell EXACTLY which methods each class needs to have. The more you segregate (with common sense), the more understandable and maintainable the code will be.

All SOLID principles revolve around responsibility and readability, but ISP is the one that makes it most visible.

If you read until here, please consider leaving a star on the repository =)

---

## Navigation

[← Introduction](0-introduction.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [5 – Dependency Inversion Principle](5-dip.md)
