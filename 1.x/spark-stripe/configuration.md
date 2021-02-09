# Configuration

[[toc]]

## Introduction

In the following documentation, we will discuss how to configure a Laravel Spark installation when using the [Stripe](https://stripe.com) payment provider. All of Spark's configuration options are housed in your application's `config/spark.php` configuration file.

## Stripe Configuration

Of course, to use Stripe as a payment provider for your Laravel Spark application, you should have an active Stripe account.

### Environment Variables

Next, you should configure the application environment variables that will be needed by Spark in order to access your Stripe account. These variables should be placed in your application's `.env` environment file.

Of course, you should adjust the variable's values to correspond to your own Stripe account's credentials. Your Stripe API credentials and public key are available in your Stripe account dashboard:

```bash
CASHIER_CURRENCY=USD
CASHIER_CURRENCY_LOCALE=en
STRIPE_KEY=pk_test_example
STRIPE_SECRET=sk_test_example
STRIPE_WEBHOOK_SECRET=sk_test_example
```

### Stripe Webhooks

In addition, your Spark powered application will need to receive webhooks from Stripe in order to keep your application's billing and subscription data in sync with Stripe's. Within your Stripe dashboard's webhook management panel, you should configure Stripe to send webhook alerts to your application's `/spark/webhook` URI. You should enable webhook alerts for the following events:

- customer.deleted
- customer.subscription.created
- customer.subscription.deleted
- customer.subscription.updated
- customer.updated
- invoice.payment_action_required
- invoice.payment_succeeded

#### Webhooks & Local Development

For Stripe to be able to send your application webhooks during local development, you will need to expose your application via a site sharing service such as [Ngrok](https://ngrok.io) or [Expose](https://beyondco.de/docs/expose/introduction). If you are developing your application locally using [Laravel Sail](http://laravel.com/docs/sail), you may use Sail's [site sharing command](https://laravel.com/docs/sail#sharing-your-site).

## Configuring Billables

Spark allows you to define the types of billable models that your application will be managing. Most commonly, applications bill individual users for monthly and yearly subscription plans. However, your application may choose to bill some other type of model, such as a team, organization, band, etc.

You may define your billable models within the `billables` array of your application's `spark` configuration file. By default, this array contains an entry for the `App\Models\User` model.

Before continuing, you should ensure that the model class that corresponds to your billable model is using the `Spark\Billable` trait:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Spark\Billable;

class User extends Authenticatable
{
    use Billable, HasFactory, Notifiable;
}
```

### Billable Slugs

As you may have noticed, each entry in the `billables` configuration array is keyed by a "slug" that is a shortened form of the billable model class. This slug can be used when accessing the Spark customer billing portal, such as `https://example.com/billing/user` or `https://example.com/billing/team`.

### Billable Resolution

When you installed Laravel Spark, an `App\Providers\SparkServiceProvider` class was created for you. Within this service provider, you will find a callback that is used by Spark to resolve the billable model instance when accessing the Spark billing portal. By default, this callback simply returns the currently authenticated user, which is the desired behavior for most applications using Laravel Spark:

```php
use App\Models\User;
use Illuminate\Http\Request;
use Spark\Spark;

Spark::billable(User::class)->resolve(function (Request $request) {
    return $request->user();
});
```

However, if your application is not billing individual users, you may need to adjust this callback. For example, if your application offers team billing instead of user billing, you might customize the callback like so:

```php
use App\Models\Team;
use Illuminate\Http\Request;
use Spark\Spark;

Spark::billable(Team::class)->resolve(function (Request $request) {
    return $request->user()->currentTeam;
});
```

### Billable Authorization

Next, let's examine the authorization callbacks that Spark will use to determine if the currently authenticated user of your application is authorized to view the billing portal for a particular billable model.

When you installed Laravel Spark, an `App\Providers\SparkServiceProvider` class was created for you. Within this service provider, you will find the authorization callback definition used to determine if a given user is authorized to view the billing portal for the `App\Models\User` billable class. Of course, if your application is not billing users, you should update the billable class and authorization callback logic to fit your application's needs. By default, Spark will simply verify that the currently authenticated user can only manage its own billing settings:

```php
use App\Models\User;
use Illuminate\Http\Request;
use Spark\Spark;

Spark::billable(User::class)->authorize(function (User $billable, Request $request) {
    return $request->user() &&
           $request->user()->id == $billable->id;
});
```

If the authorization callback returns `true`, the currently authenticated user will be authorized to view the billing portal and manage the billing settings for the given `$billable` model. If the callback returns `false`, the request to access the billing portal will be denied.

You are free to customize the `authorize` callback based on your own application's needs. For example, if your application bills teams instead of individual users, you might update the callback like so:

```php
use App\Models\Team;
use Illuminate\Http\Request;
use Spark\Spark;

Spark::billable(Team::class)->authorize(function (Team $billable, Request $request) {
    return $request->user() &&
           $request->user()->ownsTeam($billable);
});
```

## Defining Subscription Plans

As we previously discussed, Spark allows you to define the types of billable models that your application will be managing. These billable models are defined within the `billables` array of your application's `config/spark.php` configuration file:

Each billable configuration within the `billables` array contains a `plans` array. Within this array you may configure each of the billing plans offered by your application to that particular billable type. **The `monthly_id` and `yearly_id` identifiers should correspond to the price / plan identifiers configured within your Stripe account dashboard:**

```php
use App\Models\User;

'billables' => [
    'user' => [
        'model' => User::class,
        'trial_days' => 5,
        'plans' => [
            [
                'name' => 'Standard',
                'short_description' => 'This is a short, human friendly description of the plan.',
                'monthly_id' => 'price_id',
                'yearly_id' => 'price_id',
                'features' => [
                    'Feature 1',
                    'Feature 2',
                    'Feature 3',
                ],
            ],
        ],
    ],
]
```

If your subscription plan only offers a monthly billing cycle, you may omit the `yearly_id` identifier from your plan configuration. Likewise, if your plan only offers a yearly billing cycle, you may omit the `monthly_id` identifier.

In addition, you are free to supply a short description of the plan and a list of features relevant to the plan. This information will be displayed in the Spark billing portal.

## Accessing The Billing Portal

Once you have configured your Spark installation, you may access your application's billing portal at the `/billing` URI. So, if your application is being served on `localhost`, you may access your application's billing portal at `http://localhost/billing`.