# 1 - Single Responsibility Principle

The Single Responsibility Principle says that software should be split into blocks, and each block takes care of a single responsibility. You know that famous "dirty code", where the whole project is stuffed inside a single file or class? You always feel too lazy to refactor it, because it's neither readable nor maintainable, and on top of that everything lives in one ~~fucking~~ file. It sucks, right?

SRP comes to organize that monolithic file/project and spread the code across responsibilities. Think of it as a better use of **namespaces** in your project.

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

Each of the folders above has a responsibility:

- **Controller** → receive, process, and respond to requests;
- **Model** → talk to the database;
- **Event** → tell the rest of the system that something happened.

Is this already good? Maybe yes, maybe not. After all, we don't know what is written inside these files. Does each one really keep its responsibility, or is there more stuff inside?

> "A class should have only one reason to change."

Let's imagine a random chat where a user sends and receives messages. In this scenario, we're going to save every message from the user in the database after a validation.

We're going to use the Laravel ecosystem, where the entry point is the Controller. The Controller is responsible for:

- Receiving the request;
- Processing the request;
- Responding to the client.

Here is the snippet:

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

Let's list what the `MessageController` does:

- Receives the request;
- Validates the input;
- Checks whether it should fire a flooding alert;
- Creates a new message record in the database;
- Broadcasts the message to some channel;
- Responds to the client.

If we think about the responsibility a controller should have, we can see it went way beyond what was expected.

Let's refactor from top to bottom, starting with the validation. Laravel has a way to isolate this responsibility in a **Form Request** class.

The command `php artisan make:request StoreMessageRequest` generates a class in the `App\Http\Requests` namespace with the **SINGLE** responsibility of validating the request, and nothing else:

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

Now let's use this class in the controller:

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

Alright, the validation is out of the controller. Now we have to move the business rules into a new layer, known as the **Service Layer**. The idea is to have a place for your business rules: talking to the database, dispatching events, sending emails, and whatever else your use case needs.

> **PS:** the Service Layer is not the last possible abstraction layer. You can create as many layers as make sense to keep your code readable. Just don't create layers for sport.

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

With the service in place and every responsibility where it belongs, we have three ways to use it in the controller:

1. Instantiating the class directly inside the method:

    ```php
    $messageService = new MessageService();
    ```

2. Injecting the dependency through the class constructor:

    ```php
    final class MessageController extends Controller
    {
        public function __construct(
            private readonly MessageService $messageService,
        ) {}
    }
    ```

3. Asking the Laravel container for an instance:

    ```php
    $messageService = app(MessageService::class);
    ```

In our code, we're going to use constructor **Dependency Injection**. To me, it's the option that makes the most sense: the class dependencies are explicit right at the top, and Laravel resolves everything for you.

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

With that, our code became a lot cleaner and more organized. Each responsibility went to its place, and the controller went back to doing only what we agreed on at the beginning: receive the request, hand off the work, and respond to the client.

Now each class has **a single reason to change**:

| Class                 | Reason to change                                     |
| --------------------- | ---------------------------------------------------- |
| `StoreMessageRequest` | The validation rules changed                         |
| `MessageService`      | The business rule changed (e.g. the flooding limit)  |
| `MessageController`   | The HTTP request or response format changed          |

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

## Navigation

[← Introduction](0-introduction.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md) • [5 – Dependency Inversion Principle](5-dip.md)
