# 2 - Open-Closed Principle

O Princípio Aberto-Fechado (nome tosco, eu sei) diz que você deve conseguir adicionar comportamento novo ao código sem alterar o código que já está escrito.

A ideia é deixar o código genérico usando interfaces. Cada interface define um conjunto de métodos, e o polimorfismo vira a estrela do show.

> "Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification."
>
> "Entidades de software (classes, módulos, funções etc.) devem estar abertas para extensão, mas fechadas para modificação."

Digamos que a sua plataforma tenha login via OAuth (plataformas de terceiros) com Discord, Twitch e GitHub.

Para isso, você cria três rotas diferentes, uma para cada provedor (já que cada um fala com um serviço diferente):

```php
// routes/web.php
Route::get('auth/oauth/discord', [AuthController::class, 'discord']);
Route::get('auth/oauth/twitch', [AuthController::class, 'twitch']);
Route::get('auth/oauth/github', [AuthController::class, 'github']);
```

```php
// app/Http/Controllers/AuthController.php
final class AuthController extends Controller
{
    public function __construct(
        private readonly AuthService $authService,
    ) {}

    public function discord(Request $request): JsonResponse
    {
        $user = $this->authService->loginWithDiscord($request->query('code'));

        return response()->json($user);
    }

    public function twitch(Request $request): JsonResponse
    {
        $user = $this->authService->loginWithTwitch($request->query('code'));

        return response()->json($user);
    }

    public function github(Request $request): JsonResponse
    {
        $user = $this->authService->loginWithGithub($request->query('code'));

        return response()->json($user);
    }
}
```

```php
// app/Services/AuthService.php
final class AuthService
{
    public function loginWithDiscord(string $code): User
    {
        $client = new DiscordClient();
        $accessToken = $client->authWithDiscord($code);
        $discordUser = $client->getDiscordUser($accessToken);

        $user = $this->findOrCreateUser('discord', $discordUser);
        Auth::login($user);

        return $user;
    }

    public function loginWithTwitch(string $code): User
    {
        $client = new TwitchClient();
        $accessToken = $client->authWithTwitch($code);
        $twitchUser = $client->getTwitchUser($accessToken);

        $user = $this->findOrCreateUser('twitch', $twitchUser);
        Auth::login($user);

        return $user;
    }

    public function loginWithGithub(string $code): User
    {
        $client = new GithubClient();
        $accessToken = $client->authWithGithub($code);
        $githubUser = $client->getGithubUser($accessToken);

        $user = $this->findOrCreateUser('github', $githubUser);
        Auth::login($user);

        return $user;
    }

    private function findOrCreateUser(string $provider, array $providerUser): User
    {
        $providerColumn = "{$provider}_id";
        $user = User::query()->firstWhere('email', $providerUser['email']);

        if ($user === null) {
            return User::query()->create([
                'name' => $providerUser['name'],
                'email' => $providerUser['email'],
                $providerColumn => $providerUser['id'],
            ]);
        }

        if ($user->{$providerColumn} === null) {
            $user->update([$providerColumn => $providerUser['id']]);

            return $user;
        }

        $isSameAccount = (string) $user->{$providerColumn} === (string) $providerUser['id'];

        if (! $isSameAccount) {
            throw new AuthenticationException('Este e-mail já está vinculado a outra conta.');
        }

        return $user;
    }
}
```

Olhando os snippets acima, dá pra ver um padrão que podemos usar para melhorar o código. O fluxo do OAuth é **genérico**: todo provedor troca um `code` por um token de acesso e depois devolve os dados do usuário. Mas o nosso código ainda não entende isso.

Do jeito que está, FUNCIONA. Só que, para cada provedor novo, você precisa mexer na rota, no controller e no service. Dá pra fazer funcionar com uma lógica bem mais bonita.

Vamos juntar as três rotas numa só:

```php
// routes/web.php
Route::get('auth/oauth/{provider}', [AuthController::class, 'login']);
```

Só com essa troca, já dá pra perceber que vamos deixar as coisas mais genéricas, já que existe um padrão. Agora vamos ajustar o controller:

```php
// app/Http/Controllers/AuthController.php
final class AuthController extends Controller
{
    public function __construct(
        private readonly AuthService $authService,
    ) {}

    public function login(Request $request, string $provider): JsonResponse
    {
        $user = $this->authService->login($provider, $request->query('code'));

        return response()->json($user);
    }
}
```

O controller passa o provedor para o service, e o service que lute para descobrir qual dos 3 (ou N) clients chamar.

Agora vamos analisar os métodos dos clients que o service chama:

```php
// DiscordClient
$accessToken = $client->authWithDiscord($code);
$providerUser = $client->getDiscordUser($accessToken);

// TwitchClient
$accessToken = $client->authWithTwitch($code);
$providerUser = $client->getTwitchUser($accessToken);

// GithubClient
$accessToken = $client->authWithGithub($code);
$providerUser = $client->getGithubUser($accessToken);
```

Existe um padrão, mas os nomes dos métodos, apesar de intuitivos, não são genéricos. Agora, se pararmos e criarmos uma **INTERFACE**, isso muda completamente.

Vamos chamar a nossa interface de `OAuthContract`, com os seguintes métodos:

```php
interface OAuthContract
{
    public function getAccessToken(string $code): string;

    public function getAuthenticatedUser(string $accessToken): array;
}
```

Se todos os clients implementarem essa interface, o PHP vai GARANTIR que os métodos existem. E aí tanto faz qual client chegou. Se liga:

```php
final class DiscordClient implements OAuthContract { /* ... */ }
final class TwitchClient implements OAuthContract { /* ... */ }
final class GithubClient implements OAuthContract { /* ... */ }

$accessToken = $client->getAccessToken($code);
$providerUser = $client->getAuthenticatedUser($accessToken);
```

Agora precisamos dizer ao service qual client usar. O jeito certo é tipar o retorno do método com a **INTERFACE**:

```php
private function resolveClient(string $provider): OAuthContract
{
    return match ($provider) {
        'discord' => new DiscordClient(),
        'twitch' => new TwitchClient(),
        'github' => new GithubClient(),
        default => throw new InvalidArgumentException("Provedor OAuth não suportado: {$provider}"),
    };
}
```

Como o retorno é do tipo `OAuthContract`, o método só pode devolver uma classe que implemente essa interface, e é isso que obriga aqueles métodos genéricos a existirem. Se você tentar devolver uma classe sem a interface, vai dar merda (um belo `TypeError`).

Agora vamos refatorar o service para receber essa mudança:

```php
// app/Services/AuthService.php
final class AuthService
{
    public function login(string $provider, string $code): User
    {
        $client = $this->resolveClient($provider);

        $accessToken = $client->getAccessToken($code);
        $providerUser = $client->getAuthenticatedUser($accessToken);

        $user = $this->findOrCreateUser($provider, $providerUser);
        Auth::login($user);

        return $user;
    }

    private function resolveClient(string $provider): OAuthContract
    {
        return match ($provider) {
            'discord' => new DiscordClient(),
            'twitch' => new TwitchClient(),
            'github' => new GithubClient(),
            default => throw new InvalidArgumentException("Provedor OAuth não suportado: {$provider}"),
        };
    }

    private function findOrCreateUser(string $provider, array $providerUser): User
    {
        // igual ao exemplo anterior
    }
}
```

## Fechando de vez para modificação

Repare num detalhe: para adicionar o Google, você ainda precisaria abrir o `AuthService` e mexer no `match`. Ou seja, ele ainda não está 100% fechado para modificação.

Para resolver, tiramos a lista de clients de dentro do service e deixamos o container do Laravel entregar tudo pronto:

```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Services\AuthService;
use App\Services\OAuth\DiscordClient;
use App\Services\OAuth\GithubClient;
use App\Services\OAuth\TwitchClient;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;

final class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->when(AuthService::class)
            ->needs('$clients')
            ->give(fn (Application $app): array => [
                'discord' => $app->make(DiscordClient::class),
                'twitch' => $app->make(TwitchClient::class),
                'github' => $app->make(GithubClient::class),
            ]);
    }
}
```

```php
// app/Services/AuthService.php
namespace App\Services;

use App\Models\User;
use App\Services\OAuth\OAuthContract;
use Illuminate\Support\Facades\Auth;
use InvalidArgumentException;

final class AuthService
{
    /**
     * @param array<string, OAuthContract> $clients
     */
    public function __construct(
        private readonly array $clients,
    ) {}

    public function login(string $provider, string $code): User
    {
        $client = $this->clients[$provider]
            ?? throw new InvalidArgumentException("Provedor OAuth não suportado: {$provider}");

        $accessToken = $client->getAccessToken($code);
        $providerUser = $client->getAuthenticatedUser($accessToken);

        $user = $this->findOrCreateUser($provider, $providerUser);
        Auth::login($user);

        return $user;
    }

    private function findOrCreateUser(string $provider, array $providerUser): User
    {
        // igual ao exemplo anterior
    }
}
```

Seu software agora está aberto para extensão, mas fechado para modificação! Parabéns, você chegou ao fim dessa palhaçada.

Se os seus clients estiverem lindos, maravilhosos e funcionando, você não vai precisar mexer neles. E, se quiser adicionar um provedor novo, como o Google, basta:

1. Criar a classe `GoogleClient` implementando `OAuthContract`;
2. Registrar `'google'` no `AppServiceProvider`.

Nenhuma linha do `AuthService`, do controller ou das rotas precisa mudar.

> **Na vida real:** o [Laravel Socialite](https://laravel.com/docs/socialite) já resolve login OAuth aplicando exatamente essa ideia: um driver por provedor, todos com a mesma interface.

---

## Navegação

[← Introdução](0-introducao.md) • [1 – Single Responsibility Principle](1-srp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md) • [5 – Dependency Inversion Principle](5-dip.md)
