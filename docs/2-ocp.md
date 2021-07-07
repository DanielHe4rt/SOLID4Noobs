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