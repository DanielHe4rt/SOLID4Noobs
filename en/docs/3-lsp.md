# 3. Liskov Substitution Principle


"Let **q(x)** be a property provable about object **x** of type **T**. Then **q(y)** should be true for objects **y** of type **S** where **S** is a subtype of **T**."

Well, you don't have to understand this bullshit above. It's will be nice to understand? Probably. But let's explain it using PHP.

We saw a lot of code on the previous principle, where we let the code more generic using OCP. But, something very important was missing. The return of the implement methods where we implemented the interfaces.


Alright, a quick review about inheritance:

When you extends a parent class to a child, you inherit all the public/protected methods. And of course, you can override those methods.

```php
class Model {
    public function store() {
        return true;
    }
}

class User extends Model {
    public function store() {
        return ['success'];
    }
}
```

If you want to undestand the Liskov's Principle, the first thing you need to know is how to develop with **Contracts/Interfaces**. We saw something about on OCP, but you saw the example above, on the parent class the return of method store is **boolean** and on the child class the return is an **array** and it breaks completely the worked principle.

In theory, if you override something, you **HAVE** to keep the return type from the parent. If, the parent function returns X **fn parent(): x**, the override function should return X **fn children(): x**.

Until now we don't see any contract being implemented. But, what in the name of ~~fuck~~ is a contract?

Contract is a given name to **Interfaces**, where you can say which functions will be necessary to implement.

Now let's see other example that breaks the Liskov's Principle.

Here we have some responses from the OAuth Api's:

```json
// Spotify User Authenticated API
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
    "images": [
        {
            "height": null,
            "url": "https://fbcdn-profile-a.akamaihd.net/hprofile-ak-frc3/t1.0-1/1970403_10152215092574354_1798272330_n.jpg",
            "width": null
        }
    ],
    "product": "premium",
    "type": "user",
    "uri": "spotify:user:wizzler"
}
```

```json
// Twitch User Authenticated API
// GET https://api.twitch.tv/kraken/user
{
    "_id": 44322889,
    "bio": "Just a gamer playing games and chatting. :)",
    "created_at": "2013-06-03T19:12:02Z",
    "display_name": "dallas",
    "email": "email-address@provider.com",
    "email_verified": true,
    "logo": "https://static-cdn.jtvnw.net/jtv_user_pictures/dallas-profile_image-1a2c906ee2c35f12-300x300.png",
    "name": "dallas",
    "notifications": {
        "email": false,
        "push": true
    },
    "partnered": false,
    "twitter_connected": false,
    "type": "staff",
    "updated_at": "2016-12-14T01:01:44Z"
}
```

If we analyze it, both objects returns similar data and the given Interface makes sense on the API context.

```php
interface OAuthContract {
    public function auth(string $code): bool;

    public function getAuthenticatedUser(string $accessToken): array;
}

class SpotifyService implements OAuthContract {

    public function auth(string $code): bool
    {
        // do stuff
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return HTTP::get('https://api.spotify.com/v1/me');
    }
}

class TwitchService implements OAuthContract {

    public function auth(string $code): bool
    {
        // do stuff
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return HTTP::get('https://api.twitch.tv/kraken/user');
    }
}

```

Our code is projected to return an array with the listed objects of the snippet. Now let's implement a function with that interface.

We have to insert a new user into the database, the principal fields are: id and email.

```php
interface OAuthContract {
    public function auth(string $code): bool;

    public function getAuthenticatedUser(string $accessToken): array;
}

function registerUser(OAuthContract $service, $token) {

    $token = $service->auth($token);
    $user = $service->getAuthenticatedUser($token['access_token']);

    return User::create($user)
}
```

If we ran the code with the Spotify provider, it going to works magically even because it returns both fields to user. But on the Twitch provider, it returns a field id as "_id".

So, logically we could create a a validation to understand if those fields collides inside our registerUser method. Right?

```php
function registerUser(OAuthContract $service, string $token) {

    $token = $service->auth($token);
    $user = $service->getAuthenticatedUser($token['access_token']);

    if (isset($user['_id'])) {
        $user['id'] = $user['_id'];
    }

    return User::create($user)
}


registerUser(new SpotifyService, '123');
// OK

registerUser(new TwitchService, '123');
// INVALID
```

Now, and if I say that we BROKE the Liskov Principle? We shouldn't make validations inside the interfaces about the return of the methods because the structure expect should be always the same (is LITERALY a CONTRACT). If you implement a method in a interface, it's necessary that the return should be equal to all cases, in a way that the class can be replaced without change the code.

So, how we do not violate the principle? Look:

```php
interface OAuthContract {
    public function auth(string $code): bool;

    public function getAuthenticatedUser(string $accessToken): array;
}

class SpotifyService implements OAuthContract {

    public function auth(string $code): bool
    {
        // do stuff
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        $result =  HTTP::get('https://api.spotify.com/v1/me');

        return [
            'id' => $result['id'],
            'email' => $result['email']
        ];
    }
}

class TwitchService implements OAuthContract {

    public function auth(string $code): bool
    {
        // do stuff
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        $result = HTTP::get('https://api.twitch.tv/kraken/user');

        return [
            'id' => $result['_id'],
            'email' => $result['email']
        ];
    }
}
```
Now that everything is standardized we don't need to make any extra validation.

```php
function registerUser(OAuthContract $service, string $token) {

    $token = $service->auth($token);
    $user = $service->getAuthenticatedUser($token['access_token']);

    return User::create($user)
}


registerUser(new SpotifyService, '123');
// OK

registerUser(new TwitchService, '123');
// OK
```

We understood that pre-conditions not should be greather and post-conditions should be equals. In the case you have exceptions inside the code it needs to be equals too.

This a very very basic content about LSP, hope you enjoyed.


[4. Go to 'Interface Segregation'](4-isp.md)
