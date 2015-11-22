Pingpp Extension for Yii2
=================

基于 Ping++ 官方的 SDK 进行了简单的封装，用于 Yii2 框架。

Installation
--------------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
composer require --prefer-dist idarex/pingpp-yii2
```

or add

```json
"idarex/pingpp-yii2": "dev-master"
```

to the `require` section of your composer.json.


Configuration
--------------------

To use this extension, simply add the following code in your application configuration:

```php
return [
    //....
    'components' => [
        'pingpp' => [
            'class' => '\idarex\pingppyii2\PingppComponent',
            'apiKey' => '<YOUR_API_KEY>',
            'appId' => '<YOUR_APP_ID>',
        ],
    ],
];
```

使用
--------------------

#### 发起支付请求

```php
use Yii;
use yii\web\ServerErrorHttpException;
use idarex\pingppyii2\Channel;
use idarex\pingppyii2\ChargeForm;

$chargeForm = new ChargeForm();
$chargeForm->order_no = '123456789';
$chargeForm->amount = '100';
/**
 * @see Channel
 */
$chargeForm->channel = Channel::WX;
$chargeForm->currency = 'cny';
$chargeForm->client_ip = Yii::$app->getRequest()->userIP;
$chargeForm->subject = 'Your Subject';
$chargeForm->body = 'Your body';

if ($response = $chargeForm->create()) {
    return $response->__toArray(true);
} elseif ($chargeForm->hasErrors()) {
    var_dump($chargeForm->getErrors());
} else {
    throw new ServerErrorHttpException();
}
```

#### 接收 Webhooks 通知

##### Configuration

Modify your controler, add or change methode `actions()`

```php

/**
 * @inheritdoc
 */
public function beforeAction($action)
{
    if ($action->id == 'pingpp-hooks') {
        // 当用户完成交易后 Ping++ 会以 POST 方式把数据发送到你的 hook 地址
        // 所以这时候需要临时关闭掉 Yii 的 CSRF 验证
        Yii::$app->controller->enableCsrfValidation = false;
    }

    return parent::beforeAction($action);
}

public function actions()
{
	return [
        // ...
        'pingpp-hooks' => [
            'class' => '\idarex\pingppyii2\HooksAction',
            'pingppHooksComponentClass' => 'common\components\PingppHooks',
            'publicKeyPath' => '@common/config/pingpp_rsa_public_key.pem',
        ],
    ];
}
```

##### 写自己的 Webhook 业务

\#file: common/components/PingppHooks.php

* 使用 `$this->event` 来访问 Ping++ 提交过来的数据
* 用 `Yii::$app->getResponse()->data = '';` 来给返回值赋值。
* 方法最后调用 `Yii::$app->end();` 来结束请求。

```php
<?php

namespace common\components;

use idarex\pingppyii2\Hooks;
use idarex\pingppyii2\HooksInterface;
use Yii;

class PingppHooks extends Hooks implements HooksInterface
{
    /**
     * @inheritdoc
     */
    public function onAvailableDailySummary()
    {
        Yii::$app->end();
    }

    /**
     * @inheritdoc
     */
    public function onAvailableWeeklySummary()
    {
        Yii::$app->end();
    }

    /**
     * @inheritdoc
     */
    public function onAvailableMonthlySummary()
    {
        Yii::$app->end();
    }

    /**
     * @inheritdoc
     */
    public function onSucceededCharge()
    {
        $orderId = $this->event->data->object->order_no;
        Yii::$app->getResponse()->data = 'finished job';
        Yii::$app->end();
    }

    /**
     * @inheritdoc
     */
    public function onSucceededRefund()
    {
        Yii::$app->end();
    }

    /**
     * @inheritdoc
     */
    public function onSucceededTransfer()
    {
        Yii::$app->end();
    }

    /**
     * @inheritdoc
     */
    public function onSentRedEnvelope()
    {
        Yii::$app->end();
    }

    /**
     * @inheritdoc
     */
    public function onReceivedRedEnvelope()
    {
        Yii::$app->end();
    }
}
```