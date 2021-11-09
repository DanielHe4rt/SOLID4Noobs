# 1 - Single Responsibility Principle

The 'Single Responsibility Principle' has the idea of the software should be partitioned in responsibility block inside the project ecossystem. When we see the famous "dirty code", where you stick all the code inside a single file or class, you'll always have that lazyness to refactoring it because the code is not readable/mainantanable, besides that it is in a whole ~fucking~ file or class. There is fuck, isn't?


SRP came to organize better these monolith class/functions, and descentralize the code responsibilities. Think in that as a better use of **namespaces** inside your project.

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

Each of the folders above has a responsibility, being them:

- Controller -> Receive, process and return responses for requests;
- Model -> Communicate with Database;
- Event -> Dispatch Events.

We can say that is already good? Maybe yes, maybe not. After all, we don't know what is written inside these files.
Does they really maintain the responsibility or exists more stuff inside?

Podemos dizer que isso está interessante? Talvez sim, talvez não. Afinal, não sabemos o que está escrito dentro desse código. Será que eles realmente mantêm a responsabilidade ou existem mais coisas dentro?

<p class="text-align: center;">
    "A class should have only one reason to change"
</p>

Lets imagine a scenario where we have a random chat and there we have an user that sends and receives messages. On this scenario, we're going to save all messages from our user on database after the validation.

We're going to use the Laravel ecossystem, where the request entrance will be the Controller. 
The Controller has as responsibility:

- Receive a Request;
- Process a Request;
- Return a Response for the Request.

Here is the snippet:

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

        if ($this->getUserSpecificMessagesCount($data['message']) >= 5) {
            Log::alert('[User Alert] Flooding', $data)
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

Let's list what the can see on the MessagesController snippet

- Receive the request;
- Validate the data;
- Check a possibility of flooding;
- Create a new register of the message on Database;
- Broadcast the message to some channel;
- Return a message to the client.


Now thinking in the responsibility that controller should have, we can see that was a little bit far from that and it was not expected.

Let's start to refactoring from top to bottom, starting by the validation. On the Laravel ecossystem, there's a way to validate requests where you isolate the responsibility in a FormRequest Class.

Using the command **php artisan make:request CreateMessageRequest** you will generate a FormRequest class that will appear on the folder/namespace **App\Http\Requests** that will have the **UNIQUE** responsibility of validate your request and nothing else:

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

Now we're going to implement the solution on our snippet above and it should look like this:

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

Alright, we separated the validation of our main function. Now we have to extract the business rule to a new abstractiong layer, that is known as **Repository Pattern**. The idea of Repository Pattern is you have a place to work with methods/classes that communicates with database, mailing and whatever else you need it. It's literally where you put all the business logic (if you prefer).

PS: The Repository Pattern is not the last abstractiong layer, in the reality you can abstract how many layers you want for your code being more readable as possible.

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
After we create our repository and throw all the due responsibility there, we'll have three ways to invoke it on our controller:

1. Instantiate directly inside the function

    ``` php
    $repository = new MessageRepository();
    ```
2. Injecting the dependency on the class constructor 

    ```php
    class MessagesController {
        public $repository;

        public function __construct(MessageRepository $repository)
        {
            $this->repository = $repository;
        }
    }
    ```
3. Using Laravel Containers

    ```php
    $repository = app(MessageRepository::class)->create();
    ```

In our code, we're going to use Dependency Injection so we can have a better view of the code.
Particularly is the one who makes more sense to me, have in sight that we have to maintaing the the code cleaner as possible.


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

With that, our code become a lot more organized and cleaner to read. Each one of responsibilities was distributed
and the Controller responsibilities was followed like we told on the beginning of this article: receive, process and response. 

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


[2. Go to 'Open Closed Principle'](2-ocp.md)
