# Dependency Inversion Principle

The Dependency Inversion Principle is often confused with **Dependency Injection**, but they are related yet distinct concepts. DIP is a design principle, while DI is an implementation technique.

All the principles listed until now are based on INTERFACES and SOLID itself was written thinking about developing with Interfaces, making it possible to let most parts of the code be generic and legible.

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
    public function prepareMessage(string $message): string;

    public function sendMessage(string $message): bool;
}

class User implements Authenticable, Messenger {

}

class Administrator implements Authenticable, Messenger {

}

class ChatMessage {

    public $model;

    public function __construct(
        private readonly Administrator $model,
        private readonly string $message
    ) {}

    public function handle(): bool
    {
        $prepared = $this->model->prepareMessage($this->message);
        return $this->model->sendMessage($prepared);
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

    public function __construct(
        private readonly Authenticable $model,
        private readonly string $message
    ) {}

    public function handle(): bool
    {
        $prepared = $this->model->prepareMessage($this->message);
        return $this->model->sendMessage($prepared);
    }
}
```

Our dependency is now inverted. We don't need to worry about making N classes with the same process, because everything is maintained through abstractions.

**High-level module** (ChatMessage) depends on **abstraction** (Authenticable)  
**Low-level modules** (User, Administrator) depend on **abstraction** (Authenticable)

This is what "inversion" means: instead of high-level depending on low-level, both depend on abstraction.

And that's it! We finished the SOLID4Noobs!

I hope you liked this article/repository and if you want to see more content related to Laravel, consider [following me on Twitch](https://twitch.tv/danielhe4rt) and on [Twitter](https://twitter.com/danielhe4rt)

See ya! =)

---

## Navigation

[← Introduction](0-introduction.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md)
