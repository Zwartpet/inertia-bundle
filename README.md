# Inertia.js Symfony Adapter
[![CI](https://github.com/rompetomp/inertia-bundle/workflows/CI/badge.svg)](https://github.com/rompetomp/inertia-bundle/actions)
[![StyleCI](https://github.styleci.io/repos/201484253/shield?style=flat)](https://github.styleci.io/repos/201484253)

This is a Inertia.js server-side adapter based on [inertia-laravel](https://github.com/inertiajs/inertia-laravel), but
for Symfony 5 and 6.

## Installation
First, make sure you have the twig, encore and serializer recipe:
```console
composer require twig encore symfony/serializer-pack
```

Install using Composer:
```console
composer require rompetomp/inertia-bundle
```
```console
yarn add @inertiajs/inertia
```

## Setup root template
The first step to using Inertia is creating a root template. We recommend using `app.html.twig`. This template should
include your assets, as well as the `inertia(page)` function

```twig
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Welcome!{% endblock %}</title>
    {% block stylesheets %}
        {{ encore_entry_link_tags('app') }}
    {% endblock %}
</head>
<body>
{{ inertia(page) }}
{% block javascripts %}
    {{ encore_entry_script_tags('app') }}
{% endblock %}
</body>
</html>
```

The `inertia(page)` function is a helper function for creating our base `div`. It includes a `data-page` attribute which
contains the initial page information. This is what it looks like:
```php
<div id="app" data-page="<?php echo htmlspecialchars(json_encode($page)); ?>"></div>
```

If you'd like a different root view, you can change it by creating a `config/packages/rompetomp_inertia.yaml` file
and including this config:
```yaml
rompetomp_inertia:
  root_view: 'name.twig.html'
```

## Set up the frontend adapter
Find a frontend adapter that you wish to use here https://github.com/inertiajs. The README's are using Laravel's Webpack
Mix. It's not hard translating this to Webpack Encore, just follow the documentation here: https://symfony.com/doc/current/frontend.html.

### Webpack Encore Examples
For Vue:
```console
yarn add @inertiajs/inertia-vue
```

```javascript
const Encore = require('@symfony/webpack-encore')
const path = require('path')

if (!Encore.isRuntimeEnvironmentConfigured()) {
  Encore.configureRuntimeEnvironment(process.env.NODE_ENV || 'dev')
}

Encore
  .setOutputPath('public/build/')
  .setPublicPath('/build')
  .enableVueLoader()
  .addAliases({
    vue$: 'vue/dist/vue.runtime.esm.js',
    '@': path.resolve('assets/js')
  })
  .addEntry('app', './assets/js/app.js')
  .splitEntryChunks()
  .cleanupOutputBeforeBuild()
  .enableSourceMaps(!Encore.isProduction())
  .enableVersioning(Encore.isProduction())
  .disableSingleRuntimeChunk()
  .configureBabel(() => {}, {
    useBuiltIns: 'usage',
    corejs: 3
  })
  .enableSassLoader()

module.exports = Encore.getWebpackConfig()
```

```javascript
//assets/app.js
import { createInertiaApp } from '@inertiajs/inertia-vue'
import Vue from "vue";

createInertiaApp({
    resolve: name => require(`./Pages/${name}`),
    setup({ el, app, props }) {
        new Vue({
            render: h => h(app, props),
        }).$mount(el)
    },
})
```

For React:
```javascript
const Encore = require('@symfony/webpack-encore')
const path = require('path')

if (!Encore.isRuntimeEnvironmentConfigured()) {
  Encore.configureRuntimeEnvironment(process.env.NODE_ENV || 'dev')
}

Encore
  .setOutputPath('public/build/'),
  .setPublicPath('/build')
  .enableReactPreset()
  .addAliases({
    '@': path.resolve('assets/js')
  })
  .addEntry('app', './assets/js/app.js')
  .splitEntryChunks()
  .cleanupOutputBeforeBuild()
  .enableSourceMaps(!Encore.isProduction())
  .enableVersioning(Encore.isProduction())
  .disableSingleRuntimeChunk()
  .configureBabel(() => {}, {
    useBuiltIns: 'usage',
    corejs: 3
  })
  .enableSassLoader()

module.exports = Encore.getWebpackConfig()
```

For Svelte:
```javascript
const Encore = require('@symfony/webpack-encore')
const path = require('path')

if (!Encore.isRuntimeEnvironmentConfigured()) {
  Encore.configureRuntimeEnvironment(process.env.NODE_ENV || 'dev')
}

Encore
  .setOutputPath('public/build/')
  .setPublicPath('/build')
  .addLoader({
    test: /\.(svelte)$/,
    use: {
      loader: 'svelte-loader',
      options: {
        emitCss: true,
        hotReload: true,
      },
    },
  })
  .addAliases({
    '@': path.resolve('assets/js')
  })
  .addEntry('app', './assets/js/app.js')
  .splitEntryChunks()
  .cleanupOutputBeforeBuild()
  .enableSourceMaps(!Encore.isProduction())
  .enableVersioning(Encore.isProduction())
  .disableSingleRuntimeChunk()
  .configureBabel(() => {}, {
    useBuiltIns: 'usage',
    corejs: 3
  })
  .enableSassLoader()

const config = Encore.getWebpackConfig()
config.resolve.mainFields = ['svelte', 'browser', 'module', 'main']
config.resolve.extensions =  ['.wasm', '.mjs', '.js', '.json', '.jsx', '.vue', '.ts', '.tsx', '.svelte']

module.exports = config
```

## Making Inertia responses
To make an Inertia response, inject the `Rompetomp\InertiaBundle\Service\InertiaInterface $inertia` typehint in your 
controller, and use the render function on that Service:
```php
<?php
namespace App\Controller;

use Rompetomp\InertiaBundle\Service\InertiaInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

class DashboardController extends AbstractController
{
    public function index(InertiaInterface $inertia)
    {
        return $inertia->render('Dashboard', ['prop' => 'propValue']);
    }
}
```

## Sharing data
To share data with all your components, use `$inertia->share($key, $data)`. This can be done in an EventSubscriber:
```php
<?php

namespace App\EventSubscriber;

use Rompetomp\InertiaBundle\Service\InertiaInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\KernelEvents;

class InertiaSubscriber implements EventSubscriberInterface
{
    /** @var \Rompetomp\InertiaBundle\Service\InertiaInterface */
    protected $inertia;

    /**
     * AppSubscriber constructor.
     *
     * @param \Rompetomp\InertiaBundle\Service\InertiaInterface $inertia
     */
    public function __construct(InertiaInterface $inertia)
    {
        $this->inertia = $inertia;
    }

    public static function getSubscribedEvents()
    {
        return [
            KernelEvents::CONTROLLER => 'onControllerEvent',
        ];
    }

    public function onControllerEvent($event)
    {
        $this->inertia->share(
            'Auth::user', 
            [
                'name' => 'Hannes', // Synchronously
                'posts' => function () {
                    return [1 => 'Post'];
                }   
            ]
        );
    }
}
```

## View data
If you want to pass data to your root template, you can do that by passing a third parameter to the render function:
```php
return $inertia->render('Dashboard', ['prop' => 'propValue'], ['title' => 'Page Title']);
```

You can also pass these with the function `viewData`, just like you would pass data to the `share` function:
```php
$this->inertia->viewData('title', 'Page Title');
```

You can access this data in your layout file under the `viewData` variable.

## Asset versioning
Like in Laravel, you can also pass a version to the Inertia services by calling
```php
$inertia->version($version);
```

## Projects using this bundle
- [Ping CRM on Symfony](https://github.com/aleksblendwerk/pingcrm-symfony) - The official Inertia.js demo app, ported to Symfony
