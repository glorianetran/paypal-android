---
title: Pay with PayPal Custom Integration
keywords: 
contentType: docs
productStatus: current
apiVersion: TODO
sdkVersion: TODO
---
# Pay with PayPal Custom Integration

Follow these steps to add Card payments:

1. [Know before you code](#know-before-you-code)
1. [Add PayPal Payments](#add-paypal-payments)
1. [Test and go live](#test-and-go-live)

## Know before you code

You will need to set up authorization to use the PayPal Payments SDK. 
Follow the steps in [Get Started](https://developer.paypal.com/api/rest/#link-getstarted) to create a client ID and generate an access token. 

You will need a server integration to create an order and capture the funds using [PayPal Orders v2 API](https://developer.paypal.com/docs/api/orders/v2). 
For initial setup, the `curl` commands below can be used in place of a server SDK.

## Add PayPal Payments

### 1. Add the Payments SDK  to your app

In your `build.gradle` file, add the following dependency:

```groovy
dependencies {
   implementation "com.paypal.android:paypal-checkout:1.0.0"[see step 8](#8-Capture-authorize-the-order)
}
```

### 2. Configure your app to handle browser switching

The PayPal payment flow utilizes a browser switch. A URL scheme must be defined to return to your app from the browser.

Edit your `AndroidManifest.xml` to include an `intent-filter` and set the `android:scheme` on your Activity that will be responsible for handling the deep link back into the app:

```xml
<activity android:name="com.company.app.MyPaymentsActivity"
    android:exported="true">
    ...
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <data android:scheme="custom-url-scheme"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
    </intent-filter>
</activity>
```

### 3. Create a PayPal button 

Add a `PayPalButton` to your layout XML:

```xml
<com.paypal.android.checkout.paymentbutton.PayPalButton
    android:id="@+id/payPalButton"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
```

### 4. Initiate the Payments SDK

Create a `CoreConfig` using your client ID from the PayPal Developer Portal:

```kotlin
val config = CoreConfig("<CLIENT_ID>", environment = Environment.SANDBOX)
```

Set the return URL from the URL scheme you configured in the `ActivityManifest.xml` [step 2](#2-configure-your-app-to-handle-browser-switching):

```kotlin
val returnUrl = "custom-url-scheme"
```

Create a `PayPalClient` to approve an order with a PayPal payment method:

```kotlin
val payPalClient = PayPalClient(requireActivity(), config, returnUrl)
```

Set a listener on your `PayPalClient` to handle results:

```kotlin
payPalClient.listener = PayPalCheckoutListener { result ->
    when (result) {
        is PayPalCheckoutResult.Success -> {
            // order was successfully approved and is ready to be captured/authorized (see step 8)
        } 
        is PayPalCheckoutResult.Failure -> {
            // handle the error by accessing `result.error`
        } 
        is PayPalCheckoutResult.Cancellation -> {
            // the user canceled
        } 
    }
}
```

### 5. Create an order

When a user enters the payment flow, call `v2/checkout/orders` to create an order and obtain an order ID:

```bash
curl --location --request POST 'https://api.sandbox.paypal.com/v2/checkout/orders/' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <ACCESS_TOKEN>' \
--data-raw '{
    "intent": "<CAPTURE|AUTHORIZE>",
    "purchase_units": [
        {
            "amount": {
                "currency_code": "USD",
                "value": "5.00"
            }
        }
    ]
}'
```

The `id` field of the response contains the order ID to pass to your client.

### 6. Create a request object for launching the PayPal flow

Configure your `PayPalRequest` and include the order ID generated [step 5](#5-create-an-order):

```kotlin
val payPalRequest = PayPalRequest("<ORDER_ID>")
```

### 7. Approve the order through the Payments SDK

When a user taps the PayPal button created above, approve the order using your `PayPalClient`.

Attach your PayPal button to the PayPal payment flow:

```kotlin
findViewById<View>(R.id.payPalButton).setOnClickListener {
    payPalClient.checkout(payPalRequest) 
}
```

When the user completes the PayPal payment flow, the result will be returned to the listener set in (step 4).

### 8. Capture/authorize the order

If you receive a successful result in the client-side flow, you can then capture or authorize the order. 

Call `authorize` to place funds on hold:

```bash
curl --location --request POST 'https://api.sandbox.paypal.com/v2/checkout/orders/<ORDER_ID>/authorize' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <ACCESS_TOKEN>' \
--data-raw ''
```

Call `capture` to capture funds immediately:

```bash
curl --location --request POST 'https://api.sandbox.paypal.com/v2/checkout/orders/<ORDER_ID>/capture' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <ACCESS_TOKEN>' \
--data-raw ''
```

## Testing and go live

### 1. Test the PayPal integratoin

Follow the [Create sandbox accounts](https://developer.paypal.com/api/rest/#link-createsandboxaccounts) instructions to create PayPal test accounts.
When prompted to login with PayPal during the payment flow on your mobile app, you can log in with the test account credentials created above to complete the Sandbox payment flow. 

### 2. Go live with your integration

Follow [these instructions](https://developer.paypal.com/api/rest/production/) to prepare your integration to go live.