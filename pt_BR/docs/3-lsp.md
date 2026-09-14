# 3 - Liskov Substitution Principle

> "Se **q(x)** é uma propriedade demonstrável dos objetos **x** de tipo **T**, então **q(y)** deve ser verdadeira para os objetos **y** de tipo **S**, onde **S** é um subtipo de **T**."
>
> — Barbara Liskov e Jeannette Wing

Bom, você não precisa entender essa palhaçada aí em cima. Seria legal? Seria. Mas vamos explicar o rolê usando PHP.

Traduzindo para o dia a dia: **se o seu código funciona com um tipo, ele deve continuar funcionando com qualquer subclasse ou implementação desse tipo, sem precisar saber qual chegou.**

No princípio anterior, deixamos o código genérico com o OCP. Porém, algo muito importante ficou para trás: **o que** os métodos da interface retornam.

Beleza, primeiro uma revisão rápida sobre herança.

Quando uma classe filha estende uma classe pai, ela herda todos os métodos públicos e protegidos. E, claro, pode sobrescrever esses métodos:

```php
class Model
{
    public function save(): bool
    {
        return true;
    }
}

class User extends Model
{
    public function save(): array // ❌ Fatal error: Declaration of User::save(): array must be compatible with Model::save(): bool
    {
        return ['success' => true];
    }
}
```

Aqui o PHP nem deixa o código rodar: a classe pai promete um `bool` e a filha tenta devolver um `array`. Ao sobrescrever um método, a assinatura precisa ser **compatível** com a do pai:

- **Retorno:** igual ao do pai ou mais específico (se o pai retorna `?User`, a filha pode retornar `User`);
- **Parâmetros:** iguais aos do pai ou mais abrangentes (se o pai recebe `int`, a filha pode receber `int|string`).

Só que o PHP verifica apenas a **assinatura**. Ele não verifica o **comportamento**. E é aí que mora o Liskov.

Para entender o princípio de verdade, você precisa saber desenvolver com CONTRATOS. Mas o que ~~caralhos~~ é um contrato?

Contrato é o apelido que damos às **interfaces**: elas dizem quais métodos quem implementá-las vai precisar ter, com quais parâmetros e com qual retorno.

Agora vamos a um exemplo que quebra o Princípio de Liskov. Olha o que duas APIs retornam para o usuário autenticado:

```json
// Spotify
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
    "product": "premium",
    "type": "user",
    "uri": "spotify:user:wizzler"
}
```

```json
// Twitch
// GET https://api.twitch.tv/helix/users
{
    "data": [
        {
            "id": "141981764",
            "login": "twitchdev",
            "display_name": "TwitchDev",
            "type": "",
            "broadcaster_type": "partner",
            "description": "Supporting third-party developers building Twitch integrations from chatbots to game integrations.",
            "profile_image_url": "https://static-cdn.jtvnw.net/jtv_user_pictures/twitchdev-profile_image-300x300.png",
            "email": "not-real@email.com",
            "created_at": "2016-12-14T20:32:28Z"
        }
    ]
}
```

Repare: as duas retornam dados parecidos, mas **não no mesmo formato**. O Spotify devolve o usuário direto na raiz, enquanto a Twitch embrulha tudo dentro de `data[0]`.

Agora olha os clients implementando o `OAuthContract` do capítulo anterior:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): array;
}

final class SpotifyClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return Http::withToken($accessToken)
            ->get('https://api.spotify.com/v1/me')
            ->json();
    }
}

final class TwitchClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): array
    {
        return Http::withToken($accessToken)
            ->withHeaders(['Client-Id' => config('services.twitch.client_id')])
            ->get('https://api.twitch.tv/helix/users')
            ->json();
    }
}
```

Os dois clients cumprem a interface: recebem uma `string` e devolvem um `array`. O PHP está feliz. Agora vamos usar esse contrato numa função que cadastra o usuário no banco de dados com os campos `name` e `email`:

```php
function registerUser(OAuthContract $client, string $code): User
{
    $accessToken = $client->getAccessToken($code);
    $providerUser = $client->getAuthenticatedUser($accessToken);

    return User::query()->create([
        'name' => $providerUser['display_name'],
        'email' => $providerUser['email'],
    ]);
}

registerUser(new SpotifyClient(), $code);
// OK

registerUser(new TwitchClient(), $code);
// ❌ ErrorException: Undefined array key "display_name"
```

Com o Spotify, tudo lindo, já que `display_name` e `email` estão na raiz da resposta. Com a Twitch, os dados estão dentro de `data[0]`, e o código explode.

Então, logicamente, poderíamos tratar a Twitch dentro do `registerUser`. Certo?

```php
function registerUser(OAuthContract $client, string $code): User
{
    $accessToken = $client->getAccessToken($code);
    $providerUser = $client->getAuthenticatedUser($accessToken);

    if ($client instanceof TwitchClient) {
        $providerUser = $providerUser['data'][0];
    }

    return User::query()->create([
        'name' => $providerUser['display_name'],
        'email' => $providerUser['email'],
    ]);
}
```

Agora eu te digo: QUEBRAMOS o Princípio de Substituição de Liskov.

O `registerUser` recebe um `OAuthContract`, mas precisa saber **qual** implementação chegou para funcionar. Ou seja, o `TwitchClient` não consegue substituir o `SpotifyClient` sem alterar o código de quem usa o contrato. E cada provedor novo com um formato diferente vai trazer mais um `if`, o que também quebra o OCP.

O problema está no contrato: `array` não diz nada sobre o que tem dentro. Se você implementa um método de uma interface, o RETORNO precisa ter o mesmo formato em todos os casos, para que a classe possa ser substituída sem alterar o código.

Então, como poderíamos ter feito isso sem violar o princípio? Se liga: vamos criar um objeto que descreve exatamente o que o contrato devolve.

```php
final readonly class OAuthUser
{
    public function __construct(
        public string $id,
        public string $name,
        public string $email,
    ) {}
}

interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): OAuthUser;
}

final class SpotifyClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        $spotifyUser = Http::withToken($accessToken)
            ->get('https://api.spotify.com/v1/me')
            ->json();

        return new OAuthUser(
            id: $spotifyUser['id'],
            name: $spotifyUser['display_name'],
            email: $spotifyUser['email'],
        );
    }
}

final class TwitchClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        $twitchUser = Http::withToken($accessToken)
            ->withHeaders(['Client-Id' => config('services.twitch.client_id')])
            ->get('https://api.twitch.tv/helix/users')
            ->json('data.0');

        return new OAuthUser(
            id: $twitchUser['id'],
            name: $twitchUser['display_name'],
            email: $twitchUser['email'],
        );
    }
}
```

Agora cada client traduz a resposta da sua API para o mesmo formato, e o tipo `OAuthUser` garante isso. Não precisamos de nenhuma verificação EXTRA:

```php
function registerUser(OAuthContract $client, string $code): User
{
    $accessToken = $client->getAccessToken($code);
    $providerUser = $client->getAuthenticatedUser($accessToken);

    return User::query()->create([
        'name' => $providerUser->name,
        'email' => $providerUser->email,
    ]);
}

registerUser(new SpotifyClient(), $code);
// OK

registerUser(new TwitchClient(), $code);
// OK
```

Para fechar, as regras do Liskov para quem implementa um contrato:

- **Pré-condições não podem ser mais fortes:** a implementação não pode exigir mais do que o contrato pede. Ex.: recusar um `code` válido só porque ele não tem um prefixo específico.
- **Pós-condições não podem ser mais fracas:** a implementação não pode entregar menos do que o contrato promete. Ex.: devolver um `OAuthUser` com o e-mail vazio.
- **Exceções devem ser as esperadas:** a implementação não pode lançar exceções que quem usa o contrato não conhece.

Esse é o básico do básico sobre LSP, e o conteúdo ainda vai melhorar com o tempo e com os estudos.

---

## Navegação

[← Introdução](0-introducao.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [4 – Interface Segregation Principle](4-isp.md) • [5 – Dependency Inversion Principle](5-dip.md)
