# 1 - Single Responsibility Principle

O Princípio da Responsabilidade Única se dá a ideia de que o software deverá ser particionado em blocos de responsabilidade dentro do ecossistema produzido. Quando vemos os famosos "códigos de rua", onde você enfia todo o projeto num único arquivo ou classe, sempre dá aquela preguiça de refatorar porquê não tem legibilidade nem manutenibilidade além de estar tudo em um único ~fucking~ arquivo. É foda né?

O SRP vem para organizar melhor esse arquivo/projeto monolito, e descentralizar o código em responsabilidades. Pense nisso como um melhor uso dos **namespaces** dentro do seu projeto.

```
App
├── Http
│   └── Controllers
│       └── MessagesController.php
├── Models
│   └── Message.php
├── Events
│   └── ChatMessage.php
```

Cada uma das pastas acima tem uma responsabilidade. Sendo elas:

- Controller -> Receber, processar e responder requisições;
- Model -> Comunicar com o banco de dados;
- Event -> Disparar mensageria.

Podemos dizer que isso está interessante? Talvez sim, talvez não. Afinal, não sabemos o que está escrito dentro desse código. Será que eles realmente mantêm a responsabilidade ou existem mais coisas dentro?

<center>
"A class should have only one reason to change"

"Uma classe deve ter apenas uma razão para mudar"
</center>

Vamos imaginar um cenário onde temos um chat qualquer e nele temos um usuário que mande e receba mensagens. Nesse cenário, nós vamos salvar todas as mensagens do nosso usuário no banco de dados após uma validação.

Iremos utilizar o ecossistema do Laravel, onde temos como entrada o Controller. O Controlador tem como responsabilidade:

- Receber uma requisição;
- Processar a requisição;
- Responder o cliente.

Segue o snippet:

```php
namespace App\Http\Controllers;

use DB;
use Illuminate\Foundation\Http\Request;
use App\Events\ChatMessage;

class MessagesController extends Controller {

    public function postMessage(Request $request) {
        $this->validate($request, [
            'user_id' => 'required|exists:users,id',
            'message' => 'required'
        ]);

        if ($this->getUserSpecificMessagesCount($data['message'])) {
            Log::alert('[User Alert] Flooding', $data);
        }

        $model = Message::create($request->all());
        broadcast(new ChatMessage($model));

        return response()->json(['message' => 'message created'], 201);
    }

    public function getUserSpecificMessagesCount(int $userId, string $message) {
        return DB::table('user_messages')->where([
            ['user_id', '=', $userId],
            ['message', '=', $message],
        ])->count();
    }

}
```

Vamos listar o que podemos ver no snippet do MessagesController:

- Ele recebe uma requisição;
- Valida a entrada dos dados;
- Checa se deve disparar um alerta de flooding;
- Cria um novo registro de mensagem no banco de dados;
- Transmite uma mensagem para algum canal;
- Responde o cliente.

Agora pensando na responsabilidade que o controller deveria ter, podemos ver que foi um pouco além e isso não é o esperado.

Vamos começar a refatorar de cima para baixo, começando pela validação. No ecossistema Laravel, existe um jeito de validar requisições onde você isola a responsabilidade em uma classe de Request.

Utilizando o comando **php artisan make:request CreateMessageRequest** você irá gerar uma classe validadora que ficará no namespace **App\Http\Requests** que terá a responsabilidade **ÚNICA** de validar sua requisição e nada mais:

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CreateMessageRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     *
     * @return bool
     */
    public function authorize()
    {
        return true;
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
            'user_id' => 'required|exists:users,id',
            'message' => 'required'
        ];
    }
}
```

Agora nós vamos implementar isso no nosso snippet acima.

```php
namespace App\Http\Controllers;

use DB;
use App\Http\Requests\CreateMessageRequest;
use App\Events\ChatMessage;

class MessagesController extends Controller {

    public function postMessage(CreateMessageRequest $request) {
        $data = $request->validated();

        if ($this->getUserSpecificMessagesCount($data['message'])) {
            Log::alert('[User Alert] Flooding', $data)
        }

        $model = Message::create($data);
        broadcast(new ChatMessage($model));

        return response()->json(['message' => 'message created'], 201);
    }

    public function getUserSpecificMessagesCount(int $userId, string $message) {
        return DB::table('user_messages')->where([
            ['user_id', '=', $userId],
            ['message', '=', $message],
        ])->count();
    }

}
```

Beleza, separamos a nossa validação da função principal. Agora precisamos extrair a regra de negócio para uma nova camada de abstração, que é conhecida como **Repository Pattern**. A ideia do Repository Pattern é você ter um local para trabalhar funções que se comunicam com banco de dados, envio de mensageria e mais o que você quiser. É literalmente onde você injeta suas regras de negócio.

PS: O Repository Pattern não é a última camada de abstração, na realidade você pode abstrair quantas você quiser para que o seu código fique o mais legível possível.

```php
namespace App\Repositories;

use App\Models\Message;
use App\Events\ChatMessage;

class MessageRepository {

    private $model;

    public function __construct()
    {
        $this->model = new Message();
    }

    public function create(array $payload): bool
    {
        if ($this->checkFloodPossibility($data['message'])) {
            Log::alert('[User Alert] Flooding', $data)
        }

        $model = Message::create($payload);
        broadcast(new ChatMessage($model));

        return true;
    }

    public function getUserSpecificMessagesCount(int $userId, string $message) {
        return DB::table('user_messages')->where([
            ['user_id', '=', $userId],
            ['message', '=', $message],
        ])->count();
    }
}
```

Após nós criarmos nosso repositório e jogarmos toda a responsabilidade devida nele, vamos ter três maneiras de colocar ele no nosso controller

1. Instanciando a classe direto na função

    ``` php
    $repository = new MessageRepository();
    ```
2. Injetando a dependência no construtor da classe e

    ```php
    class MessagesController {
        public $repository;

        public function __construct(MessageRepository $repository)
        {
            $this->repository = $repository;
        }
    }
    ```
3. Usando os containers do Laravel

    ```php
    $repository = app(MessageRepository::class)->create();
    ```

No nosso código, iremos utilizar a Injeção de Dependência para que possamos ter uma visão melhor do código. Particularmente é a que mais faz sentido pra mim, tendo em vista que temos que manter o código organizado.


```php
namespace App\Http\Controllers;

use App\Http\Requests\CreateMessageRequest;
use App\Repositories\MessageRepository;

class MessagesController extends Controller {

    private $repository;

    public function __construct(MessageRepository $repository)
    {
        $this->repository = $repository;
    }

    public function postMessage(CreateMessageRequest $request)
    {
        $data = $request->validated();

        $this->repository->create($data);

        return response()->json(['message' => 'message created'], 201);
    }

}
```

Com isso, nosso código ficou bem mais limpo e organizado. Cada uma das responsabilidades foram distribuídas e o código do controlador foi concluído como o especificado antes de começar o código, onde você: recebe a requisição, passa a responsabilidade após a validação e responde o cliente.

```
App
├── Http
│   └── Controllers
│       └── MessagesController.php
│       Requests
│       └── CreateMessageRequest.php
├── Models
│   └── Message.php
├── Events
│   └── ChatMessage.php
└── Repositories
    └── MessageRepository.php
```


[2. Ir para 'Open Closed Principle'](2-ocp.md)

