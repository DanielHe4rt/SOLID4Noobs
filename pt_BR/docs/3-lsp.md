# 3. Liskov Substitution Principle


"Se **q(x)** é uma propriedade demonstrável dos objetos **x** de tipo **T**. Então **q(y)** deve ser verdadeiro para objetos **y** de tipo **S** onde **S** é subtipo de **T**."

Bom, você não precisa entender essa palhaçada ai em cima. Seria legal? Seria. Mas vamos explicar o rolê usando PHP.

Vimos muito código no princípio anterior, onde nós deixamos o código genérico usando OCP. Porém, ficou algo muito importante pra trás. O retorno dos métodos onde implementamos a interface.

Beleza, vamos entender um pouco sobre herança:

Quando você estende uma classe pai para uma classe filho, você herda todos os seus métodos públicos/protegidos. E claro, você pode sobrescrever esses métodos sem maiores problemas.

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

Se você quer entender o Princípio de Liskov, a primeira coisa que vai precisar é entender como desenvolver com CONTRATOS. Isso já vimos no OCP, porém vocês viram no exemplo acima que na classe pai, o retorno é um **boolean** e na classe filho o retorno é um **array** e isso quebra completamente o princípio trabalhado.

Em tese, se você for sobrescrever algo, você **DEVE** alinhar os tipos de retorno. Se a função pai é do tipo X **fn parent(): x**, a função sobrescrita também deve retornar X **fn children(): x**.

Mas até aqui não vimos nenhum contrato sendo implementado. Mas, o que ~~caralhos~~ é um contrato?

Contrato é um nome dado a **Interfaces**, onde você consegue dizer quais funções serão necessárias para quem implementá-las.

Agora vamos pra outro exemplo que quebra o princípio de Liskov.

Vamos olhar duas API's e seus retornos:

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

Se analisarmos bem, os dois objetos retornam dados similares e dada a nossa interface faz sentido no contexto da API.

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

Nosso código tá projetado para trazer um array com os objetos listados acima do snippet. Vamos implementar uma função com essa interface.

Precisamos inserir um novo usuário no nosso banco de dados, que tem como campos principais: id e email.

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

Se rodarmos o código com o provider do Spotify, vai ser tudo uma maravilha até porquê ele retorna os dois campos para o usuário. Mas, no provedor da Twitch, ele retorna o campo de id como "_id".

Então, lógicamente nós poderíamos criar uma validação pra entender se esses campos colidem dentro do nosso registerUser. Certo?


```php
function registerUser(OAuthContract $service, $token) {

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
// INVALIDO
```

Agora eu te digo que QUEBRAMOS o princípio de substituição de Liskov. Nós não deveríamos fazer validações dentro de interfaces pois ela espera sempre os MESMOS dados. Se você implementa um método em uma interface, é necessário que o RETORNO seja igual para todos os casos, de um jeito onde a classe possa ser substituída sem alteração do código.

Então, como poderíamos ter feito isso para não violar o princípio? Se liga:

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

Agora está padronizado e nós não precisamos fazer nenhuma validação EXTRA.

```php
function registerUser(OAuthContract $service, $token) {

    $token = $service->auth($token);
    $user = $service->getAuthenticatedUser($token['access_token']);

    return User::create($user)
}


registerUser(new SpotifyService, '123');
// OK

registerUser(new TwitchService, '123');
// OK
```


Entendemos que as pré-condições não devem ser maiores e as pós condições devem ser iguais. Caso você tenha exceptions dentro do seu código, elas devem ser iguais também.

Esse é um básico bem básico sobre LSP e ainda vai ser melhorado com o tempo e com os estudos.


[4. Ir para 'Interface Segregation'](4-isp.md)
