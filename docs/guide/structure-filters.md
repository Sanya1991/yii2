Filters
=======

Filters are objects that run before and/or after [controller actions](structure-controllers.md#actions). For example,
an access control filter may run before actions to ensure that they are allowed to be accessed by particular end users;
a content compression filter may run after actions to compress the response content before sending them out to end users.

A filter may consist of a pre-filter (filtering logic applied *before* actions) and/or a post-filter (logic applied
*after* actions).


## Using Filters <a name="using-filters"></a>

Filters are essentially a special kind of [behaviors](concept-behaviors.md). Therefore, using filters is the same
as [using behaviors](concept-behaviors.md#attaching-behaviors). You can declare filters in a controller class
by overriding its [[yii\base\Controller::behaviors()|behaviors()]] method like the following:

```php
public function behaviors()
{
    return [
        [
            'class' => 'yii\filters\HttpCache',
            'only' => ['index', 'view'],
            'lastModified' => function ($action, $params) {
                $q = new \yii\db\Query();
                return $q->from('user')->max('updated_at');
            },
        ],
    ];
}
```

By default, filters declared in a controller class will be applied to *all* actions in that controller. You can,
however, explicitly specify which actions the filter should be applied to by configuring the
[[yii\base\ActionFilter::only|only]] property. In the above example, the `HttpCache` filter only applies to the
`index` and `view` actions. You can also configure the [[yii\base\ActionFilter::except|except]] property to blacklist
some actions from being filtered.

Besides controllers, you can also declare filters in a [module](structure-modules.md) or [application](structure-applications.md).
When you do so, the filters will be applied to *all* controller actions belonging to that module or application,
unless you configure the filters' [[yii\base\ActionFilter::only|only]] and [[yii\base\ActionFilter::except|except]]
properties like described above.

> Note: When declaring filters in modules or applications, you should use [routes](structure-controllers.md#routes)
  instead of action IDs in the [[yii\base\ActionFilter::only|only]] and [[yii\base\ActionFilter::except|except]] properties.
  This is because action IDs alone cannot fully specify actions within the scope of a module or application.

When multiple filters are configured for a single action, they are applied according to the rules described below,

* Pre-filtering
    - Apply filters declared in the application in the order they are listed in `behaviors()`.
    - Apply filters declared in the module in the order they are listed in `behaviors()`.
    - Apply filters declared in the controller in the order they are listed in `behaviors()`.
    - If any of the filters cancel the action execution, the filters (both pre-filters and post-filters) after it will
      not be applied.
* Running the action if it passes the pre-filtering.
* Post-filtering
    - Apply filters declared in the controller in the reverse order they are listed in `behaviors()`.
    - Apply filters declared in the module in the reverse order they are listed in `behaviors()`.
    - Apply filters declared in the application in the reverse order they are listed in `behaviors()`.


## Creating Filters <a name="creating-filters"></a>

To create a new action filter, extend from [[yii\base\ActionFilter]] and override the
[[yii\base\ActionFilter::beforeAction()|beforeAction()]] and/or [[yii\base\ActionFilter::afterAction()|afterAction()]]
methods. The former will be executed before an action runs while the latter after an action runs.
The return value of [[yii\base\ActionFilter::beforeAction()|beforeAction()]] determines whether an action should
be executed or not. If it is false, the filters after this one will be skipped and the action will not be executed.

The following example shows a filter that logs the action execution time:

```php
namespace app\components;

use Yii;
use yii\base\ActionFilter;

class ActionTimeFilter extends ActionFilter
{
    private $_startTime;

    public function beforeAction($action)
    {
        $this->_startTime = microtime(true);
        return parent::beforeAction($action);
    }

    public function afterAction($action, $result)
    {
        $time = microtime(true) - $this->_startTime;
        Yii::trace("Action '{$action->uniqueId}' spent $time second.");
        return parent::afterAction($action, $result);
    }
}
```


## Core Filters <a name="core-filters"></a>

Yii provides a set of commonly used filters, found primarily under the `yii\filters` namespace. In the following,
we will briefly introduce these filters.


### [[yii\filters\AccessControl|AccessControl]] <a name="access-control"></a>

AccessControl provides simple access control based on a set of [[yii\filters\AccessControl::rules|rules]].
In particular, before an action is executed, AccessControl will examine the listed rules and find the first one
that matches the current context variables (such as user IP address, user login status, etc.) The matching
rule will dictate whether to allow or deny the execution of the requested action. If no rule matches, the access
will be denied.

The following example shows how to allow authenticated users to access the `create` and `update` actions
while denying all other users from accessing these two actions.

```php
use yii\filters\AccessControl;

public function behaviors()
{
    return [
        'access' => [
            'class' => AccessControl::className(),
            'only' => ['create', 'update'],
            'rules' => [
                // allow authenticated users
                [
                    'allow' => true,
                    'roles' => ['@'],
                ],
                // everything else is denied by default
            ],
        ],
    ];
}
```

For more details about access control in general, please refer to the [Authorization](security-authorization.md) section.


### [[yii\filters\ContentNegotiator|ContentNegotiator]] <a name="content-negotiator"></a>

ContentNegotiator supports response format negotiation and application language negotiation. It will try to
determine the response format and/or language by examining `GET` parameters and `Accept` HTTP header.

In the following example, ContentNegotiator is configured to support JSON and XML response formats, and
English (United States) and German languages.

```php
use yii\filters\ContentNegotiator;
use yii\web\Response;

public function behaviors()
{
    return [
        [
            'class' => ContentNegotiator::className(),
            'formats' => [
                'application/json' => Response::FORMAT_JSON,
                'application/xml' => Response::FORMAT_XML,
            ],
            'languages' => [
                'en-US',
                'de',
            ],
        ],
    ];
}
```

Response formats and languages often need to be determined much earlier during
the [application lifecycle](structure-applications.md#application-lifecycle). For this reason, ContentNegotiator
is designed in a way such that it can also be used as a [bootstrap component](structure-applications.md#bootstrap)
besides filter. For example, you may configure it in the [application configuration](structure-applications.md#application-configurations)
like the following:

```php
use yii\filters\ContentNegotiator;
use yii\web\Response;

[
    'bootstrap' => [
        [
            'class' => ContentNegotiator::className(),
            'formats' => [
                'application/json' => Response::FORMAT_JSON,
                'application/xml' => Response::FORMAT_XML,
            ],
            'languages' => [
                'en-US',
                'de',
            ],
        ],
    ],
];
```


### [[yii\filters\HttpCache|HttpCache]] <a name="http-cache"></a>

HttpCache implements client-side caching by utilizing the `Last-Modified` and `Etag` HTTP headers.
For example,

```php
use yii\filters\HttpCache;

public function behaviors()
{
    return [
        [
            'class' => HttpCache::className(),
            'only' => ['index'],
            'lastModified' => function ($action, $params) {
                $q = new \yii\db\Query();
                return $q->from('user')->max('updated_at');
            },
        ],
    ];
}
```

Please refer to the [HTTP Caching](caching-http.md) section for more details about using HttpCache.


### [[yii\filters\PageCache|PageCache]] <a name="page-cache"></a>

PageCache implements server-side caching of whole pages. In the following example, PageCache is applied
to the `index` action to cache the whole page for maximum 60 seconds or until the count of entries in the `post`
table changes. It also stores different versions of the page depending on the chosen application language.

```php
use yii\filters\PageCache;
use yii\caching\DbDependency;

public function behaviors()
{
    return [
        'pageCache' => [
            'class' => PageCache::className(),
            'only' => ['index'],
            'duration' => 60,
            'dependency' => [
                'class' => DbDependency::className(),
                'sql' => 'SELECT COUNT(*) FROM post',
            ],
            'variations' => [
                \Yii::$app->language,
            ]
        ],
    ];
}
```

Please refer to the [Page Caching](caching-page.md) section for more details about using PageCache.


### [[yii\filters\RateLimiter|RateLimiter]] <a name="rate-limiter"></a>

RateLimiter implements a rate limiting algorithm based on the [leaky bucket algorithm](http://en.wikipedia.org/wiki/Leaky_bucket).
It is primarily used in implementing RESTful APIs. Please refer to the [Rate Limiting](rest-rate-limiting.md) section
for details about using this filter.


### [[yii\filters\VerbFilter|VerbFilter]] <a name="verb-filter"></a>

VerbFilter checks if the HTTP request methods are allowed by the requested actions. If not allowed, it will
throw an HTTP 405 exception. In the following example, VerbFilter is declared to specify a typical set of allowed
request methods for CRUD actions.

```php
use yii\filters\VerbFilter;

public function behaviors()
{
    return [
        'verbs' => [
            'class' => VerbFilter::className(),
            'actions' => [
                'index'  => ['get'],
                'view'   => ['get'],
                'create' => ['get', 'post'],
                'update' => ['get', 'put', 'post'],
                'delete' => ['post', 'delete'],
            ],
        ],
    ];
}
```
