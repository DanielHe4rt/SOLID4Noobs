# Dependency Inversion Principle

The Dependency Inversion Principle has a few characteristics that is pretty similar to **Dependency Injection**, but it's not like that at all.

All the principles listed until the moment are based in INTERFACES and SOLID itself was written thinking about developing with Interfaces, for being possible to let most part of the code generic and legible.

Now, let's understand about DIP dogmas:

- High level modules should not depend on low level modules. Both should depend on abstractions
- Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions.

This concept is pretty hard to find something to apply, because Laravel itself was structured using SOLID (at least I think). So, let's take a random example to apply it.

Let's say  that our application has two types of users: Users and Administrators. Both are authenticable and you have some message stuff between both. Now let's apply the ISP to be more clear.

```php
interface Authenticable {
    public function auth(): bool;
}

interface Messenger {
    public function prepareMessage(): string;

    public function sendMessage(): bool;
}

class User implements Authenticable, Messenger {

}

class Administrator implements Authenticable, Messenger {

}

class ChatMessage {

    public $model;

    public function __construct(Administrator $model, string $message)
    {
        $this->model = $model;
    }

    public function handle() {
        // pretends that this method handle messages stuff
    }
}
```

If we stop to think, our High Level module in this example is the ChatMessage and it DEPENDS on a low level module, that is the Administrador class. Following the 5th principle, it's already wrong because both should be depending on an abstraction.

"But how? I injected the dependency and it WORKS!!!!"

It works, but you forgot that you have to send messages to model `User` too, right? So it will be interesting to INVERT the dependency so that it can be used based on abstractions.

```php
interface Authenticable {
    public function auth(): bool;
}

interface Messenger {
    public function prepareMessage(): string;

    public function sendMessage(): bool;
}

class User implements Authenticable, Messenger {

}

class Administrator implements Authenticable, Messenger {

}

class ChatMessage {

    public $model;

    public function __construct(Authenticable $model, string $message)
    {
        $this->model = $model;
    }

    public function handle() {
        // pretends that this method handle messages stuff
    }
}
```

Our dependency is now inverted. We don't need to worry about making N classes the same process, because everything is maintained in abstractions.

And that 's it! We finished the SOLID4Noobs!

I hope you liked this article/repository and if you want to see more content relate to Laravel, consider [follow me on Twitch](https://twitch.tv/danielhe4rt) and on [Twitter](https://twitter.com/danielhe4rt)

See ya! =)
