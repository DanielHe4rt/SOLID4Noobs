# Interface Segregation Principle

Esse princípio é um dos mais simples de entender, porém um dos mais difíceis de arrumar exemplos práticos, então primeiro vamos entender a teoria e depois vamos pro código.

ISP se dá a segregação de Interfaces responsáveis por coisas específicas. Lembra do primeiro princípio? Single Responsibility? Aqui temos o mesmo ponto, porém apenas com Interfaces.

Não entendeu? Vamos pro exemplo então:

```php
interface OAuthContract {
    public function auth(string $code): bool;

    public function getAuthenticatedUser(string $accessToken): array;

    public function findUserById(string $accessToken, $userId): array;

    public function followUser(string $accessToken, $userId): array;

    public function unfollowUser(string $accessToken, $userId): array;
}
```

Se você notar, temos duas funções que em tese deveriam estar juntas. Tá errado? Não tá. Mas quando se trata de ISP, possivelmente está errado. Mas porquê exatamente?

Se você observar BEM PRA CARALHO, existem duas coisas sendo feitas. Uma é essencial, a outra nem tanto.

Ok, vamos dar um cenário:

- Nosso software de chat tem como possibilidade logar com Spotify, Twitch e Github;
- Porém você poderá deixar mensagens para quando um usuário se cadastrar, e poderá pesquisar usuários da Twitch e Github;
- Você poderá seguir essas pessoas nas redes sociais Twitch e Github para chamar atenção delas;

Como iremos segregar essas funções em interfaces? Se liga:


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

Segregamos as funções para cada responsabilidade. E como ficaria para a nossa aplicação isso?

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

Você entendeu que para login, os três provedores estão habilitados, mas para algumas interações com as redes sociais da aplicação, apenas dois dos três provedores estão implementando?

A ideia é você não escrever código desnecessário e dizer EXATAMENTE quais são as funções necessárias dentro daquela classe. Quanto mais você segregar, mais entendível vai ficar seu código e o que ele DEVE fazer.

Os princípios dentro do SOLID entram praticamente em responsabilidade e legibilidade, porém o ISP te dá a melhor visão sobre. Se você leu até aqui, não esqueça de dar uma estrela no repositório =)

[5. Ir para 'Dependency Inversion Principle'](5-dip.md)
