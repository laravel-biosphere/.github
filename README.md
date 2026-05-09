# About Laravel Biosphere
**Laravel Biosphere** is a **third-party** package for realtime bi-directional communication
between a server and a client.

## Why not Laravel Reverb
[**Laravel Reverb**](https://reverb.laravel.com) is a nice package, however it is not truly
bi-directional out of the box. Server can send messages to clients,
clients can 'whisper' messages to other clients, however you cannot send a message from a client 
to a server without some ugly workarounds, such as
using an underlying Echo backend, which can (and will)
eventually break on backend change.

On the other hand, Biosphere is a true WebSocket solution,
supporting sending messages both from a client, and from a server.

## How it works
**Biosphere** consists of three parts:
* A client that can listen, and emit events;
* A proxy that passes events back and forth between the client, and the server;
* And a server that, like the client, can listen, and emit events.

Why proxy, you ask? Implementing WebSocket server in PHP is slow for users, and pain for programmers.
Using a proxy powered by [Bun](https://bun.com) accelerates the exchange speed, 
keeping developers in the cozy warmth offered by Laravel.

## Installation
To start using Laravel Biosphere, install its parts:
```bash
composer require laravel-biosphere/server
npm install @laravel-biosphere/proxy
npm install @laravel-biosphere/client
```

Also, to run proxy, you will need Bun:
```bash
# Linux, MacOS
curl -fsSL https://bun.sh/install | bash
```

```bash
# ...sigh, and Windows
powershell -c "irm bun.sh/install.ps1 | iex"
```

## Configuration
Before running Biosphere, you need to configure it.
It's all done in your .env, copy the following, and follow the instructions:
```env
# Used by the proxy to authorize clients on channels;
# We will tell you more about channels later;
# If you run Laravel without Docker, use `localhost` for the domain,
# Otherwise, use you container's name in your Docker network;
BIOSPHERE_LARAVEL_AUTHORIZE_URL=http://<your domain>/biosphere/authorize

# Used by clients to request personal short lived tokens;
# These tokens are requested by a client automatically, and used to authenticate clients in the proxy;
# Laravel generates a token, sends it to the client, and the client can use it during channel authorization,
# for the proxy to ensure that this is a real, trusted request;
BIOSPHERE_LARAVEL_NEW_TOKEN_URL=http://<localhost OR Docker domain>:3000/biosphere/new-token

# These are the names of channels we use in Redis pub/sub;
# You are unlikely to change those, unless it somehow collides with your Redis pub/sub channels;
BIOSPHERE_REDIS_CHANNEL_TO_SERVER=biosphere:to-server
BIOSPHERE_REDIS_CHANNEL_FROM_SERVER=biosphere:from-server

# The server will check the proxy requests using this token;
# This is a secret, do not compromise!
# Use any random string that would be hard to generate;
BIOSPHERE_TOKEN=<any random string>
```

## Running
Biosphere needs to run 2 more processes in your system - the proxy, and the client;
To run the proxy, use:
```bash
bun run ./node_modules/@laravel-biosphere/proxy/src/main.ts & disown
```

And to run the server, use:
```bash
php artisan biosphere:serve
```

## Terminology

> [!TIP]
> If you have experience in Laravel Reverb or Socket.io,
> we use the same terminology, such as channels!

In **Biosphere**, you can define channels.
Channels have names, and users can connect to, and disconnect from them.
Both clients, and the server can send events;
Events can carry any information you want. For example, if you are making a trading platform,
you may wish to emit a price change event from the servers, so everyone stays in sync pricewise.
If event is emitted by a client, it's heard only by the server;
If event is emitted by the server, it's heard by all the clients connected to the channel;

## Usage
### Defining a channel

Let's begin our journey by creating a class for a channel.

> [!TIP]
> I stick to Laravel Reverb's guidelines, so in Biosphere 
> it's recommended for your channel classes to be in `App\Broadcasting\` namespace,
> and have `XxxChannel.php` name.

```php
<?php

namespace App\Broadcasting;

use Anafro\Biosphere\Channels\Channel;
use Anafro\Biosphere\Messages\Message;
use App\Rooms\Room;
use Illuminate\Support\Facades\Log;

class ChatChannel extends Channel
{
    public function __construct(string $pattern, string $name)
    {
        parent::__construct($pattern, $name);
    }

    /**
     * @param \App\Models\User $user
     */
    public function authorize(mixed $user): bool
    {
        //
    }

    /**
     * @param \App\Models\User $user
     */
    public function connected(mixed $user): void
    {
        //
    }

    /**
     * @param \App\Models\User $user
     */
    public function disconnected(mixed $user): void
    {
        //
    }

    public function message(Message $message): void
    {
        //
    }

    /**
     * @param \App\Models\User $user
     */
    public function heartbeat(mixed $user): void
    {
        //
    }
}
```

* `authorize` decides whether a client is allowed to connect to this channel, or not;
It works like `authorize` method in `FormRequest`s.
* `connected` and `disconnected` are self-explanatory - run when clients connect/disconnect;
* `message` is run when this channel receives a message.
* `heartbeat` is run when this channel receives a `pong` event. I will talk about heartbeat later;
Worth to mention now that for most cases it's never used;

### Authorization
Before connecting, we can authorize users.
To do that, implement the `authorize` method.
Return `true` to allow connection, `false` to deny. Simple!

Here are some examples:
```php
/**
 * @param \App\Models\User $user
 */
public function authorize(mixed $user): bool
{
    // Only admins allowed
    return $user->isAdmin;
}
```

```php
/**
 * @param \App\Models\User $user
 */
public function authorize(mixed $user): bool
{
    // If you return true, everyone can join the channel.
    // In Reverb world, such channels are called 'public'.
    return true;
}
```

```php
/**
 * @param \App\Models\User $user
 */
public function authorize(mixed $user): bool
{
    // You can even ignore $user at all,
    // and depend authorization on something else!
    // E.g. you can connect to this channel in January
    // whoever you are.
    return Carbon::now()->month === 1;
}
```

### Receiving events
Any event is received in the `message` method.
A message consists of:
* The name of the event (`$message->event`);
* The event-related data (`$message->data`);
* The user that sent this message (`$message->user`);

By checking the event name, you can handle separate events as you wish.
For example:
```php
public function message(Message $message): void
{
    switch ($message->event) {
    case 'like':
        $username = $message->user->name;
        $video = $message->data['video'];
        Log::info("$username left a like on video $video <3!");
        break;
    case '...':
        ...
    }
}
```

### Sending events
You can send events from the server. To do so, you can use `$this->send` inside channel class.
It will send the message to this channel.

`send` method accepts two arguments:
* An event name to send to this channel;
* And a event data array.

If your event has no data, you can omit the second arguments.

For example:
```php
public function message(Message $message): void
{
    switch ($message->event) {
    case 'like':
        $username = $message->user->name;
        $video = $message->data['video'];

        // It's a common scenario to let the other people know
        // that the event has happened.
        // A client likes a video, that events is sent here,
        // and effectively is sent back to all users,
        // including the person who sent the event;
        // You can filter out that event on client by username for example,
        // so clients ignore `liked` event emitted by themselves,
        // or actually wait for `liked` event to arrive to ensure
        // that the video was liked for real;
        $this->send('liked', compact('username', 'video'));
        break;
    case '...':
        ...
    }
}
```

### Heartbeat system
If WebSocket connection is unused for some period of time (likely for a minute), it closes.
To prevent connection close, Biosphere uses a heartbeat system.
Every 3 seconds, the server sends a `ping` event to all the channels.
Clients connected to channels respond with a `pong` event.
`pong`s trigger the `heartbeat` method instead of `message`.

> [!WARNING]
> This feature is just a possibility left for unusual usages,
> and I cannot provide useful examples for this.

### Plug the channel class in
To make channels available, use `Biosphere::channel` in `routes/channels.php`:

```php
<?php

use App\Broadcasting\ChatChannel;
use App\Models\User;
use Anafro\Biosphere\Facades\Biosphere;

Biosphere::channel("/Chat/", ChatChannel::class);
```

### Defining channel name parameters
Channel names can have parameters. It's useful if you have multiple channels with the same behavior,
for example - chats! You can have multiple chats in your projects with their own names.
Think of it like parameters in web routes, but instead of having `/chat/{name}`, we have:

```php
Biosphere::channel("/Chat #(?<name>[a-zA-Z0-9-_]+)/", ChatChannel::class);
```

> [!TIP]
> If you don't know about `?<>` syntax in regex, read [Mozilla docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Named_capturing_group).
> tl;dr: it gives regex match group names.

### Retrieving channel name parameters
In any channel method, you can use `$this->parameter` method:
```php
public function message(Message $message): void
{
    switch ($message->event) {
    case 'chat message':
        // If your channel name was "Chat #cool_chat_1337",
        // the following will be 'cool_chat_1337':
        $chatName = $this->parameter('name');

        $this->send('liked', compact('username', 'video'));
        break;
    case '...':
        ...
    }
}
```

### Scheduling events
You can schedule event dispatch by using `$this->schedule` inside a channel.
```php
public function message(Message $message): void
{
    switch ($message->event) {
    case 'timer':
        // ID is for cancellation - IDs must be unique, so we suggest
        // using something like UUIDs;
        // Read about cancellation below.
        $id = Str::uuid();
        $this->schedule($id, 'ding-dong', 60000, [
            'message' => 'Time\'s up!',
        ]);
        break;
    case '...':
        ...
    }
}
```

> [!WARNING]
> Events sent via `schedule` method **are sent back to the server**!
> Use `message` method to handle them!

> [!WARNING]
> Messages triggered via `schedule` won't have `user`!

### Cancelling scheduled events
You can cancel scheduled event by call `$this->cancel($id)`:
```php
public function message(Message $message): void
{
    switch ($message->event) {
    case 'stop timer':
        // You need an ID to cancel the specific scheduled event.
        // You might want to send the ID to the user,
        // or store it in Redis, you name it!
        $id = $message->data['id'];
        $this->cancel($id);
        break;
    case '...':
        ...
    }
}
```
> [!INFO]
> Cancellation of non-existent scheduled, or already emitted events
> is silently ignored.

### Connecting to Biosphere channels
To connect to Biosphere, import and use Biosphere client:

```ts
import { biosphere } from "@laravel-biosphere/client";

const chat = await biosphere.channel("Chat #cool_chat_1337");
```

### Listening to server events
To listen, use `on` method of a channel object:
```ts
import { biosphere } from "@laravel-biosphere/client";

const chat = await biosphere.channel("Chat #cool_chat_1337");
chat.on(/chat message/, message => {
    console.info(`New chat message from ${message.username}: ${message.text}`);
    // It would receive $this->send('chat message', compact('username', 'text')) from the server.
});
```

To stop listening, use `off`:
```ts
chat.off(/chat message/);
```

> [!WARNING]
> `.off` does not match listener regexes, but strictly compares them.
> So, `.off(/.*/)` WILL NOT remove all listeners as you could expect.
> Instead, it will remove listeners that was exactly declared as `.on(/.*/)`.

To stop listening to all events, use `.offAll()`.

### Sending events to server
To emit an event to a server, use `.send`. It is identical to the server's one:
```ts
chat.send(/chat message/, {
    text: 'Yooo, hows that goiinnng tiger?',
});
```

> [!INFO]
> User is added to the message by the proxy,
> so you must not include `userId` as in data array.

### Close connection with a channel
To close the connection, yep, use `.close()`:
```ts
chat.close();
```

> [!WARNING]
> Channels cannot be reopened.
> Instead, create another channel object via `await biosphere.channel(name)`.

## License
MIT, except for everything related to Laravel team.
Please, consult their README to know about their license.
