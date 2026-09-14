# 4 - Interface Segregation Principle

Esse princípio é um dos mais simples de entender, porém um dos mais difíceis de ilustrar com exemplos práticos. Então, primeiro a teoria e depois o código.

> "Clients should not be forced to depend upon interfaces that they do not use."
>
> "Clientes não devem ser forçados a depender de interfaces que não usam."

O ISP trata de separar interfaces por responsabilidades específicas. Lembra do primeiro princípio, o Single Responsibility? Aqui a ideia é a mesma, só que aplicada às interfaces: é melhor ter várias interfaces pequenas do que uma interface gigante que faz de tudo.

Não entendeu? Então vamos ao exemplo. O nosso `OAuthContract` cresceu e ganhou um método para renovar o token de acesso quando ele expira:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): OAuthUser;

    public function refreshAccessToken(string $refreshToken): string;
}
```

Tá errado? Não necessariamente. Mas, quando falamos de ISP, provavelmente está. Por quê?

Vamos ao cenário:

- O nosso chat permite login com Spotify, Twitch e GitHub;
- O Spotify e a Twitch entregam tokens que expiram. Para continuar usando a API, você precisa renová-los com um **refresh token**;
- O GitHub (num OAuth App) entrega um token que não expira. Ou seja: não existe refresh token para renovar.

Agora olha o que acontece com o `GithubClient`:

```php
final class GithubClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }

    public function refreshAccessToken(string $refreshToken): string
    {
        throw new LogicException('O GitHub não usa refresh token.');
    }
}
```

A interface obrigou o `GithubClient` a implementar um método que ele não usa. Resultado: um método que só existe para lançar uma exceção. E, se alguém chamar `refreshAccessToken()` num `OAuthContract` qualquer, o código explode em produção. Lembra do LSP? Pois é, quebramos ele também.

Como vamos separar essas responsabilidades em interfaces? Se liga:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): OAuthUser;
}

interface RefreshableOAuthContract
{
    public function refreshAccessToken(string $refreshToken): string;
}
```

Separamos os métodos por responsabilidade. E como isso fica na aplicação?

```php
final class SpotifyClient implements OAuthContract, RefreshableOAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }

    public function refreshAccessToken(string $refreshToken): string
    {
        return Http::asForm()
            ->withBasicAuth(config('services.spotify.client_id'), config('services.spotify.client_secret'))
            ->post('https://accounts.spotify.com/api/token', [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
            ])
            ->json('access_token');
    }
}

final class TwitchClient implements OAuthContract, RefreshableOAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }

    public function refreshAccessToken(string $refreshToken): string
    {
        return Http::asForm()
            ->post('https://id.twitch.tv/oauth2/token', [
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
                'client_id' => config('services.twitch.client_id'),
                'client_secret' => config('services.twitch.client_secret'),
            ])
            ->json('access_token');
    }
}

final class GithubClient implements OAuthContract
{
    public function getAccessToken(string $code): string
    {
        // ...
    }

    public function getAuthenticatedUser(string $accessToken): OAuthUser
    {
        // ...
    }
}
```

Agora o tipo diz EXATAMENTE o que cada client sabe fazer. Quem precisa renovar um token pede um `RefreshableOAuthContract`, e o PHP não deixa passar um client que não sabe fazer isso:

```php
function renewAccessToken(RefreshableOAuthContract $client, string $refreshToken): string
{
    return $client->refreshAccessToken($refreshToken);
}

renewAccessToken(new TwitchClient(), $refreshToken);
// OK

renewAccessToken(new GithubClient(), $refreshToken);
// ❌ TypeError: renewAccessToken(): Argument #1 ($client) must be of type RefreshableOAuthContract, GithubClient given
```

Percebeu? Os três provedores fazem login, mas só dois sabem renovar token. E nenhum deles carrega um método que não usa.

A ideia é não escrever código desnecessário e dizer EXATAMENTE quais métodos cada classe precisa ter. Quanto mais você segregar (com bom senso), mais fácil fica entender o código e o que ele DEVE fazer.

Todos os princípios do SOLID giram em torno de responsabilidade e legibilidade, mas o ISP é o que deixa isso mais visível. Se você leu até aqui, não esquece de deixar uma estrela no repositório =)

---

## Navegação

[← Introdução](0-introducao.md) • [1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [5 – Dependency Inversion Principle](5-dip.md)
