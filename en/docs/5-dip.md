# 5 - Dependency Inversion Principle

The Dependency Inversion Principle is often confused with **Dependency Injection**, but they are not the same thing:

- **Dependency Injection (DI)** is a technique: the class receives its dependencies from the outside (through the constructor, for example) instead of creating them;
- **Dependency Inversion (DIP)** is a principle: it tells you **what type** those dependencies should be.

You can use DI and still break DIP. You'll see it in a moment.

All the principles we've seen so far revolve around INTERFACES. SOLID was designed with interfaces in mind, because they make most of the code generic and keep readability growing.

Now, let's understand the DIP dogmas:

- High-level modules should not depend on low-level modules. Both should depend on abstractions;
- Abstractions should not depend on details. Details (concrete implementations) should depend on abstractions.

In other words: a **high-level module** orchestrates a business rule (e.g. "send a chat message"). A **low-level module** takes care of the details (e.g. how each type of user receives that message).

Let's say our application has two types of users: regular users and administrators. Both can authenticate and exchange messages with each other. Let's apply ISP right away to make it cleaner:

```php
interface Authenticatable
{
    public function authenticate(): bool;
}

interface Messenger
{
    public function prepareMessage(string $message): string;

    public function sendMessage(string $message): bool;
}

final class User implements Authenticatable, Messenger
{
    // ...
}

final class Administrator implements Authenticatable, Messenger
{
    // ...
}

final readonly class SendChatMessage
{
    public function __construct(
        private Administrator $recipient,
        private string $message,
    ) {}

    public function handle(): bool
    {
        $preparedMessage = $this->recipient->prepareMessage($this->message);

        return $this->recipient->sendMessage($preparedMessage);
    }
}
```

If we stop to think, the high-level module in this example is `SendChatMessage`, and it DEPENDS on a low-level module: the `Administrator` class. According to the 5th principle, this is already wrong, because both should depend on an abstraction.

"But how? I injected the dependency and it WORKS!!!!"

It works. But you forgot that you also have to send messages to `User`, right? The way it is, you'd have to create a `SendChatMessageToUser`, and then one more class for every new type of user. So it's worth INVERTING the dependency and depending on the abstraction:

```php
final readonly class SendChatMessage
{
    public function __construct(
        private Messenger $recipient,
        private string $message,
    ) {}

    public function handle(): bool
    {
        $preparedMessage = $this->recipient->prepareMessage($this->message);

        return $this->recipient->sendMessage($preparedMessage);
    }
}
```

Notice that the dependency is now `Messenger`, not `Authenticatable`: `SendChatMessage` only needs to prepare and send messages. That's ISP helping you pick the smallest possible abstraction.

Our dependency is now inverted. We don't need to create N classes for the same process, because everything depends on abstractions:

```text
BEFORE                                  AFTER

┌─────────────────┐                     ┌─────────────────┐
│ SendChatMessage │                     │ SendChatMessage │
│  (high level)   │                     │  (high level)   │
└────────┬────────┘                     └────────┬────────┘
         │ depends on                            │ depends on
         ▼                                       ▼
┌─────────────────┐                     ┌─────────────────┐
│  Administrator  │                     │   «interface»   │
│   (low level)   │                     │    Messenger    │
└─────────────────┘                     └────────▲────────┘
                                                 │ implement
                                        ┌────────┴────────┐
                                  ┌─────┴─────┐   ┌───────┴───────┐
                                  │   User    │   │ Administrator │
                                  └───────────┘   └───────────────┘
```

In short:

- The **high-level module** (`SendChatMessage`) depends on an **abstraction** (`Messenger`);
- The **low-level modules** (`User`, `Administrator`) implement that **abstraction** (`Messenger`).

This is what "inversion" means: instead of the high level depending on the low level, both depend on the abstraction. And notice that the "before" code was already using dependency injection. Injecting isn't enough: what matters is **the type** you inject.

And that's it! We finished SOLID4Noobs!

I hope you liked this content. If you want to see more stuff like this applied live, consider [following me on Twitch](https://twitch.tv/danielhe4rt) and on [Twitter](https://twitter.com/danielhe4rt)!

See ya! =)

---

## Navigation

[← Introduction](0-introduction.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md)
