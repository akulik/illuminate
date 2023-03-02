Installation
============

First of all, create a new branch in your version control system to hold the project migration.
Do not merge this branch into current development one, until you ensure created "Two-headed Beast" works fine for you.


Project structure <span id="project-structure"></span>
-----------------

Move all your existing project files under the new directory called "legacy". You should move all PHP files and even a
"webroot" folder there. Only 'composer.json', 'vendor', docs and service files like '.gitignore' can be left at the
project root.

Next you will need to fill the project root with Laravel application.
You can do this manually taking files from [Laravel repository](https://github.com/laravel/laravel/).
You can also use Composer for this:

```
composer create-project --prefer-dist laravel/laravel my-project
```

In result your project root will look like following:

```
app/
bootstrap/
config/
database/
legacy/
public/
resurces/
routes/
storage/
tests/
.gitignore
artisan
composer.json
...
```

Now you can move public resources like images, CSS and JavaScript files from your former "webroot" folder to the
"public" directory. For example:

```
git mv ./legacy/web/img ./public/img
git mv ./legacy/web/css ./public/css
git mv ./legacy/web/js ./public/js
```

Make sure you create an "assets" folder in your "public" directory for Yii assets publishing.


Composer setup <span id="composer-setup"></span>
--------------

You will need to compose new 'composer.json' file, which will include both the requirements from your Yii project and
the ones [demanded by Laravel](https://github.com/laravel/laravel/blob/master/composer.json).
Also, obviously, you will need to add this package to requirements section as well.
Thus in the end your 'composer.json' file will look like following:

```json
{
    "name": "my/project",
    ...
    "require": {
        "php": "^8.0",
        "guzzlehttp/guzzle": "^7.2",
        "laravel/framework": "^10.0",
        "laravel/sanctum": "^3.2",
        "laravel/tinker": "^2.8",
        "yiisoft/yii2": "~2.0.14",
        "yiisoft/yii2-bootstrap5": "~2.0.0",
        "yiisoft/yii2-swiftmailer": "~2.0.0",
        "yii2tech/illuminate": "^1.3.0",
        ...
    },
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Database\\Factories\\": "database/factories/",
            "Database\\Seeders\\": "database/seeders/",
            "legacy\\": "legacy/"
        }
    },
    "repositories": [
        {
            "type": "composer",
            "url": "https://asset-packagist.org"
        }
    ],
    ...
}
```

> Tip: In case you do not want to install assets via Composer in Yii way, dealing with 'asset-packagist.org' or composer
  asset plugin, you can list asset packages at "replace" section of your 'composer.json'. For example:

```json
{
    ...
    "replace": {
        "bower-asset/jquery": ">=1.11.0",
        "bower-asset/inputmask": ">=3.2.0",
        "bower-asset/punycode": ">=1.3.0",
        "bower-asset/yii2-pjax": ">=2.0.0"
    },
    ...
}
```


Legacy Namespace renaming <span id="legacy-namespace-renaming"></span>
-------------------------

Most likely you were using "app" as a root namespace at your Yii project. Laravel uses "App" for the same purpose.
While technically those two are different, it is better to make them more contrast, especially if you run your project
on OS with case-insensitive filesystem, like Windows. It is recommended to rename "app" namespace of the old Yii project to
something different like "legacy", e.g. full class name like `app\models\User` will become `legacy\models\User`.
Such naming will serve you well in the future: whenever you see "legacy" in class name indicates the part of the code,
which need to be rewritten. You can perform namespace renaming manually or using your preferable tool like IDE refactor helper.
However, you can use console command provided by this extension for the same purpose:

```
php artisan namespace:rename --from=app --to=legacy ./legacy
```

Make sure you have setup correct namespace binding for class autoload in you 'composer.json':

```json
{
    ...
    "autoload": {
        "psr-4": {
            ...
            "legacy\\": "legacy/"
        },
        ...
    },
    ...
}
```


Yii Application Middleware <span id="yii-application-middleware"></span>
--------------------------

Yii application will run inside Laravel one as a middleware.
`\Yii2tech\Illuminate\Http\YiiApplicationMiddleware` will serve as such middleware.
You should add it to your Laravel HTTP kernel class, for example:

```php
<?php

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    protected $middleware = [
        \App\Http\Middleware\CheckForMaintenanceMode::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
        // ...
        \Yii2tech\Illuminate\Http\YiiApplicationMiddleware::class, // This will run Yii application
    ];
    // ...
}
```

As an alternative, you can create a fallback Laravel route, which triggers 404 HTTP error, with this middleware attached.
For example:

```php
<?php

use Illuminate\Support\Facades\Route;

// ...

Route::any('{fallbackPlaceholder}', function () {
    abort(404);
})
    ->middleware(Yii2tech\Illuminate\Http\YiiApplicationMiddleware::class)
    ->where('fallbackPlaceholder', '.*')
    ->fallback();
```

This middleware will run Yii application from its originally entry script, which is now, most likely,
located at 'legacy/web/index.php'. The common content of such file is following:

```php
<?php

require(__DIR__ . '/../vendor/autoload.php');
require(__DIR__ . '/../vendor/yiisoft/yii2/Yii.php');

$config = require(__DIR__ . '/../config/web.php');

(new yii\web\Application($config))->run();
```

You do not need to change any logic inside of it, however, requirements of "vendor/autoload.php" and
"Yii.php" are redundant here and will cause an error, if they will remain. So adjusted entry script
should look like following:

```php
<?php

$config = require(__DIR__ . '/../config/web.php');

(new yii\web\Application($config))->run();
```

Path "legacy/web/index.php" is used by `\Yii2tech\Illuminate\Http\YiiApplicationMiddleware` by default. In case
you need to change it, you should specify your own value as a middleware argument. For example:

```php
<?php

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    protected $middleware = [
        // ...
        \Yii2tech\Illuminate\Http\YiiApplicationMiddleware::class.':legacy/frontend/public/index.php',
    ];
    // ...
}
```

You may adjust middleware settings using laravel configuration key "yii.middleware". Configuration file for can be created
using following console command:

```
php artisan vendor:publish --provider="Yii2tech\Illuminate\YiiIlluminateServiceProvider" --tag=config
```

This will create file "config/yii.php" will following content:

```php
<?php

return [
    'middleware' => [
        'defaultEntryScript' => 'legacy/web/index.php',
        // ...
    ],
];
```

> Note: if you need to use custom `Yii` class, you should specify path to it via Composer "classmap".

**Heads up!** Make sure you do not specify routes, which catch all HTTP requests in your Yii URL Manager.
Middleware will pass request resolving to Laravel only, if Yii application ends up with 404 `HttpException`.

`\Yii2tech\Illuminate\Http\YiiApplicationMiddleware` automatically defines `YII_DEBUG` and `YII_ENV` constants from
corresponding Laravel configuration, thus setting them at the Yii entry script or config file will cause no effect or an error.
You can use `\Yii2tech\Illuminate\Http\YiiApplicationMiddleware::$bootstrap` in case you need to define Yii constants manually.


Yii Application Configuration <span id="yii-application-configuration"></span>
-----------------------------

There are several options, which have to be adjusted at Yii application configuration array.
These options are:

- `\yii\base\Application::$vendorPath` - path to Composer 'vendor' directory.
- path alias(es) to Yii application root(s), e.g. "@app", "@frontend" and so on.
- `\yii\base\Application::$controllerNamespace` - the namespace, which controller classes are located in.

Thus configuration array for Yii application should look like following:

```php
<?php
// file 'legacy/config/web.php'

return [
    'vendorPath' => realpath(__DIR__.'/../../vendor'),
    'aliases' => [
        '@app' => dirname(__DIR__),
        '@legacy' => dirname(__DIR__),
        // ...
    ],
    'controllerNamespace' => 'legacy\controllers',
    // ...
];
```

You may also need to replace standard Yii components with the ones provided by this extension, creating
proper bridges between Yii and Laravel. Those components are described at other sections of this documentation.


Verifying Results <span id="verifying-results"></span>
-----------------

Once all steps listed above are done, you should receive working HTTP application, where all your former
URLs are responding as they were. E.g. if you type "https://my-project.test/login" in browser, you should
be able to see your login form as it was before.
To check up Laravel URL handling is working you can create a test route at "routes/web.php", like following:

```php
<?php

use Illuminate\Support\Facades\Route;

Route::get('test-laravel', function () {
    return view('welcome');
});
```

Now, if you type "https://my-project.test/test-laravel" in browser, you should be able to see Laravel welcome page.


Resolving several Yii applications <span id="resolving-several-yii-applications"></span>
----------------------------------

In case your original Yii project has several web application entry points, serving different subdomains, like [Yii 2.0 Advanced Application Template](https://github.com/yiisoft/yii2-app-advanced)
does, you will need to register multiple route fallbacks with `\Yii2tech\Illuminate\Http\YiiApplicationMiddleware` attached.
One - per each sub-domain you use. See [Laravel subdomain routing](https://laravel.com/docs/routing#route-group-subdomain-routing) for more details.
The result Laravel routes may look like following:

```php
<?php

use Illuminate\Support\Facades\Route;

//...

Route::domain('frontend.my-project.test')->any('{fallbackPlaceholder}', function () {
    abort(404);
})
    ->middleware(Yii2tech\Illuminate\Http\YiiApplicationMiddleware::class.':legacy/frontend/web/index.php')
    ->where('fallbackPlaceholder', '.*')
    ->fallback();

Route::domain('backend.my-project.test')->any('{fallbackPlaceholder}', function () {
    abort(404);
})
    ->middleware(Yii2tech\Illuminate\Http\YiiApplicationMiddleware::class.':legacy/backend/web/index.php')
    ->where('fallbackPlaceholder', '.*')
    ->fallback();

Route::domain('api.my-project.test')->any('{fallbackPlaceholder}', function () {
    abort(404);
})
    ->middleware(Yii2tech\Illuminate\Http\YiiApplicationMiddleware::class.':legacy/api/web/index.php')
    ->where('fallbackPlaceholder', '.*')
    ->fallback();
```
