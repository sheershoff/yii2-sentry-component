Yii2 sentry component
=====================

Yii2 Sentry component allowing for unified array format passing parameters to Sentry log target and other log targets. The ability to pass arrays as the log message is suitable practice for
those who want to use Logstash, ElasticSearch, etc. for getting statistical inferences of these data later on.

## Install
```
php composer.phar require e96/yii2-sentry:dev-master
```

In config file:

```php
'bootstrap' => ['log', 'raven'],
'components' => [
    'raven' => [
        'class' => 'sheershoff\sentry\ErrorHandler',
        'dsn' => '', // Sentry DSN
    ],
    'log' => [
        'targets' => [
            [
                'class' => 'sheershoff\sentry\Target',
                'levels' => ['error', 'warning'],
                'dsn' => '', // Sentry DSN
            ]
        ],
    ],
]
```

## Usage

Exceptions and PHP errors are caught without effort. Standart `Yii::(error|warning|info|trace)` logging works as usual, but you also can use the following format:

```php
Yii::warning([
    'msg' => 'SomeWarning', // event name that will be sent to Sentry
    'data' => [ // extra data for the event
        'userId' => Yii::$app->user->id,
        'someDataOnWarningSituation' => $controller->state,
        'modelThatCausedFailure' => $model->attributes,
    ],
], 'eventCategory');
```

Or you can replace this with `Log::warning` as in the exception example below, since the exception argument is not required. Notice that `eventCategory` is not sent to Sentry and is used only for log messages routing and filtering.

Wherever you need to log a caught exception with stacktrace and additional data, use

```php
use sheershoff\sentry\Log;
// some code here
try{
    $model1->save();
}catch (\Exception $e){
    Log::warning([
        'msg' => 'ExceptionWarning', // event name that will be sent to Sentry
        'data' => [ // extra data for the event
            'userId' => Yii::$app->user->id,
            'someDataOnWarningSituation' => $controller->state,
            'modelThatCausedFailure' => $model->attributes,
        ],
    ], 'exceptionCategory', $e);
}
```

There are proxy methods in `Log` for the four Yii static methods: `error`, `warning`, `info`, `trace`. If `$e` is not null the component expects that it is an exception and after calling the
corresponding Yii method also captures the exception for Sentry.

Also, the following use case is also possible:

```php
/** @var ErrorHandler $raven */
$raven = \Yii::$app->get('raven');
$raven->client->extra_context($task->attributes);

throw new Exception('unknown task type');
```

## Other log targets

To use the power of the component you should keep in mind that other log targets will receive arrays instead of strings in the log message. You can create a proxy class, redefining `formatMessage` method of the parent LogTarget, e.g.:

```php
namespace common\components;

use Yii;

class SyslogJsonTarget extends \yii\log\SyslogTarget
{
	/**
	 * @inheritdoc
	 */
	public function formatMessage($message)
	{
		list($text, $level, $category, $timestamp) = $message;
		$level = \yii\log\Logger::getLevelName($level);
		if (!is_string($text)) {
			$text = \yii\helpers\Json::encode($text);
		} else {
			$text = \yii\helpers\Json::encode(['rawstring' => $text]);
		}
		$prefix = $this->getMessagePrefix($message);
		return "{$prefix}[$level][$category] $text";
	}
}
```

## Inspired by

*  <https://github.com/E96/yii2-sentry>
*  <https://github.com/DarkAiR/yii2-sentry-log>