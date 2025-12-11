# Dependency Inversion Principle

O Princípio de Inversão de Dependência tem algumas caracteristicas que poderiam assimilar à **Injeção de Dependência**, porém não é bem assim.

Todos os princípios listados até o momento são dados baseado em INTERFACES e o SOLID em si foi escrito e pensado enquanto desenvolvendo com interfaces, por ser possível deixar genérico boa parte do código e a legibilidade crescer cada vez mais.

Agora, vamos entender sobre os dogmas do DIP:

- Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações;
- Abstrações não devem depender de detalhes. Detalhes (implementações concretas) devem depender das abstrações.

Esse conceito é bem difícil de achar algo no nosso exemplo anterior para aplicar, pois o Laravel em si foi arquitetado seguindo o SOLID. Então, vamos pegar um exemplo qualquer para aplicar o que precisa.

Digamos que nossa aplicação tenha dois tipos de usuários: Usuários padrões e Administradores. Ambos são autenticáveis e você tem uma mensageria entre eles. Já vamos aplicar o ISP pra ficar bonito, né?

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

Se pararmos para pensar, o nosso módulo de alto nível aqui nesse exemplo é o ChatMessage e ele está DEPENDENDO de um módulo de baixo nível, que é a classe Administrator. Segundo o nosso quinto princípio, isso já tá erradasso pois ambos devem depender da abstração.

"Mas como assim? Injetei a dependência e ele FUNCIONA!!!! Né?"

Funciona, mas você esqueceu que você deve mandar mensagens para o usuário `User` também, né? Então seria interessante INVERTER a dependência para que possa ser usado baseado nas abstrações.

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

Nossa dependência agora está invertida. Não precisamos nos preocupar em fazer N classes para o mesmo processo, sendo que agora é tudo mantido em abstrações.

**Módulo de alto nível** (ChatMessage) depende de **abstração** (Authenticable)  
**Módulos de baixo nível** (User, Administrator) dependem de **abstração** (Authenticable)

Isso é o que significa "inversão": em vez do alto nível depender do baixo nível, ambos dependem da abstração.

E é isso! Fim do SOLID4Noobs!

Espero que você tenha curtido o conteúdo e se você quiser ver mais coisas como essa sendo aplicação em tempo real, considere se [inscrever no meu canal da twitch!](https://twitch.tv/danielhe4rt)

Até a próxima! =)

---

## Navegação

[← Introdução](0-introducao.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md)
