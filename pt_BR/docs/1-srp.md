# 1 - Single Responsibility Principle

O Princípio da Responsabilidade Única diz que o software deve ser dividido em blocos, e cada bloco cuida de uma única responsabilidade. Sabe aqueles famosos "códigos de rua", onde você enfia o projeto inteiro num único arquivo ou classe? Sempre dá aquela preguiça de refatorar, porque não tem legibilidade, não tem manutenibilidade e ainda está tudo num único ~~fucking~~ arquivo. É foda, né?

O SRP vem para organizar esse arquivo/projeto monolítico e distribuir o código por responsabilidades. Pense nisso como um uso melhor dos **namespaces** do seu projeto.

```
app
├── Events
│   └── MessageSent.php
├── Http
│   └── Controllers
│       └── MessageController.php
└── Models
    └── Message.php
```

Cada uma das pastas acima tem uma responsabilidade:

- **Controller** → receber, processar e responder requisições;
- **Model** → conversar com o banco de dados;
- **Event** → avisar o resto do sistema que algo aconteceu.

Isso já está bom? Talvez sim, talvez não. Afinal, não sabemos o que está escrito dentro desses arquivos. Será que cada um mantém a sua responsabilidade ou tem mais coisa lá dentro?

> "A class should have only one reason to change."
>
> "Uma classe deve ter apenas um motivo para mudar."

Vamos imaginar um chat qualquer, onde um usuário envia e recebe mensagens. Nesse cenário, vamos salvar todas as mensagens do usuário no banco de dados depois de uma validação.

Vamos usar o ecossistema do Laravel, onde a porta de entrada é o Controller. O Controller tem como responsabilidade:

- Receber a requisição;
- Processar a requisição;
- Responder o cliente.

```php
namespace App\Http\Controllers;

use App\Events\MessageSent;
use App\Models\Message;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

final class MessageController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $data = $request->validate([
            'user_id' => ['required', 'integer', 'exists:users,id'],
            'message' => ['required', 'string', 'max:500'],
        ]);

        $isFlooding = $this->countRepeatedMessages($data['user_id'], $data['message']) >= 5;

        if ($isFlooding) {
            Log::alert('[User Alert] Flooding', $data);
        }

        $message = Message::query()->create($data);
        broadcast(new MessageSent($message));

        return response()->json(['message' => 'Message created.'], 201);
    }

    private function countRepeatedMessages(int $userId, string $content): int
    {
        return Message::query()
            ->where('user_id', $userId)
            ->where('message', $content)
            ->count();
    }
}
```

Vamos listar o que o `MessageController` faz:

- Recebe a requisição;
- Valida os dados de entrada;
- Verifica se deve disparar um alerta de flood;
- Cria um novo registro de mensagem no banco de dados;
- Transmite a mensagem para algum canal;
- Responde o cliente.

Se pensarmos na responsabilidade que um controller deveria ter, dá pra ver que ele foi bem além do esperado.

Vamos refatorar de cima para baixo, começando pela validação. No Laravel, existe um jeito de isolar essa responsabilidade numa classe de **Form Request**.

O comando `php artisan make:request StoreMessageRequest` gera uma classe no namespace `App\Http\Requests` com a responsabilidade **ÚNICA** de validar a requisição, e nada mais:

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

final class StoreMessageRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'user_id' => ['required', 'integer', 'exists:users,id'],
            'message' => ['required', 'string', 'max:500'],
        ];
    }
}
```

Agora vamos usar essa classe no controller:

```php
namespace App\Http\Controllers;

use App\Events\MessageSent;
use App\Http\Requests\StoreMessageRequest;
use App\Models\Message;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Log;

final class MessageController extends Controller
{
    public function store(StoreMessageRequest $request): JsonResponse
    {
        $data = $request->validated();

        $isFlooding = $this->countRepeatedMessages($data['user_id'], $data['message']) >= 5;

        if ($isFlooding) {
            Log::alert('[User Alert] Flooding', $data);
        }

        $message = Message::query()->create($data);
        broadcast(new MessageSent($message));

        return response()->json(['message' => 'Message created.'], 201);
    }

    private function countRepeatedMessages(int $userId, string $content): int
    {
        return Message::query()
            ->where('user_id', $userId)
            ->where('message', $content)
            ->count();
    }
}
```

Beleza, a validação saiu do controller. Agora precisamos levar a regra de negócio para uma nova camada, conhecida como **Service Layer**. A ideia é ter um lugar para as regras de negócio: conversar com o banco, disparar eventos, mandar e-mail e o que mais o seu caso de uso precisar.

> **PS:** a Service Layer não é a última camada de abstração possível. Você pode criar quantas camadas fizerem sentido para deixar o código legível. Só não crie camada por esporte.

```php
namespace App\Services;

use App\Events\MessageSent;
use App\Models\Message;
use Illuminate\Support\Facades\Log;

final class MessageService
{
    private const int FLOOD_LIMIT = 5;

    public function create(array $payload): Message
    {
        if ($this->isFlooding($payload['user_id'], $payload['message'])) {
            Log::alert('[User Alert] Flooding', $payload);
        }

        $message = Message::query()->create($payload);
        broadcast(new MessageSent($message));

        return $message;
    }

    private function isFlooding(int $userId, string $content): bool
    {
        $repeatedMessages = Message::query()
            ->where('user_id', $userId)
            ->where('message', $content)
            ->count();

        return $repeatedMessages >= self::FLOOD_LIMIT;
    }
}
```

Com o service criado e cada responsabilidade no seu lugar, temos três formas de usá-lo no controller:

1. Instanciando a classe direto no método:

    ```php
    $messageService = new MessageService();
    ```

2. Injetando a dependência no construtor da classe:

    ```php
    final class MessageController extends Controller
    {
        public function __construct(
            private readonly MessageService $messageService,
        ) {}
    }
    ```

3. Pedindo a instância ao container do Laravel:

    ```php
    $messageService = app(MessageService::class);
    ```

No nosso código, vamos usar a **Injeção de Dependência** pelo construtor. Para mim, é a opção que mais faz sentido: as dependências da classe ficam explícitas logo no topo, e o Laravel resolve tudo sozinho.

```php
namespace App\Http\Controllers;

use App\Http\Requests\StoreMessageRequest;
use App\Services\MessageService;
use Illuminate\Http\JsonResponse;

final class MessageController extends Controller
{
    public function __construct(
        private readonly MessageService $messageService,
    ) {}

    public function store(StoreMessageRequest $request): JsonResponse
    {
        $this->messageService->create($request->validated());

        return response()->json(['message' => 'Message created.'], 201);
    }
}
```

Com isso, o código ficou bem mais limpo e organizado. Cada responsabilidade foi para o seu lugar, e o controller voltou a fazer só o que combinamos no começo: receber a requisição, repassar o trabalho e responder o cliente.

Agora cada classe tem **um único motivo para mudar**:

| Classe                | Motivo para mudar                                   |
| --------------------- | --------------------------------------------------- |
| `StoreMessageRequest` | As regras de validação mudaram                      |
| `MessageService`      | A regra de negócio mudou (ex.: o limite de flood)   |
| `MessageController`   | O formato da requisição ou da resposta HTTP mudou   |

```
app
├── Events
│   └── MessageSent.php
├── Http
│   ├── Controllers
│   │   └── MessageController.php
│   └── Requests
│       └── StoreMessageRequest.php
├── Models
│   └── Message.php
└── Services
    └── MessageService.php
```

---

## Navegação

[← Introdução](0-introducao.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md) • [5 – Dependency Inversion Principle](5-dip.md)
