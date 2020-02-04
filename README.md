
# Telegram Bots in Django

Bot API version 4.6 ([Telegram bot API docs](https://core.telegram.org/bots/api))

### Quick Setup

1. Install package from pip:  
```
pip install django-tgbot
```

2. Add `django_tgbot` to your Django project's `INSTALLED_APPS`

That's it :) You installed django-tgbot.

<hr>

### Create a new Telegram bot
1. Create a bot in Telegram using [BotFather](https://t.me/BotFather) and recieve your API token
2. Open the Django project with `django-tgbot` installed in it
3. Enter this command in the command line (terminal / cmd):
```
python manage createtgbot
```
4. Enter your API token:
```
> python manage.py createtgbot
Enter the bot token (retrieved from BotFather): <YOUR_TOKEN>
Setting up @BotDevTestBot ...
```
5. Enter the URL your Django project is deployed on. If your project is not deployed yet and is not accessible, press Enter to skip. (If you have not deployed yet and want to test your bot, you can use services like [Ngrok](http://ngrok.com) to do so)
```
Enter the url of this project to set the webhook (Press Enter to skip): https://URL.com
Bot webhook will be set to https://URL.com/botdevtestbot/update/. Do you confirm? (Y/N): y
Webhook was successfully set.
```

6. A new app will be created in your Django project. Add this app to your `INSTALLED_APPS`
7. Include this new app's urls in your project urls as described in the output of the above command
8. Update the database:
```
python manage.py migrate
```

Your bot is created! If you have set the webhook correctly you can now send messages to your bot and it responds all messages with `Hello!`.

<b> Overview of the process: </b>

```
> python manage.py createtgbot
Enter the bot token (retrieved from BotFather): 521093911:AAEe6X-KTJHO98tK2skJLsYJsE7NRpjL8Ic
Setting up @BotDevTestBot ...
Enter the url of this project to set the webhook (Press Enter to skip): https://URL.com
Bot webhook will be set to https://URL.com/botdevtestbot/update/. Do you confirm? (Y/N): y
Webhook was successfully set.
Successfully created bot Test Bot(@botdevtestbot).
Next steps:
	1. Add 'botdevtestbot' to INSTALLED_APPS in project settings
	2. Add `from botdevtestbot import urls as botdevtestbot_urls` to project urls file
	3. Add `path('botdevtestbot/', include(botdevtestbot_urls))` to project urls' urlpatterns
	4. `python manage.py migrate`
Enjoy!
```

<hr>

### Definitions

This is an overview of the flow:

Bot receives a message/update --> Telegram sends an Update to your webhook --> Django receives this request and passes it to the app created for this bot (In steps above) --> `django_tgbot` creates an `Update` object from the HTTP request --> One or several `processor`s will be assigned to handle this `Update` --> All of these `processor`s run and possibly make some calls to Telegram API.

#### Types
Telegram uses some methods and `Type`s (read classes) to handle everything. All of these types are implemented as Python classes and you can use them from `django_tgbot.types`. You can view a full list of all Telegram available types [here](https://core.telegram.org/bots/api#available-types) (The API version based on which this package is designed is written at the top of this document. With newer versions there might be new types not implemented in this package)

#### Models
With every new bot you create come 3 models:
* <b>TelegramUser</b>: represents a user in Telegram. Can be a person or a bot. Basically, it is an entity that can send messages.
* <b>TelegramChat</b>: represents a chat in Telegram. Can be a private chat, a group, a supergroup or a channel. Basically, it is a place that messages can be sent from one or several parties.
* <b>TelegramState</b>: From the bot's perspective, each user in a chat or a chat by itself or a user by itself, are in a state. This state is stored in `TelegramState` model. It holds the user (can be blank), the chat (can be blank), a memory and a name. This name helps the bot to easily determine what this state is and what needs to be done.

#### Bot
When you create a new bot, an app will be created which has a new class named `TelegramBot` and a bot instantiated from this class. This is your interface for working with the Telegram Bot API.

For every request it receives, it will do some preprocessing and then finds the `processor`s responsible for this request and runs all of them. Finally, it will do some post-processings.

The mentioned preprocessing and post-processings can be changed or removed from the Bot class in `bot.py` in your newly created app. By default, for all requests, the user's `id`, `first_name`, `last_name` and `username` and the chat's `id`, `title` and `type` will be stored in database in the preprocessing and nothing is done in the postprocessing.

#### Processors

This is the core of your bot's functionality and for this you write most of your codes.

So far, we have seen every client of our bot (a user with/without a chat or a chat itself) has a state (i.e. is in a state). Processors take a client with its state and based on their state and the sent update, they will forward this client to another state.

Each processor should declare the states from which clients can enter and the state to which clients should be sent in case processor ran successfully or in case it failed.

Processors are just Python functions that take as input the bot instance, the update received from Telegram and the state of the client.


<hr>

### Developing your bot

When you create a new bot, a new app will be created in your Django project which will contain a file named `processors.py`. This module is where all your processors lie. You can also remove this file and instead, create a package with the same name in the same directory since the number of processors might get large and it's a good idea to keep them separated in different modules. If you do so, do not forget to import all of these modules in the `__init__.py` of `processors` package.  
If you replace `processors.py` with a `processors` package:

```
processors
├── __init__.py
├── greetings.py
├── signup.py
└── ...
```

Where `__init__.py` contains:
```
from . import greetings, signup, ...
```

Initially, there is a `hello_world` processor available in `processors.py`. As you can see, all of your processors should get registered by `@processor` decorator in order to be recognized later by the bot. They should also take three named parameters (like the `hello_world` sample):
* <b>bot</b>: An instance of the TelegramBot. Can be used to call Telegram API (sending messages, etc.)
* <b>update</b>: The received update loaded into the `Update` type explained above.
* <b>state</b>: The state of this client, a TelegramState instance. You can change their state or memory using this.

As said earlier, each processor should declare what states it accept to process and if processing was done successfully, what should the client's state become and what should it become if the processing fails.

These are declared above the function definition in the `@processor` arguments:
* <b>state_names</b>: Name of the accepting states for this processor. It can be a <i>string</i>, a <i>list</i> or `state_types.All` which will accept all states. If you want to accept the empty state (the client's state is initially empty and it's a good idea to use empty string as a reset state), leave it or set it as `state_names = ''`.
* <b>success</b>: The new state of the client, if processor runs successfully. Here successfully means being run without raising `ProcessFailure` exception.
* <b>fail</b>: The new state of the client, if processor fails to run. If you want to fail the processor you should raise `ProcessFailure`. This exception can be imported from `django_tgbot.exceptions`.

You may use `state_types.Keep` as the value for `success` or `fail` to not change the state or use `state_types.Reset` to reset the state (equivalent as saying `success = ''`).

Additionally, you can define the update types and the message types you want this processor to handle. For example, you may want to have a processor that only handles requests regarding a message getting edited (which is a different update type than receiving a new message) or you may want to have a processor only handling requests of video messages (which is a different message type than text messages). To see a full list of different updates types and message types read Telegram Bot API docs. Parameters for `@processor`:
* <b>message_types</b>: Can be a single value or a list of values. Leave unfilled to accept any message type. Use values from `message_types` module to fill in the parameter. For example you can say `message_types = message_types.Text` to only handle text messages or `message_types = [message_types.NewChatMembers, message_types.LeftChatMembers]` to handle updates about group members coming and going.
* <b>exclude_message_types</b>: Works exactly like `message_types` except that values passed here will be excluded from the valid message types to handle.
* <b>update_types</b>: Can be a single value or a list of values. Leave unfilled to accept any update type. Like the message_updates, available update_types are accessible from the `update_types` module. For example, `update_types = [update_types.ChannelPost, update_types.EditedMessage]` makes the processor handle only updates about a new post being sent to a channel or a message being edited.
* <b>exclude_update_types</b>: Works exactly like `update_types` except that values passed here will be excluded from the acceptable update types to handle.

Please note that the first parameter for `@processor` should be always an state manager. An state manager is created automatically when you create a new bot and it's imported in the processors module. You can use that and give it to all of the processors. You may change this state manager for having different behaviors in your bot, which is not a common case and will be explained in advanced documentations.

Example of a processor definition:
```python
@processor(state_manager, from_states='asked_for_name', success='got_their_name', fail=state_types.Keep, message_types=message_types.Text, update_types=update_types.Message)
def say_hello(bot, update, state):
    bot.sendMessage(update.get_chat().get_id(), "Hello {}!".format(update.get_message().get_text()))
```

You may also leave the `success` and `fail` arguments in order to not change the state automatically, if you want to change it yourself in the processor:

```python
@processor(state_manager, from_states='asked_for_name', message_types=message_types.Text, update_types=update_types.Message)
def say_hello(bot, update, state):
	text = update.get_message().get_text()
    if text == 'Alireza':
   		bot.sendMessage(update.get_chat().get_id(), "Hello Alireza!")
        state.name = 'got_their_name'
        state.save()
    else:
    	bot.sendMessage(update.get_chat().get_id(), "Nah")
        state.name = 'failed_to_give_name'
        state.save()
```

Please note that leaving the `success` and `fail` parameters is NOT the same as setting them to `state_types.Keep`. Leaving them will not change them and allows you to set them in the processor's run time. However, setting them to `state_types.Keep` will force the state to be the same as what is was before entering the processor.

A demo bot created with `django-tgbot`: [https://github.com/ARKhoshghalb/django-tgbot_demo](https://github.com/ARKhoshghalb/django-tgbot_demo)  
Full documentation: [https://django-tgbot.readthedocs.io/en/latest/](https://django-tgbot.readthedocs.io/en/latest/)