![Goh Rhy Yan - Unsplash #y8CtjK0ej6A](https://images.unsplash.com/photo-1496327249223-c84a3c1db090?ixlib=rb-1.2.1&auto=format&fit=crop&w=1200&h=400&q=80)

[![Latest Version on Packagist](https://img.shields.io/packagist/v/darkghosthunter/laraload.svg?style=flat-square)](https://packagist.org/packages/darkghosthunter/laraload) [![License](https://poser.pugx.org/darkghosthunter/laraload/license)](https://packagist.org/packages/darkghosthunter/laraload)
![](https://img.shields.io/packagist/php-v/darkghosthunter/laraload.svg)
 ![](https://github.com/DarkGhostHunter/Laraload/workflows/PHP%20Composer/badge.svg)
[![Coverage Status](https://coveralls.io/repos/github/DarkGhostHunter/Laraload/badge.svg?branch=master)](https://coveralls.io/github/DarkGhostHunter/Laraload?branch=master)

# Laraload

Effortlessly create a PHP 7.4 Preload script for your Laravel project.

## Requirements

* Laravel 6 or Laravel 7
* PHP 8, PHP 7.4.3 or later
* `ext-opcache`

> The Opcache extension is not enforced by the package. Just be sure to enable it in your project's PHP main process.

## Installation

Call composer and you're done.

```bash
composer require darkghosthunter/laraload
```

## What is Preloading? What does this?

Preloading is a new feature for PHP 8, PHP 7.4 and Opcache. It "compiles" a list of files into memory, thus making the application code _fast_ without warming up. For that to work, it needs to read a PHP script that uploads the files, at startup.

This package wraps the Preloader package that generates a preload file. Once it's generated, you can point the generated list into your `php.ini`:

```ini
opcache.preload = 'www/app/storage/preload.php';
```

## Usage

By default, this package constantly recreates your preload script each 500 requests in `storage/preload.php`. That's it. But you want the details, don't you?

1. A global terminable middleware checks for non-error response.
2. Then it calls a custom *Condition* class.
3. If the *Condition* evaluates to `true`, the script is generated.
4. A `PreloadCalledEvent` is called with the generation status.

## Configuration

Some people may not be happy with the "default" behaviour. Luckily, you can configure your own way to generate the script.

First publish the configuration file:

```bash
php artisan vendor:publish --provider="DarkGhostHunter\Laraload\LaraloadServiceProvider"
```

Let's check config array:

```php
<?php

return [
    'condition' => \DarkGhostHunter\Laraload\Conditions\CountRequests::class,
    'output' =>  storage_path('preload.php'),
    'memory' => 32,
    'use_require' => false,
    'autoload' => base_path('vendor/autoload.php'),
    'ignore-not-found' => true,
];
```

#### Condition

```php
<?php

return [
    'condition' => 'App\MyCustomCondition@handle',
];
```

This package comes with a _simple_ condition class that returns `true` every 500 requests, which triggers the script generation. 

You can define your own condition class to generate the Preload script. This will be called after the request is handled to the browser, and it will be resolved by the Service Container.

The condition is called the same way as a Controller action: as an invokable class or using _Class@action_ notation.

#### Output

```php
<?php

return [
    'output' => '/var/www/preloads/my_preload.php',
];
```
 
By default, the script is saved in your storage path, but you can change the filename and path to save it as long PHP has permissions to write on it.

#### Memory Limit

```php
<?php

return [
    'memory' => 64,
];
```

For most applications, 32MB is fine as a preload limit, but you may fine-tune it for your project specifically.

#### Method

```php
<?php

return [
    'use_require' => true,
    'autoload' => base_path('vendor/autoload.php'),
];
```

Opcache allows to preload files using `require_once` or `opcache_compile_file()`.

From version 2.0, Laraload now uses `opcache_compile_file()` for better manageability on the files preloaded. Some unresolved links may output warnings, but nothing critical. 

Using `require_once` will execute all files, resolving all the links (parent classes, traits, interfaces, etc.) before compiling it, and may output heavy errors on files that shouldn't be executed. Depending on your application, you may want to use one over the other.

If you plan use `require_once`, ensure you have set the correct path to the Composer Autoloader, since it will be used to resolve classes, among other files.

### Ignore not found files

```php
<?php

return [
    'ignore-not-found' => true,
];
```

Version 2.1.0 and onward ignores non-existent files by default. This may work for files created by Laravel at runtime and actively cached by Opcache, but that on deployment are absent, like [real-time facades](https://laravel.com/docs/facades#real-time-facades).

You can disable this for any reason, but is recommended leaving it alone unless you know what you're doing (and why).

### Include & Exclude directories

For better manageability, you can now append or exclude files from directories using the [Symfony Finder](https://symfony.com/doc/current/components/finder.html), which is included in this package, to retrieve a list of files inside of them with better filtering options.

To do so, add an `array` of directories, or register a callback receiving a Symfony Finder instance to further filter which files you want to append or exclude. You can do this in your App Service Provider by using the `Laravel` facade (or injecting Laraload).

```php

use Symfony\Component\Finder\Finder;
use Illuminate\Support\ServiceProvider;
use DarkGhostHunter\Laraload\Facades\Laraload;

class AppServiceProvider extends ServiceProvider
{
    // ...
    
    public function boot()
    {
        Laraload::append(function (Finder $find) {
            $find->in('www/app/vendor/name/package/src')->name('*.php');
        });
        
        Laraload::exclude(function (Finder $find) {
            $find->in('www/app/resources/views')->name('*.blade.php');
        });
    }
}
```

### FAQ

* **The package returns errors when I used it!**
  
Check you're using latest PHP 7.4 version (critical), and Opcache is enabled. Also, check your storage directory is writable.

As a safe-bet, you can use the safe preloader script in `darkghosthunter/preloader/helpers/safe-preloader.php` and debug the error.

If you're sure this is an error by the package, [open an issue](https://github.com/DarkGhostHunter/Laraload/issues/new) with full details and stack trace. If it's a problem on the Preloader itself, [issue it there](https://github.com/DarkGhostHunter/Preloader/issues).

* **Why I can't use something like `php artisan laraload:generate` instead? Like a [Listener](https://laravel.com/docs/events) or [Scheduler](https://laravel.com/docs/scheduling)?**

Opcache is not enabled when using PHP CLI. You must let the live application generate the list automatically _on demand_.

* **Does this excludes the package itself from the list?**

It does not: since the underlying Preloader package may be not heavily requested, it doesn't matter if its excluded or not. The files in Laraload are also not excluded from the list, since these are needed to trigger the Preloader itself without hindering performance. 

* **I activated Laraload but my application still doesn't feel _fast_. What's wrong?**

Laraload creates a preloading script, but **doesn't load the script into Opcache**. Once the script is generated, you must include it in your `php.ini` - currently there is no other way to do it. This will take effect only at PHP process startup.

If you still _feel_ your app is slow, remember to benchmark your app, cache your config and views, check your database queries and API calls, and queue expensive logic, among other things.

* **How the list is created?**

Basically: the most hit files in descending order. Each file consumes memory, so the list is _soft-cut_ when the files reach a given memory limit (32MB by default).

* **You said "_soft-cut_", why is that?**

Each file is loaded using `opcache_compile_file()`. If the last file is a class with links outside the list, PHP will issue some warnings, which is normal and intended.

* **Can I just put all the files in my project?**

You shouldn't. Including all the files of your application may have diminishing returns compared to, for example, only the most requested. You can always benchmark your app yourself to prove this is wrong for your exclusive case.

* **Can I use a Closure for my condition?**

No, you must use your the default condition class or your own class, or use `Class@method` notation.

* **Can I deactivate the middleware? Or check only XXX status?**

Nope. If you are looking for total control, [use directly the Preloader package](https://github.com/DarkGhostHunter/Preloader/).

* **Does the middleware works on unit testing?**

Nope. The middleware is not registered if the application is running under Unit Testing environment.

* **How can I know when a Preload script is successfully generated?**

When the Preload script is called, you will receive a `PreloadCalledEvent` instance with the compilation status (`true` on success, `false` on failure). You can [add a Listener](https://laravel.com/docs/events#registering-events-and-listeners) to dispatch an email or a Slack notification.

If there is a bigger problem, your application logger will catch the exception.

* **Why now I need to use a callback to append/exclude files, instead of a simple array of files?**

This new version uses Preloader 2, which offers greater flexibility to handle files inside a directory. This approach is incompatible with just issuing directly an array of files, but is more convenient in the long term. Considering that appending and excluding files mostly requires pin-point precision, it was decided to leave it as method calls for this kind of flexibility.

## License

This package is licenced by the [MIT License](LICENSE).
