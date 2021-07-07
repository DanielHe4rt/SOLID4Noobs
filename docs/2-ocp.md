# 2 - Open-closed Principle

O principio aberto-fechado (nome tosco) reflete uma parte do código que precisa ser implementada, porém sem alterar o código já escrito. 

A ideia é deixar o código genérico, com a aplicação de interfaces, com funções pré-definidas onde o polimosfismo vira o principal atrativo pro desenvolvimento usando esse principio.


<center>
    "Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification."
</center>

Vamos dizer que você tenha três rotas para autenticação OAuth (plataforma de terceiros), onde você quer que sua plataforma tenha login pela Twitch, Github e Spotify.

Nela você cria três rotas diferentes, uma pra cada tipo de autenticação (já que vão para serviços diferentes).


```php
// routes/web.php
Route::get('auth/oauth/discord', [AuthController:: class,'getDiscordAuth']);
Route::get('auth/oauth/twitch', [AuthController:: class,'getTwitchAuth']);
Route::get('auth/oauth/github', [AuthController:: class,'getGithubAuth']);
```



```php
// app/Http/Controllers/AuthController.php
class AuthController {
    
    private $repository;

    public function __construct(AuthRepository $repository) 
    {
        $this->repository = $repository;
    }

    public function getDiscordAuth(Request $request) 
    {
        try {
            $result = $this->repository->discordAuth($request->input('code'));return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }

    public function getTwitchAuth(Request $request) 
    {
        try {
            $result = $this->repository->twitchAuth($request->input('code'));return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }

    public function getGithubAuth(Request $request) 
    {
        try {
            $result = $this->repository->githubAuth($request->input('code'));return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }
}
```

```php

class AuthRepository {
    
    public function discordAuth(string $code) 
    {
        $service = new DiscordService();
        $authData = $service->authWithDiscord($code);

        $response = $authService->getDiscordUser($authData['access_token']);
        $authUser = $this->findOrCreate('discord',$response);

        Auth::user($authUser);
        return true;
    }

    public function twitchAuth(string $code) 
    {
        $service = new TwitchService();
        $authData = $service->authWithTwitch($code);

        $response = $authService->getTwitchUser($authData['access_token']);
        $authUser = $this->findOrCreate('twitch',$response);

        Auth::user($authUser);
        return true;
    }

    public function githubAuth(string $code) 
    {
        $service = new GithubService();
        $authData = $service->authWithGithub($code);

        $response = $authService->getGithubUser($authData['access_token']);
        $authUser = $this->findOrCreate('github',$response);

        Auth::user($authUser);
        return true;
    }

    public function findOrCreate(string $provider, $providerData): User
    {
        $auth = User::where('email', $providerData['email'])->first();
        if (!$auth) {
            return User::create([
                'name' => $providerData['name'],
                'email' => $providerData['email'],
                $provider . "_id" => $providerData['id'],
            ]);
        }

        if (empty($auth->{$provider . "_id"})) {
            $auth->update([
                $provider . "_id" => $providerData['id']
            ]);
            return $auth;
        }

        if ($auth->{$provider . "_id"} == $providerData['id']) {
            return $auth;
        }

        throw new \Exception('deu ruim');
    }

}
```

Se você ver os snippets acima, dá pra ver que tem um padrão que podemos seguir pra melhorar o código. Se começarmos a perceber, o OAuth em si é **genérico** então as requisições trazem os mesmos dados enquanto logando, mas o nosso código não entende isso.

Do jeito feito, ele FUNCIONA, porém dá pra fazer funcionar com uma lógica mais bonita ainda.

Vamos resumir essas três rotas em uma só desse jeito:

```php
Route::get('auth/oauth/{provider}', [AuthController:: class, 'getOAuth']);
```

Apenas trocando essa rota, já deu pra entender que vamos deixar as coisas mais genéricas tendo em vista que há um padrão. Agora vamos alterar o nosso controller para comportar essas mudanças:


```php
// app/Http/Controllers/AuthController.php
class AuthController {
    
    private $repository;

    public function __construct(AuthRepository $repository) 
    {
        $this->repository = $repository;
    }

    public function getOAuth(Request $request, string $provider) 
    {
        try {
            $result = $this->repository->authenticateOAuth($provider,$request->input('code'));
            return response()->json($result);
        } catch(UnauthorizedException $e) {
            return response()->json($e->getMessage(), 401);
        }
    }
}
```

Passsamos o provedor de dados que queremos consumir pra dentro do nosso repositório e ele que lute pra saber qual dos 3/N chamar.

Agora vamos analisar as funções do serviço que estão sendo chamadas no nosso repositório.

```php
// GithubService
$service = new GithubService();
$authData = $service->authWithGithub($code);
$response = $authService->getGithubUser($authData['access_token']);

// DiscordService
$service = new DiscordService();
$authData = $service->authWithDiscord($code);
$response = $authService->getDiscordUser($authData['access_token']);

// TwitchService
$service = new TwitchService();
$authData = $service->authWithTwitch($code);
$response = $authService->getTwitchUser($authData['access_token']);
```

Podemos ver que existe um padrão nisso, porém os nomes das funções são intuitivos mas não genéricos. Agora, se nós pararmos e criassemos uma **INTERFACE**, isso mudaria completamente.

Vamos dar o nome da nossa interface OAuthContract com as seguintes funções: 

```php
interface OAuthContract {

    public function auth(string $code);

    public function getAuthenticatedUser(string $accessToken);
}
```

Se a gente conseguir padronizar as funções, tudo que precisamos fazer é dar um jeito de chamar um serviço que tenha essa interface, porquê aí nos vamos GARANTIR que os métodos estão implementados. Se liga:

```php
// GithubService
$service = new GithubService();
$authData = $service->auth($code);
$response = $authService->getAuthenticatedUser($authData['access_token']);

// DiscordService
$service = new DiscordService();
$authData = $service->auth($code);
$response = $authService->getAuthenticatedUser($authData['access_token']);

// TwitchService
$service = new TwitchService();
$authData = $service->auth($code);
$response = $authService->getAuthenticatedUser($authData['access_token']);
```

Agora pra finalizar, precisamos dizer pro nosso repositório que há um método polimórfico entrando, e o correto pra isso seria tipar o retorno desse metodo com a **INTERFACE**. Você vai retornar

```php

public function getProvider(string $provider): OAuthContract
{
    return match($provider) {
        'discord' => new DiscordService(),
        'twitch' => new TwitchService(),
        'github' => new GithubService()
    };
}
```

Só tendo a ideia que você pode retornar Classes/Interfaces/Tipos, dá pra entender que você vai retornar uma classe que tenha implementado a interface OAuthContract que irá forçar aqueles métodos genéricos estarem presentes. Caso você tente passar uma classe que não tenha essa interface implementada, vai dar merda.

Agora, vamos refatorar o Repositório pra receber essa mudança genérica dentro do nosso projeto.

```php
class AuthRepository {
    
    public function authenticateOAuth(string $provider, string $code) 
    {
        $service = $this->getProvider($provider);
        $authData = $service->auth($code);

        $response = $authService->getAuthenticatedUser($authData['access_token']);
        $authUser = $this->findOrCreate($provider, $response);

        Auth::user($authUser);
        return true;
    }

    public function findOrCreate(string $provider, $providerData): User
    {
        $auth = User::where('email', $providerData['email'])->first();
        if (!$auth) {
            return User::create([
                'name' => $providerData['name'],
                'email' => $providerData['email'],
                $provider . "_id" => $providerData['id'],
            ]);
        }

        if (empty($auth->{$provider . "_id"})) {
            $auth->update([
                $provider . "_id" => $providerData['id']
            ]);
            return $auth;
        }

        if ($auth->{$provider . "_id"} == $providerData['id']) {
            return $auth;
        }

        throw new \Exception('deu ruim');
    }

    public function getProvider(string $provider): OAuthContract
    {
        return match($provider) {
            'discord' => new DiscordService(),
            'twitch' => new TwitchService(),
            'github' => new GithubService()
        };
    }

}
```

Seu software está aberto a extensão, porém fechado para modificação! Congratz você conseguiu chegar até o fim dessa palhaçada.

Se seus Services tiverem lindos, maravilhosos e funcionais, você não vai precisar modificá-los. Mas caso você queira implementar um novo provedor de OAuth, basta criar uma nova classe, implementar a interface e adicionar ao match do getProvider e tá lá. Sem modificar o código antigo, apenas aberto a extensão.

[3. Ir para 'Liskov's Substitution Principle'](3-lsp.md)