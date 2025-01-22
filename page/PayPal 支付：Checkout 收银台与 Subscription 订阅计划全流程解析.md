> 废话不多说，我们从请求的生命周期开始分析，逐步实现整个过程。

## 一、生命周期

### 1. Checkout - 收银台支付

#### 流程拆解

流程如下所示：

1. 本地应用组装好参数并请求 Checkout 接口，接口同步返回一个支付 URL；
2. 本地应用重定向至该 URL，用户登录 PayPal 账户并确认支付，支付完成后跳转至设置好的本地应用地址；
3. 本地请求 PayPal 执行付款接口发起扣款；
4. PayPal 发送异步通知至本地应用，本地接收数据包后进行验签操作；
5. 验签成功后，执行支付完成后的业务逻辑（如修改订单状态、增加销量、发送邮件等）。

### 2. Subscription - 订阅支付

#### 流程拆解

1. 创建一个订阅计划；
2. 激活该计划；
3. 使用已激活的计划创建订阅申请；
4. 本地跳转至订阅申请链接，用户授权并完成首期付款，支付完成后携带 token 跳转至设置好的本地应用地址；
5. 回跳后请求执行订阅；
6. 接收订阅授权异步回调结果，验证支付异步回调成功后，执行支付完成后的业务逻辑。

## 二、具体实现

> 了解了以上流程，接下来开始 Coding。

### Checkout 实现

#### 安装扩展

bash
$ composer require paypal/rest-api-sdk-php:*


#### 配置文件

创建配置文件 `config/paypal.php`，内容如下：

php
<?php

return [
    'sandbox' => [
        'client_id' => env('PAYPAL_SANDBOX_CLIENT_ID', ''),
        'secret' => env('PAYPAL_SANDBOX_SECRET', ''),
        'notify_web_hook_id' => env('PAYPAL_SANDBOX_NOTIFY_WEB_HOOK_ID', ''),
    ],
    'live' => [
        'client_id' => env('PAYPAL_CLIENT_ID', ''),
        'secret' => env('PAYPAL_SECRET', ''),
        'notify_web_hook_id' => env('PAYPAL_NOTIFY_WEB_HOOK_ID', ''),
    ],
];


#### 创建 PayPal 服务类

创建 `app/Services/PayPalService.php`，实现 Checkout 支付逻辑。

php
<?php

namespace App\Services;

use PayPal\Api\Payment;
use PayPal\Api\Payer;
use PayPal\Api\Amount;
use PayPal\Api\Transaction;
use PayPal\Api\RedirectUrls;
use PayPal\Auth\OAuthTokenCredential;
use PayPal\Rest\ApiContext;

class PayPalService
{
    protected $apiContext;

    public function __construct($config)
    {
        $this->apiContext = new ApiContext(
            new OAuthTokenCredential($config['client_id'], $config['secret'])
        );
        $this->apiContext->setConfig(['mode' => $config['mode']]);
    }

    public function checkout($order)
    {
        $payer = new Payer();
        $payer->setPaymentMethod('paypal');

        $amount = new Amount();
        $amount->setCurrency($order->currency)
               ->setTotal($order->total_amount);

        $transaction = new Transaction();
        $transaction->setAmount($amount)
                    ->setDescription($order->description);

        $redirectUrls = new RedirectUrls();
        $redirectUrls->setReturnUrl(route('payment.paypal.return'))
                     ->setCancelUrl(route('payment.paypal.cancel'));

        $payment = new Payment();
        $payment->setIntent('sale')
                ->setPayer($payer)
                ->setTransactions([$transaction])
                ->setRedirectUrls($redirectUrls);

        $payment->create($this->apiContext);

        return $payment->getApprovalLink();
    }
}


#### 注册服务类

在 `app/Providers/AppServiceProvider.php` 中注册 PayPal 服务类。

php
$this->app->singleton('paypal', function () {
    $config = config('paypal.sandbox');
    return new PayPalService($config);
});


#### 创建控制器

创建 `app/Http/Controllers/PaymentController.php`，实现支付逻辑。

php
<?php

namespace App\Http\Controllers;

use App\Models\Order;

class PaymentController extends Controller
{
    public function payByPayPalCheckout(Order $order)
    {
        $approvalUrl = app('paypal')->checkout($order);
        return redirect($approvalUrl);
    }

    public function payPalReturn()
    {
        // TODO: 支付成功后的业务逻辑
    }
}


#### 路由配置

在 `routes/web.php` 中添加路由。

php
Route::get('payment/{order}/paypal', 'PaymentController@payByPayPalCheckout')->name('payment.paypal_checkout');
Route::get('payment/paypal/return', 'PaymentController@payPalReturn')->name('payment.paypal.return');


### Subscription 实现

#### 创建计划与激活计划

在 `PayPalService` 中添加创建计划的方法。

php
public function createPlan($order)
{
    $plan = new Plan();
    $plan->setName($order->name)
         ->setDescription($order->description)
         ->setType('INFINITE');

    $paymentDefinition = new PaymentDefinition();
    $paymentDefinition->setName('Regular Payments')
                      ->setType('REGULAR')
                      ->setFrequency('MONTH')
                      ->setFrequencyInterval('1')
                      ->setCycles('0')
                      ->setAmount(new Currency(['value' => $order->price, 'currency' => $order->currency]));

    $plan->setPaymentDefinitions([$paymentDefinition]);

    $merchantPreferences = new MerchantPreferences();
    $merchantPreferences->setReturnUrl(route('subscriptions.paypal.return'))
                        ->setCancelUrl(route('subscriptions.paypal.cancel'));

    $plan->setMerchantPreferences($merchantPreferences);

    $plan->create($this->apiContext);

    return $plan;
}


#### 创建订阅申请

php
public function createAgreement($plan, $order)
{
    $agreement = new Agreement();
    $agreement->setName($plan->getName())
              ->setDescription($plan->getDescription())
              ->setStartDate(Carbon::now()->addMonth()->toIso8601String());

    $agreement->setPlan($plan);

    $payer = new Payer();
    $payer->setPaymentMethod('paypal');
    $agreement->setPayer($payer);

    $agreement->create($this->apiContext);

    return $agreement->getApprovalLink();
}


#### 路由配置

在 `routes/web.php` 中添加路由。

php
Route::get('subscriptions/{order}/paypal/plan', 'SubscriptionsController@createPlan')->name('subscriptions.paypal.createPlan');
Route::get('subscriptions/paypal/return', 'SubscriptionsController@executeAgreement')->name('subscriptions.paypal.return');


## 广告推荐

👉 [WildCard | 一分钟注册，轻松订阅海外线上服务](https://bit.ly/bewildcard)