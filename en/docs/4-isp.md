# Interface Segregation Principle

This principle in my opinion is straightforward to understand and powerful in practice. First, let's understand how it works in theory and then we go to code.


ISP takes place as Interface Segregation of specific things. Remember the first principle? Single Responsibility? Now we have the same point, but with Interfaces.

Didn't get it? Let's go to the example:

```php
interface OAuthContract {
    public function auth(string $code): bool;

    public function getAuthenticatedUser(string $accessToken): array;

    public function findUserById(string $accessToken, $userId): array;

    public function followUser(string $accessToken, $userId): array;

    public function unfollowUser(string $accessToken, $userId): array;
}
```

If you notice, we have two groups of functions that in theory should be together. Is it wrong? Not at all. But when we're talking about ISP, it probably is. But why exactly?

If you look again, you'll see two things being done. One is essential, the other not so much.

Let's build a scenario for that:

- Our chatting software has a possibility to Sign In with Spotify, Twitch and Github;
- But you can leave messages for when the user gets registered, and you should be able to search for Users from Twitch and Github;
- You'll be able to follow these people on social networks such as Twitch or Github.

How can we segregate those interfaces? Look:

```php
interface OAuthBaseContract {
    public function auth(string $code): bool;

    public function getAuthenticatedUser(string $accessToken): array;
}

interface OAuthSocialContract {
    public function findUserById(string $accessToken, $userId): array;

    public function followUser(string $accessToken, $userId): array;

    public function unfollowUser(string $accessToken, $userId): array;
}
```
We segregate the functions for each responsibility. What should our application look like after that?

```php
interface OAuthBaseContract {
    public function auth(string $code): bool;

    public function getAuthenticatedUser(string $accessToken): array;
}

interface OAuthSocialContract {
    public function findUserById(string $accessToken, $userId): array;

    public function followUser(string $accessToken, $userId): array;

    public function unfollowUser(string $accessToken, $userId): array;
}

class SpotifyService implements OAuthBaseContract {

    public function auth(string $code): bool
    {
        // Authenticate with Spotify API
        return true;
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        // Return user data from Spotify
        return ['id' => '123', 'email' => 'user@example.com'];
    }
}

class TwitchService implements OAuthBaseContract, OAuthSocialContract {

    public function auth(string $code): bool
    {
        // Authenticate with Twitch API
        return true;
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        // Return user data from Twitch
        return ['id' => '456', 'email' => 'user@twitch.tv'];
    }
    }

    public function findUserById(string $accessToken, string $userId): array
    {
        // Find Twitch user by ID
        return ['id' => $userId, 'username' => 'twitchuser'];
    }

    public function followUser(string $accessToken, string $userId): array
    {
        // Follow user on Twitch
        return ['success' => true];
    }

    public function unfollowUser(string $accessToken, string $userId): array
    {
        // Unfollow user on Twitch
        return ['success' => true];
    }
}

class GithubService implements OAuthBaseContract, OAuthSocialContract  {

    public function auth(string $code): bool
    {
        // Authenticate with Github API
        return true;
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        // Return user data from Github
        return ['id' => '789', 'email' => 'user@github.com'];
    }

    public function findUserById(string $accessToken, string $userId): array
    {
        // Find Github user by ID
        return ['id' => $userId, 'login' => 'githubuser'];
    }

    public function followUser(string $accessToken, string $userId): array
    {
        // Follow user on Github
        return ['success' => true];
    }

    public function unfollowUser(string $accessToken, string $userId): array
    {
        // Unfollow user on Github
        return ['success' => true];
    }
}
```

You understand that for Login, all providers need to be able to run it, but for some social network interactions, only 2/3 needs to be implemented? 

The idea is to not write unnecessary code and tell EXACTLY which functions need to be implemented inside that class. The more you segregate, the more understandable and maintainable the code will be.

The principles of SOLID are practical in terms of responsibilities and legibility, but ISP provides a better understanding.

If you read until here, please consider leave a Star on the repository =)

---

## Navigation

[← Introduction](0-introduction.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [5 – Dependency Inversion Principle](5-dip.md)
