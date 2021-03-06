Modules
=======

Modules are self-contained software units that consist of [models](structure-models.md), [views](structure-views.md),
[controllers](structure-controllers.md), and other supporting components. End users can access the controllers
of a module when it is installed in [application](structure-applications.md). For these reasons, modules are
often viewed as mini-applications. Modules differ from [applications](structure-applications.md) in that
modules cannot be deployed alone and must reside within applications.


## Creating Modules <a name="creating-modules"></a>

A module is organized as a directory which is called the [[yii\base\Module::basePath|base path]] of the module.
Within the directory, there are sub-directories, such as `controllers`, `models`, `views`, which hold controllers,
models, views, and other code, just like in an application. The following example shows the content within a module:

```
forum/
    Module.php                   the module class file
    controllers/                 containing controller class files
        DefaultController.php    the default controller class file
    models/                      containing model class files
    views/                       containing controller view and layout files
        layouts/                 containing layout view files
        default/                 containing view files for DefaultController
            index.php            the index view file
```


### Module Classes <a name="module-classes"></a>

Each module should have a module class which extends from [[yii\base\Module]]. The class should be located
directly under the module's [[yii\base\Module::basePath|base path]] and should be [autoloadable](concept-autoloading.md).
When a module is being accessed, a single instance of the corresponding module class will be created.
Like [application instances](structure-applications.md), module instances are used to share data and components
for code within modules.

The following is an example how a module class may look like:

```php
namespace app\modules\forum;

class Module extends \yii\base\Module
{
    public function init()
    {
        parent::init();

        $this->params['foo'] = 'bar';
        // ...  other initialization code ...
    }
}
```

If the `init()` method contains a lot of code initializing the module's properties, you may also save them in terms
of a [configuration](concept-configurations.md) and load it with the following code in `init()`:

```php
public function init()
{
    parent::init();
    // initialize the module with the configuration loaded from config.php
    \Yii::configure($this, require(__DIR__ . '/config.php'));
}
```

where the configuration file `config.php` may contain the following content, similar to that in an
[application configuration](structure-applications.md#application-configurations).

```php
<?php
return [
    'components' => [
        // list of component configurations
    ],
    'params' => [
        // list of parameters
    ],
];
```


### Controllers in Modules <a name="controllers-in-modules"></a>

When creating controllers in a module, a convention is to put the controller classes under the `controllers`
sub-namespace of the namespace of the module class. This also means the controller class files should be
put in the `controllers` directory within the module's [[yii\base\Module::basePath|base path]].
For example, to create a `post` controller in the `forum` module shown in the last subsection, you should
declare the controller class like the following:

```php
namespace app\modules\forum\controllers;

use yii\web\Controller;

class PostController extends Controller
{
    // ...
}
```

You may customize the namespace of controller classes by configuring the [[yii\base\Module::controllerNamespace]]
property. In case when some of the controllers are out of this namespace, you may make them accessible
by configuring the [[yii\base\Module::controllerMap]] property, similar to [what you do in an application](structure-applications.md#controller-map).


### Views in Modules <a name="views-in-modules"></a>

Views in a module should be put in the `views` directory within the module's [[yii\base\Module::basePath|base path]].
For views rendered by a controller in the module, they should be put under the directory `views/ControllerID`,
where `ControllerID` refers to the [controller ID](structure-controllers.md#routes). For example, if
the controller class is `PostController`, the directory would be `views/post` within the module's
[[yii\base\Module::basePath|base path]].

A module can specify a [layout](structure-views.md#layouts) that is applied to the views rendered by the module's
controllers. The layout should be put in the `views/layouts` directory by default, and you should configure
the [[yii\base\Module::layout]] property to point to the layout name. If you do not configure the `layout` property,
the application's layout will be used instead.


## Using Modules <a name="using-modules"></a>

To use a module in an application, simply configure the application by listing the module in
the [[yii\base\Application::modules|modules]] property of the application. The following code in the
[application configuration](structure-applications.md#application-configurations) uses the `forum` module:

```php
[
    'modules' => [
        'forum' => [
            'class' => 'app\modules\forum\Module',
            // ... other configurations for the module ...
        ],
    ],
]
```

The [[yii\base\Application::modules|modules]] property takes an array of module configurations. Each array key
represents a *module ID* which uniquely identifies the module among all modules in the application, and the corresponding
array value is a [configuration](concept-configurations.md) for creating the module.


### Routes <a name="routes"></a>

Like accessing controllers in an application, [routes](structure-controllers.md#routes) are used to address
controllers in a module. A route for a controller within a module must begin with the module ID followed by
the controller ID and action ID. For example, if an application uses a module named `forum`, then the route
`forum/post/index` would represent the `index` action of the `post` controller in the module. If the route
only contains the module ID, then the [[yii\base\Module::defaultRoute]] property, which defaults to `default`,
will determine which controller/action should be used. This means a route `forum` would represent the `default`
controller in the `forum` module.


### Accessing Modules <a name="accessing-modules"></a>

Within a module, you may often need to get the instance of the [module class](#module-classes) so that you can
access the module ID, module parameters, module components, etc. You can do so by using the following statement:

```php
$module = MyModuleClass::getInstance();
```

where `MyModuleClass` refers to the name of the module class that you are interested in. The `getInstance()` method
will return the currently requested instance of the module class. If the module is not requested, the method will
return null. Note that You do not want to manually create a new instance of the module class because it will be
different from the one created by Yii in response to a request.

> Info: When developing a module, you should not assume the module will use a fixed ID. This is because a module
  can be associated with an arbitrary ID when used in an application or within another module. In order to get
  the module ID, you should use the above approach to get the module instance first, and then get the ID via
  `$module->id`.

You may also access the instance of a module using the following approaches:

```php
// get the module whose ID is "forum"
$module = \Yii::$app->getModule('forum');

// get the module to which the currently requested controller belongs
$module = \Yii::$app->controller->module;
```

The first approach is only useful when you know the module ID, while the second approach is best used when you
know about the controllers being requested.

Once getting hold of a module instance, you can access parameters or components registered with the module. For example,

```php
$maxPostCount = $module->params['maxPostCount'];
```


### Bootstrapping Modules <a name="bootstrapping-modules"></a>

Some modules may need to be run for every request. The [[yii\debug\Module|debug]] module is such an example.
To do so, list the IDs of such modules in the [[yii\base\Application::bootstrap|bootstrap]] property of the application.

For example, the following application configuration makes sure the `debug` module is always load:

```php
[
    'bootstrap' => [
        'debug',
    ],

    'modules' => [
        'debug' => 'yii\debug\Module',
    ],
]
```


## Nested Modules <a name="nested-modules"></a>

Modules can be nested in unlimited levels. That is, a module can contain another module which can contain yet
another module. We call the former *parent module* while the latter *child module*. Child modules must be declared
in the [[yii\bas\Module::modules|modules]] property of their parent modules. For example,

```php
namespace app\modules\forum;

class Module extends \yii\base\Module
{
    public function init()
    {
        parent::init();

        $this->modules = [
            'admin' => [
                // you should consider using a shorter namespace here!
                'class' => 'app\modules\forum\modules\admin\Module',
            ],
        ];
    }
}
```

For a controller within a nested module, its route should include the IDs of all its ancestor module.
For example, the route `forum/admin/dashboard/index` represents the `index` action of the `dashboard` controller
in the `admin` module which is a child module of the `forum` module.


## Best Practices <a name="best-practices"></a>

Modules are best used in large applications whose features can be divided into several groups, each consisting of
a set of closely related features. Each such feature group can be developed as a module which is developed and
maintained by a specific developer or team.

Modules are also a good way of reusing code at the feature group level. Some commonly used features, such as
user management, comment management, can all be developed in terms of modules so that they can be reused easily
in future projects.
