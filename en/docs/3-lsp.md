# 3 - Liskov Substitution Principle

> "Let **q(x)** be a property provable about objects **x** of type **T**. Then **q(y)** should be true for objects **y** of type **S**, where **S** is a subtype of **T**."
>
> — Barbara Liskov and Jeannette Wing

Well, you don't have to understand this confusing formal definition. Would it be nice? Sure. But let's explain it using PHP.

In everyday terms: **if your code works with a type, it must keep working with any subclass or implementation of that type, without needing to know which one it got.**

In the previous principle, we made the code generic using OCP. But something very important was left behind: **what** the interface methods return.

Alright, first a quick review about inheritance.

When a child class extends a parent class, it inherits all the public and protected methods. And of course, it can override those methods:

```php
class Model
{
    public function save(): bool
    {
        return true;
    }
}

class User extends Model
{
    public function save(): array // ❌ Fatal error: Declaration of User::save(): array must be compatible with Model::save(): bool
    {
        return ['success' => true];
    }
}
```

Here PHP doesn't even let the code run: the parent class promises a `bool`, and the child tries to return an `array`. When you override a method, its signature must be **compatible** with the parent's:

- **Return type:** the same as the parent's or more specific (if the parent returns `?User`, the child can return `User`);
- **Parameters:** the same as the parent's or broader (if the parent accepts `int`, the child can accept `int|string`).

But PHP only checks the **signature**. It doesn't check the **behavior**. And that's where Liskov lives.

To really understand this principle, you need to know how to develop with CONTRACTS. But what the ~~hell~~ is a contract?

Contract is the nickname we give to **interfaces**: they say which methods an implementation must have, with which parameters, and with which return type.

Now let's look at an example that breaks the Liskov Substitution Principle. Here is what two APIs return for the authenticated user:

```json
// Spotify
// GET https://api.spotify.com/v1/me
{
    "country": "SE",
    "display_name": "JM Wizzler",
    "email": "email@example.com",
    "external_urls": {
        "spotify": "https://open.spotify.com/user/wizzler"
    },
    "followers": {
        "href": null,
        "total": 3829
    },
    "href": "https://api.spotify.com/v1/users/wizzler",
    "id": "wizzler",
    "product": "premium",
    "type": "user",
    "uri": "spotify:user:wizzler"
}
```

```json
// Twitch
// GET https://api.twitch.tv/helix/users
{
    "data": [
        {
            "id": "141981764",
            "login": "twitchdev",
            "display_name": "TwitchDev",
            "type": "",
            "broadcaster_type": "partner",
            "description": "Supporting third-party developers building Twitch integrations from chatbots to game integrations.",
            "profile_image_url": "https://static-cdn.jtvnw.net/jtv_user_pictures/twitchdev-profile_image-300x300.png",
            "email": "not-real@email.com",
            "created_at": "2016-12-14T20:32:28Z"
        }
    ]
}
```

Notice that both return similar data, but **not in the same shape**. Spotify returns the user at the root, while Twitch wraps everything inside `data[0]`.

Now look at the clients implementing the `OAuthContract` from the previous chapter:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): array;
}

final class SpotifyClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return Http::withToken($accessToken)
            ->get('https://api.spotify.com/v1/me')
            ->json();
    }
}

final class TwitchClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return Http::withToken($accessToken)
            ->withHeaders(['Client-Id' => config('services.twitch.client_id')])
            ->get('https://api.twitch.tv/helix/users')
            ->json();
    }
}
```

Both clients honor the interface: they receive a `string` and return an `array`. PHP is happy. Now let's use this contract in a function that registers the user in the database with the `name` and `email` fields:

```php
function registerUser(OAuthContract $client, string $code): User
{
    $accessToken = $client->getAccessToken($code);
    $providerUser = $client->getAuthenticatedUser($accessToken);

    return User::query()->create([
        'name' => $providerUser['display_name'],
        'email' => $providerUser['email'],
    ]);
}

registerUser(new SpotifyClient(), $code);
// OK

registerUser(new TwitchClient(), $code);
// ❌ ErrorException: Undefined array key "display_name"
```

With Spotify, everything works, since `display_name` and `email` are at the root of the response. With Twitch, the data is inside `data[0]`, and the code blows up.

So, logically, we could handle Twitch inside `registerUser`. Right?

```php
function registerUser(OAuthContract $client, string $code): User
{
    $accessToken = $client->getAccessToken($code);
    $providerUser = $client->getAuthenticatedUser($accessToken);

    if ($client instanceof TwitchClient) {
        $providerUser = $providerUser['data'][0];
    }

    return User::query()->create([
        'name' => $providerUser['display_name'],
        'email' => $providerUser['email'],
    ]);
}
```

Now, what if I told you that we BROKE the Liskov Substitution Principle?

`registerUser` receives an `OAuthContract`, but it needs to know **which** implementation it got in order to work. In other words, `TwitchClient` can't replace `SpotifyClient` without changing the code that uses the contract. And every new provider with a different shape will bring another `if`, which also breaks OCP.

The problem is the contract: `array` says nothing about what's inside. If you implement an interface method, the RETURN must have the same shape in every case, so the class can be replaced without changing the code (it's LITERALLY a CONTRACT).

So, how could we have done this without violating the principle? Look: let's create an object that describes exactly what the contract returns.

```php
final readonly class OAuthUser
{
    public function __construct(
        public string $id,
        public string $name,
        public string $email,
    ) {}
}

interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): OAuthUser;
}

final class SpotifyClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        $spotifyUser = Http::withToken($accessToken)
            ->get('https://api.spotify.com/v1/me')
            ->json();

        return new OAuthUser(
            id: $spotifyUser['id'],
            name: $spotifyUser['display_name'],
            email: $spotifyUser['email'],
        );
    }
}

final class TwitchClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        $twitchUser = Http::withToken($accessToken)
            ->withHeaders(['Client-Id' => config('services.twitch.client_id')])
            ->get('https://api.twitch.tv/helix/users')
            ->json('data.0');

        return new OAuthUser(
            id: $twitchUser['id'],
            name: $twitchUser['display_name'],
            email: $twitchUser['email'],
        );
    }
}
```

Now each client translates its API response into the same shape, and the `OAuthUser` type guarantees it. We don't need any EXTRA checks:

```php
function registerUser(OAuthContract $client, string $code): User
{
    $accessToken = $client->getAccessToken($code);
    $providerUser = $client->getAuthenticatedUser($accessToken);

    return User::query()->create([
        'name' => $providerUser->name,
        'email' => $providerUser->email,
    ]);
}

registerUser(new SpotifyClient(), $code);
// OK

registerUser(new TwitchClient(), $code);
// OK
```

To wrap it up, here are the Liskov rules for anyone implementing a contract:

- **Preconditions can't be stronger:** the implementation can't require more than the contract asks for. E.g. rejecting a valid `code` just because it doesn't have a specific prefix.
- **Postconditions can't be weaker:** the implementation can't deliver less than the contract promises. E.g. returning an `OAuthUser` with an empty email.
- **Exceptions must be the expected ones:** the implementation can't throw exceptions that the contract's users don't know about.

This is the very basics of LSP. Hope you enjoyed it!

---

## Navigation

[← Introduction](0-introduction.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [4 – Interface Segregation Principle](4-isp.md) • [5 – Dependency Inversion Principle](5-dip.md)
