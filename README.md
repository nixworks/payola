# Payola

[![Gem Version](https://badge.fury.io/rb/payola-payments.svg)](http://badge.fury.io/rb/payola-payments) [![CircleCI](https://circleci.com/gh/peterkeen/payola.svg?style=shield)](https://circleci.com/gh/peterkeen/payola) [![Dependency Status](https://gemnasium.com/peterkeen/payola.svg)](https://gemnasium.com/peterkeen/payola)

Payments with Stripe for your Rails application.

## What does this do?

Payola is a drop-in Rails engine that lets you sell one or more products by just including a module in your models. It includes:

* An easy to embed, easy to customize, async Stripe Checkout button
* Asynchronous payments, usable with any background processing system
* Full webhook integration
* Easy extension hooks for adding your own functionality

To see Payola in action, check out the site for [Mastering Modern Payments: Using Stripe with Rails](https://www.masteringmodernpayments.com). Read the book to find out the whys behind Payola's design.

## Installation

Add Payola to your Gemfile:

```ruby
gem 'payola-payments'
```

Run the installer and install the migrations:

```bash
$ rails g payola:install
$ rake db:migrate
```

### Single Sales

To start selling products, just include `Payola::Sellable`. For example, if you have a `Book` model:

```ruby
class Book < ActiveRecord::Base
  include Payola::Sellable
end
```

Each sellable model requires three attributes:

* `price`, (attribute) an amount in the format that Stripe expects. For USD this is cents.
* `permalink`, (attribute) a human-readable slug that is exposed in the URL
* `name`, (attribute) a human-readable name exposed on product pages

There are also two optional methods you can implement on your sellable:

* `redirect_path` takes the sale as an argument and returns a path. The buyer's browser will be redirected to that path after a successful sale. This defaults to `/`.
* `currency` returns the currency for this product. Payola will default to `usd`.

When people buy your product, Payola records information in `Payola::Sale` records and will record history if you have the `paper_trail` gem installed. **It is highly recommended to install paper_trail**.

### Checkout Button

To sell a product, use the `checkout` partial like this:

```rhtml
<%= render 'payola/transactions/checkout', sellable: YourProductClass.first %>
```

This will insert a Stripe Checkout button. The `checkout` partial has a bunch of options:

* `sellable`: The product to sell. Required.
* `button_text`: What to put on the button. Defaults to "Pay Now"
* `button_class`: What class to put on the actual button. Defaults to "stripe-button-el".
* `name`: What to put at the top of the Checkout popup. Defaults to `product.name`.
* `description`: What to show as the description in the popup. Defaults to product name + the formatted price.
* `product_image_path`: An image to insert into the Checkout popup. Defaults to blank.
* `panel_label`: The label of the button in the Checkout popup.
* `allow_remember_me`: Whether to show the Remember me checkbox. Defaults to true.
* `email`: Email address to pre-fill. Defaults to blank.

## Configuration

```ruby
# config/initializers/payola.rb

Payola.configure do |payola|
  payola.secret_key = 'sk_live_iwillnevertell'
  payola.publishable_key = 'pk_live_iguessicantell'

  payola.subscribe 'payola.book.sale.finished' do |sale|
    SaleMailer.receipt(sale.guid).deliver
  end

  payola.subscribe 'payola.book.sale.failed' do |sale|
    SaleMailer.admin_failed(sale.guid).deliver
  end

  payola.subscribe 'payola.book.sale.refunded' do |sale|
    SaleMailer.admin_refunded(sale.guid).deliver
  end
end
```

### Keys

You can set your Stripe keys in two ways:

1. By setting `STRIPE_SECRET_KEY` and `STRIPE_PUBLISHABLE_KEY` environment variables to their corresponding values.
2. By setting the `secret_key` and `publishable_key` settings in your Payola config initializer as shown above.

### Events

Payola wraps the StripeEvent gem for event processing and adds a few special sale-related events. Each one of these events passes the related `Sale` instance instead of a `Stripe::Event`. They are sent in-process so you don't have to wait for Stripe to send the corresponding webhooks.

* `payola.<product_class>.sale.finished`, when a sale completes successfully
* `payola.<product_class>.sale.failed`, when a charge fails
* `payola.<product_class>.sale.refunded`, when a charge is refunded

(In these examples, `<product_class>` is the underscore'd version of the product's class name.)

### Pre-charge Filter

You can set a callback that Payola will call immediately before attempting to make a charge. You can use this to, for example, check to see if the email address has been used already. To stop Payola from making a charge, throw a `RuntimeError`. The sale will be set to `errored` state and the message attached to the runtime error will be propogated back to the user.

```ruby
Payola.configure do |payola|
  payola.charge_verifier = lambda do |sale|
    raise "Improper sale!" unless sale.amount > 10_000_000
  end
end
```

### Webhooks

You can subscribe to any webhook events you want as well. Payola will dedupe events as they come in. Make sure to set your webhook address in Stripe's management interface to:

`https://www.example.com/payola/events`

To subscribe to a webhook event:

```ruby
Payola.configure do |payola|
  payola.subscribe 'charge.succeeded' do |event|
    sale = Sale.find_by(stripe_id: event.data.object.id)
    SaleMailer.admin_receipt(sale.guid)
  end
end
```

Payola uses `StripeEvent#event_retriever` internally. If you would like to customize or filter the events that come through, use Payola's `event_filter`:

```ruby
Payola.configure do |payola|
  payola.event_filter = lambda do |event|
    return nil unless event.blah?
    event
  end
end
```

`event_filter` takes an event and returns either `nil` or an event. If you return nil, the event ID will be recorded in the database but no further action will be taken. Returning the event allows processing to continue.

### Background Jobs

Payola will attempt to auto-detect the job queuing system you are using. It currently supports the following systems:

* ActiveJob (`:active_job`)
* Sidekiq (`:sidekiq`)
* SuckerPunch (`:sucker_punch`)

Payola will attempt ActiveJob *first* and then move on to try autodetecting other systems. If you want to force Payola to use a specific supported system, just set `background_worker` to the appropriate symbol. For example:

```ruby
Payola.background_worker = :sidekiq
```

You can also set this to anything with a `call` method, for complete control over how Payola's jobs get queued. For example, you can run transactions in-process like this:

```ruby
Payola.background_worker = lambda do |sale|
  sale.process!
end
```

## TODO

* Custom forms
* Subscriptions
* Affiliate tracking
* Coupon codes

## License

Please see the LICENSE file for licensing details.

## Author

Pete Keen, [@zrail](https://twitter.com/zrail), [https://www.petekeen.net](https://www.petekeen.net)

