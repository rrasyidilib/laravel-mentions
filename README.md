# End-to-end Mentions in Laravel 5

This Laravel >=5.4 package provides an easy way to setup mentions for Eloquent models. It provides the front-end for inserting mentions into **content-editable** elements, the back-end for associating mentions with models and lastly an elegant way to notify the mentioned models that they have been mentioned.

Here are a few short examples of what you can do:

```php
// Mention a single user
$comment->mention($user);

// Mention a collection of users
$comment->mention($users);

// Handle the form data
$comment->mention($request->mentions);

// Get all mentions, resolved to their models
$comment->mentions();

// Unmention a single user
$comment->unmention($user);

// Unmention a collection of users
$comment->unmention($users);
```

It handles notifications for you as well. If your mentions config has `auto_notify` enabled, it will do it for you. If you need to handle the logic yourself, disable `auto_notify` and explicitly notify the mention, for example:

```php
$mention = $comment->mention($user);
$mention->notify();
```

## Requirements

- [jQuery](https://jquery.com/)
- [Tribute](https://github.com/zurb/tribute)

## Installation

You can install this package via composer using this command:

```bash
composer require jameslkingsley/laravel-mentions
```

Next, you must install the service provider in `config/app.php`:

```php
'providers' => [
    ...
    Kingsley\Mentions\MentionServiceProvider::class,
];
```

Now publish the migration, front-end assets and config:

```bash
php artisan vendor:publish
```

After the migration has been published you can create the mentions table by running the migrations:

```bash
php artisan migrate
```

This is the contents of the published config file:

```php
return [
    // Pools are what you reference on the front-end
    // They contain the model that will be mentioned
    'pools' => [
        'users' => [
            // Model that will be mentioned
            'model' => 'App\User',

            // The column that will be used to search the model
            'column' => 'name',

            // Notification class to use when this model is mentioned
            'notification' => 'App\Notifications\UserMentioned',

            // Automatically notify upon mentions
            'auto_notify' => true
        ]
    ]
];
```

## Usage

Start off by including the front-end assets; **don't forget to include Tribute and jQuery!**

```html
<script type="text/javascript" src="/js/laravel-mentions.js"></script>
<link rel="stylesheet" type="text/css" href="/css/laravel-mentions.css">
```

Now let's setup the form where we'll write a comment that has mentions:

```html
<form method="post" action="{{ route('comments.store') }}">
    <!-- This field is required, it stores the mention data -->
    <input type="hidden" name="mentions" value="">

    <!-- We write the comment in the div -->
    <!-- The for attribute is used by laravel-mentions to auto-populate the textarea -->
    <textarea class="hide" name="text"></textarea>
    <div class="has-mentions" contenteditable="true" for="text"></div>

    <button type="submit">
        Post Comment
    </button>

    <!-- CSRF field for Laravel -->
    {{ csrf_field() }}
</form>
```

Next add the script to initialize the mentions:

```html
<script>
    $(document).ready(function(e) {
        $('.has-mentions').mentions([{
            // Trigger the popup on the @ symbol
            trigger: '@',

            // Pool name from the mentions config
            pool: 'users',

            // Same value as the pool's 'column' value
            display: 'name',

            // The model's primary key field name
            reference: 'id'
        }]);
    });
</script>
```

Now onto the back-end. Choose the model that you want to assign mentions to. In this example I'll choose `Comments`. We'll import the trait and use it in the class.

```php
namespace App;

use Illuminate\Database\Eloquent\Model;
use Kingsley\Mentions\HasMentionsTrait;

class Comment extends Model
{
    use HasMentionsTrait;
}
```

Next switch to the controller for where you store the comment. In this case it's `CommentController`. Create the comment however you like, and afterwards just call the `mention` method.

```php
public function store(Request $request)
{
    // Handle the comment creation however you like
    $comment = Comment::create($request->all());

    // Call the mention method with the form mentions data
    $comment->mention($request->mentions);
}
```

That's it! Now when displaying your comments you can style the `.mention-node` class that is inserted via Tribute. That same node also has a `data-object` attribute that contains the pool name and reference value, eg: `users:1`.

### Notifications

If you want to use notifications, here's some stuff you may need to know.

- When a mention is notified, it will use Laravel's built-in Notification trait to make the notification. That means the model class defined in the pool's config must have the `Notifiable` trait.
- It will use the notification class defined in the pool's config, so you can handle it differently for each one.
- The data stored in the notification will always be the model that did the mention, for example `$comment->mention($user)` will store `$comment` in the data field.