# 5 - Dependency Inversion Principle

O Princípio da Inversão de Dependência costuma ser confundido com a **Injeção de Dependência**, mas não é a mesma coisa:

- **Injeção de Dependência (DI)** é uma técnica: a classe recebe as dependências de fora (pelo construtor, por exemplo) em vez de criá-las;
- **Inversão de Dependência (DIP)** é um princípio: ele diz **de que tipo** essas dependências devem ser.

Você pode usar DI e mesmo assim quebrar o DIP. Já já você vai ver.

Todos os princípios que vimos até agora giram em torno de INTERFACES. O SOLID foi pensado para desenvolver com interfaces, porque elas deixam boa parte do código genérica e a legibilidade só cresce.

Agora, vamos entender os dogmas do DIP:

- Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações;
- Abstrações não devem depender de detalhes. Detalhes (implementações concretas) devem depender de abstrações.

Traduzindo: **módulo de alto nível** é quem orquestra a regra de negócio (ex.: "enviar uma mensagem no chat"). **Módulo de baixo nível** é quem cuida dos detalhes (ex.: como cada tipo de usuário recebe essa mensagem).

Digamos que a nossa aplicação tenha dois tipos de usuário: usuários comuns e administradores. Os dois podem se autenticar e trocar mensagens entre si. Já vamos aplicar o ISP para ficar bonito, né?

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

Se pararmos para pensar, o módulo de alto nível desse exemplo é o `SendChatMessage`, e ele DEPENDE de um módulo de baixo nível: a classe `Administrator`. Pelo quinto princípio, isso já está erradasso, pois os dois deveriam depender de uma abstração.

"Mas como assim? Eu injetei a dependência e FUNCIONA!!!! Né?"

Funciona. Mas você esqueceu que também precisa mandar mensagens para o `User`, né? Do jeito que está, você teria que criar um `SendChatMessageToUser`, e depois mais uma classe para cada tipo novo de usuário. Então vale INVERTER a dependência e depender da abstração:

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

Repare que a dependência agora é `Messenger`, e não `Authenticatable`: o `SendChatMessage` só precisa preparar e enviar mensagens. É o ISP ajudando a escolher a menor abstração possível.

Nossa dependência agora está invertida. Não precisamos criar N classes para o mesmo processo, já que tudo depende de abstrações:

```text
ANTES                                   DEPOIS

┌─────────────────┐                     ┌─────────────────┐
│ SendChatMessage │                     │ SendChatMessage │
│  (alto nível)   │                     │  (alto nível)   │
└────────┬────────┘                     └────────┬────────┘
         │ depende de                            │ depende de
         ▼                                       ▼
┌─────────────────┐                     ┌─────────────────┐
│  Administrator  │                     │   «interface»   │
│  (baixo nível)  │                     │    Messenger    │
└─────────────────┘                     └────────▲────────┘
                                                 │ implementam
                                        ┌────────┴────────┐
                                  ┌─────┴─────┐   ┌───────┴───────┐
                                  │   User    │   │ Administrator │
                                  └───────────┘   └───────────────┘
```

Resumindo:

- **Módulo de alto nível** (`SendChatMessage`) depende de uma **abstração** (`Messenger`);
- **Módulos de baixo nível** (`User`, `Administrator`) implementam essa **abstração** (`Messenger`).

É isso que significa "inversão": em vez de o alto nível depender do baixo nível, os dois dependem da abstração. E repare que o código de antes já usava injeção de dependência. Injetar não basta: o que importa é **o tipo** que você injeta.

E é isso! Fim do SOLID4Noobs!

Espero que você tenha curtido o conteúdo. Se quiser ver mais coisas assim sendo aplicadas ao vivo, considere [me seguir na Twitch](https://twitch.tv/danielhe4rt)!

Até a próxima! =)

---

## Navegação

[← Introdução](0-introducao.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md)
