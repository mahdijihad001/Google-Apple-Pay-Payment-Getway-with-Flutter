# Node.js (Express) + Flutter integration for Google Pay & Apple Pay

এই ডকুমেন্টে আমি একটি working উদাহরণ দিলাম — **Node.js (Express) backend** যা Stripe দিয়ে PaymentIntent তৈরি করে এবং **Flutter** ক্লায়েন্ট যা Google Pay / Apple Pay থেকে token নিয়ে backend-এ পাঠাবে।

> কোডগুলো একটি reference implementation — production-এ ব্যবহার করার আগে অবশ্যই security, error-handling, এবং PCI/DSS নির্দেশিকা অনুযায়ী যাচাই করে নেবে।

---

## প্রস্তাবিত ফোল্ডার স্ট্রাকচার

```
server/
  ├─ .env
  ├─ package.json
  ├─ index.js
  └─ routes/payment.js

flutter_app/
  ├─ pubspec.yaml
  └─ lib/main.dart
```

---

# Backend (Node.js + Express + Stripe)

**ফাইল: server/package.json**

```json
{
  "name": "gpay-applepay-server",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "stripe": "^12.0.0",
    "body-parser": "^1.20.2",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3"
  }
}
```

**ফাইল: server/.env (example)**

```
STRIPE_SECRET_KEY=sk_test_xxx
PORT=4242
FRONTEND_ORIGIN=http://localhost:8080
```

**ফাইল: server/index.js**

```js
import express from "express";
import dotenv from "dotenv";
import cors from "cors";
import bodyParser from "body-parser";
import paymentRoutes from "./routes/payment.js";

dotenv.config();

const app = express();
const PORT = process.env.PORT || 4242;

app.use(cors({ origin: process.env.FRONTEND_ORIGIN || true }));
app.use(bodyParser.json());

app.use("/payment", paymentRoutes);

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

**ফাইল: server/routes/payment.js**

```js
import express from "express";
import Stripe from "stripe";

const router = express.Router();
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, { apiVersion: "2023-11-01" });

// Endpoint: create PaymentIntent from client token (Google/Apple token)
router.post("/create-payment", async (req, res) => {
  try {
    const { amount, currency, paymentToken, description } = req.body;

    if (!amount || !paymentToken) {
      return res.status(400).json({ error: "amount and paymentToken required" });
    }

    // Create a PaymentMethod using tokenized card data
    // Use payment_method_data with token depending on token format
    // We'll try creating PaymentIntent by directly using payment_method_data
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount),
      currency: currency || "usd",
      payment_method_data: {
        type: "card",
        card: {
          token: paymentToken // Google/Apple token string from client
        }
      },
      confirm: true,
      description: description || "Mobile app payment"
    });

    return res.json({ success: true, paymentIntent });
  } catch (error) {
    console.error("payment error:", error);
    // Sometimes Stripe returns nested error objects
    return res.status(500).json({ error: error.message || error.raw?.message });
  }
});

export default router;
```

### ব্যাখ্যা

* Mobile app `paymentToken` পাঠাবে — এটি Google Pay বা Apple Pay থেকে পাওয়া token।
* সার্ভার Stripe-এ `PaymentIntent` তৈরি করে এবং `payment_method_data.card.token` হিসেবে token ব্যবহার করে confirm করে।

> **নোট:** বিভিন্ন tokenization providers token format ভিন্ন হতে পারে। যদি উপরের পদ্ধতিতে error আসে, বিকল্প হল `stripe.tokens.create({card: { ... }})` বা PaymentMethod token use করে confirm করা — কিন্তু সাধারণত `payment_method_data.card.token` কাজ করে যখন token একটি compatible token হয়।

---

# Webhook (Optional — সাধারণত ব্যবহার করা ভাল)

**ফাইল: server/index.js** এ `.use(bodyParser.raw({type: 'application/json'}))` দিয়ে webhook verification আলাদা করে রাখতে হবে। এখানে আমি webhook full দেখাইনি — production-এ অবশ্যই webhook verification যোগ করবে।

---

# Flutter ক্লায়েন্ট (Google Pay & Apple Pay)

এই উদাহরণে `pay` প্যাকেজ ব্যবহার করা হয়েছে (Google-এর maintained package):

**pubspec.yaml (উদ্ধৃতি অংশ)**

```yaml
name: gpay_applepay_example
description: Example app
publish_to: 'none'
version: 1.0.0

environment:
  sdk: '>=2.18.0 <3.0.0'

dependencies:
  flutter:
    sdk: flutter
  pay: ^1.0.9
  http: ^0.13.5
```

**lib/main.dart**

```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:pay/pay.dart';
import 'package:http/http.dart' as http;

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: PaymentPage(),
    );
  }
}

class PaymentPage extends StatefulWidget {
  @override
  _PaymentPageState createState() => _PaymentPageState();
}

class _PaymentPageState extends State<PaymentPage> {
  // Google Pay config (example - merchant name & allowed networks)
  final _googlePayConfig = '''
  {
    "provider": "google_pay",
    "data": {
      "environment": "TEST",
      "apiVersion": 2,
      "apiVersionMinor": 0,
      "allowedPaymentMethods": [
        {
          "type": "CARD",
          "parameters": {
            "allowedAuthMethods": ["PAN_ONLY", "CRYPTOGRAM_3DS"],
            "allowedCardNetworks": ["AMEX", "MASTERCARD", "VISA"]
          },
          "tokenizationSpecification": {
            "type": "PAYMENT_GATEWAY",
            "parameters": {
              "gateway": "stripe",
              "stripe:version": "2020-08-27",
              "stripe:publishableKey": "pk_test_xxx"
            }
          }
        }
      ]
    }
  }
  ''';

  // Apple Pay config
  final _applePayConfig = '''
  {
    "provider": "apple_pay",
    "data": {
      "merchantIdentifier": "merchant.com.example",
      "displayName": "Example",
      "merchantCapabilities": ["3DS", "debit", "credit"],
      "countryCode": "US",
      "currencyCode": "USD"
    }
  }
  ''';

  // Example payment items
  final _paymentItems = <PaymentItem>[
    PaymentItem(label: 'Total', amount: '4.99', status: PaymentItemStatus.final_price),
  ];

  Future<void> _handlePayResult(Map<String, dynamic> result) async {
    // `result` contains tokenizationData from the pay plugin
    // token location depends on provider. For Google Pay (with gateway=stripe) token is:
    // result['paymentMethodData']['tokenizationData']['token']

    String token = '';
    try {
      if (result.containsKey('paymentMethodData')) {
        final pmd = result['paymentMethodData'];
        token = pmd['tokenizationData']['token'];
      } else if (result.containsKey('token')) {
        token = result['token'];
      }

      // Send this token to your backend to create & confirm PaymentIntent
      final resp = await http.post(
        Uri.parse('http://localhost:4242/payment/create-payment'),
        headers: { 'Content-Type': 'application/json' },
        body: jsonEncode({
          'amount': 499, // amount in cents
          'currency': 'usd',
          'paymentToken': token,
          'description': 'Test payment from Flutter app'
        }),
      );

      final body = jsonDecode(resp.body);
      if (resp.statusCode == 200 && body['success'] == true) {
        _showMessage('Payment success: ' + body['paymentIntent']['status']);
      } else {
        _showMessage('Payment failed: ' + (body['error'] ?? resp.body));
      }
    } catch (e) {
      _showMessage('Exception: ' + e.toString());
    }
  }

  void _showMessage(String m) {
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(m)));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Google/Apple Pay Example')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            SizedBox(height: 20),
            // Google Pay Button
            GooglePayButton(
              paymentConfiguration: PaymentConfiguration.fromJsonString(_googlePayConfig),
              paymentItems: _paymentItems,
              type: GooglePayButtonType.pay,
              onPaymentResult: (data) => _handlePayResult(data as Map<String, dynamic>),
              loadingIndicator: const CircularProgressIndicator(),
            ),
            SizedBox(height: 20),
            // Apple Pay Button
            ApplePayButton(
              paymentConfiguration: PaymentConfiguration.fromJsonString(_applePayConfig),
              paymentItems: _paymentItems,
              style: ApplePayButtonStyle.black,
              type: ApplePayButtonType.buy,
              onPaymentResult: (data) => _handlePayResult(data as Map<String, dynamic>),
              loadingIndicator: const CircularProgressIndicator(),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

# Important Production Notes

1. **Apple Pay domain verification:** Apple requires you to host a verification file at `https://yourdomain/.well-known/apple-developer-merchantid-domain-association`. আপনার domain-এ এই ফাইল host করতে হবে এবং Apple Developer account-এ domain verify করতে হবে.

2. **Google Pay merchant config:** Google Pay-এ merchant id & gateway configuration ঠিক করতে হবে; `environment` production করার আগে `TEST` থেকে `PRODUCTION`-এ ফিরিয়ে আনিবে এবং real `stripe:publishableKey` ব্যবহার করব।

3. **TLS (HTTPS) বাধ্যতামূলক:** Mobile app + payments production-এ HTTPS lagbe।

4. **Webhook:** PaymentIntent-এর স্টেট change (succeeded/failed) verify করতে Stripe Webhook ব্যবহার কর। Webhook secret ব্যবহার করে verify করলে নিরাপদ।

5. **Token compatibility:** Google/Apple token format provider অনুযায়ী আলাদা হতে পারে। যদি direct `payment_method_data.card.token` কাজ না করে, Stripe docs অনুসারে token-to-paymentmethod বা token-to-source route প্রয়োগ করতে হবে।

6. **Stripe SDK alternative:** Mobile থেকে Stripe SDK (e.g., Stripe's mobile SDK / PaymentSheet) ব্যবহার করলে অনেক কাজ সহজ হয় এবং SCA/ERC handling স্বয়ংক্রিয় হয়।

---

# Testing locally

1. Set `.env` with your Stripe test secret key.
2. `npm install` inside `server/` এবং `npm start` চালাও।
3. Flutter app চালাও (emulator/device). Google Pay emulator supports TEST environment. Apple Pay testing requires a real device and properly configured merchant id + entitlements.

---

# Stripe Webhook Integration (Recommended)

Stripe webhook ব্যবহার করলে PaymentIntent এর final status নিশ্চিত হওয়া যায়। এটি fraud protection, dispute handling এবং asynchronous payment verification এর জন্য অত্যন্ত গুরুত্বপূর্ণ।

---

## 1. Server এ Webhook Endpoint যোগ করা

**ফাইল: `server/index.js`** — webhook এর জন্য raw body লাগবে, তাই নিচের মতো middleware যোগ করতে হবে:

```js
import express from "express";
import dotenv from "dotenv";
import cors from "cors";
import bodyParser from "body-parser";
import paymentRoutes from "./routes/payment.js";
import webhookRoutes from "./routes/webhook.js";

dotenv.config();

const app = express();
const PORT = process.env.PORT || 4242;

app.use(cors({ origin: process.env.FRONTEND_ORIGIN || true }));

// Stripe webhook needs raw body
app.use('/payment/webhook', bodyParser.raw({ type: 'application/json' }));

// Normal JSON body for other routes
app.use(bodyParser.json());

app.use("/payment", paymentRoutes);
app.use("/payment", webhookRoutes);

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## 2. Webhook Route তৈরি

**ফাইল: `server/routes/webhook.js`**

```js
import express from "express";
import Stripe from "stripe";

const router = express.Router();
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, { apiVersion: "2023-11-01" });

// Webhook-secret env
const endpointSecret = process.env.STRIPE_WEBHOOK_SECRET;

router.post("/webhook", async (req, res) => {
  const sig = req.headers['stripe-signature'];

  let event;

  try {
    event = stripe.webhooks.constructEvent(req.body, sig, endpointSecret);
  } catch (err) {
    console.error('Webhook signature verification failed:', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  // Handle event types
  if (event.type === 'payment_intent.succeeded') {
    const paymentIntent = event.data.object;
    console.log('Payment succeeded:', paymentIntent.id);
    // এখানে database update করতে পারো
  }

  if (event.type === 'payment_intent.payment_failed') {
    const paymentIntent = event.data.object;
    console.log('Payment failed:', paymentIntent.last_payment_error?.message);
  }

  res.json({ received: true });
});

export default router;
```

---

## 3. `.env` এ Webhook Secret যোগ করা

```
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxx
```

---

## 4. Stripe CLI দিয়ে লোকালে Webhook টেস্ট করা

টার্মিনালে চালাও:

```
stripe listen --forward-to localhost:4242/payment/webhook
```

Test trigger:

```
stripe trigger payment_intent.succeeded
```

---

## 5. কেন Webhook বাধ্যতামূলক:

✔ PaymentIntent asynchronous (network / 3DSecure delay হতে পারে)
✔ Fraud checks complete হতে সময় লাগে
✔ Client-side payment failures detect নাও হতে পারে
✔ Server-side database এর সাথে sync প্রয়োজন

---
