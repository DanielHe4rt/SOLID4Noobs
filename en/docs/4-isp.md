# Interface Segregation Principle

This principle in my opinion is the most simple to understand, but one of the most boring to implement. First, let's understand how it works in theory and then we go to code.


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

If you notice, we have two functions that in theory should be together. It's wrong? Not at all. But when is about ISP, probably it's wrong. But why exactaly?

If you look again, you'll see two thinkgs being done. One is essential, the other not so much.

Let's build a scenario for that:

- Our chatting software has a possibility to Sign In with Spotify, Twitch and Github;
- But you can leave messages for when the user register, and will should be able to search for Users from Twitch and Github;
- You'll be able to follow these people social networks such Twitch or Github.

How we will segregate those interfaces? Look:

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
We segregate the functions for each responsibility. How our application should look like after that?

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
        return [];
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return [];
    }
}

class TwitchService implements OAuthBaseContract, OAuthSocialContract {

    public function auth(string $code): array
    {
        return [];
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return [];
    }

    public function findUserById(string $accessToken, $userId): array
    {
        return [];
    }

    public function followUser(string $accessToken, $userId): array
    {
        return [];
    }

    public function unfollowUser(string $accessToken, $userId): array
    {
        return [];
    }
}

class GithubService implements OAuthBaseContract, OAuthSocialContract  {

    public function auth(string $code): array
    {
        return [];
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return [];
    }

    public function findUserById(string $accessToken, $userId): array
    {
        return [];
    }

    public function followUser(string $accessToken, $userId): array
    {
        return [];
    }

    public function unfollowUser(string $accessToken, $userId): array
    {
        return [];
    }
}
```

You understand that for Login, all providers needs to be able to ran it, but for some social network interactions, only 2/3 needs to be implemented? 

The idea is to you not write not necessarily code and tell EXACTLY which functions needs to be implement inside that class. How much more you segregate, more understandable will be your code and what is happening there.

The principles on SOLID goes practically in responsibilities and legibility, but ISP gives you the a better vision about. If you read until here, please consider leave a Star on the repository =)

[5. Go to 'Dependency Inversion Principle'](5-dip.md)
