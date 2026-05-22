<div align="center">

<br/>

<img alt="iraq-pay" src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=800&size=44&duration=2400&pause=900&color=A78BFA&center=true&vCenter=true&width=900&height=80&lines=iraq-pay"/>

**One SDK. Every Iraqi payment gateway. Zero pain.**
_ZainCash · FIB · AsiaCell Cash · FastPay · Qi Card · Switch — unified API._

<br/>

```bash
npm i iraq-pay
# or
pip install iraq-pay
# or  
flutter pub add iraq_pay
```

<br/>

<p>
<img src="https://img.shields.io/badge/Iraq-006C35?style=for-the-badge&logoColor=white"/>
<img src="https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white"/>
<img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white"/>
<img src="https://img.shields.io/badge/Flutter-02569B?style=for-the-badge&logo=flutter&logoColor=white"/>
<img src="https://img.shields.io/badge/MIT-000000?style=for-the-badge"/>
</p>

<p>
<img src="https://img.shields.io/github/stars/kasimmj/iraq-pay?style=social"/>
<img src="https://img.shields.io/github/forks/kasimmj/iraq-pay?style=social"/>
</p>

</div>

---

## 🇮🇶 The Iraqi Payment Problem

كل بوابة دفع عراقية لها:
- API مختلف (REST vs SOAP vs custom)
- توثيق متفرق
- صياغة مختلفة للأرقام والعملات
- webhooks متباينة
- بيئة sandbox غير موحدة

**النتيجة:** أي مطور يقرر يدعم متجره أكثر من بوابة → 3 أسابيع شغل + bugs لا نهاية لها.

**`iraq-pay` يحل هذا.** SDK واحد. API موحد. كل البوابات.

---

## ⚡ One API, every gateway

```typescript
import { IraqPay } from "iraq-pay";

const pay = new IraqPay({
  zaincash: { merchant_id: "...", secret: "..." },
  fib: { client_id: "...", client_secret: "..." },
  asiacell: { api_key: "..." },
  fastpay: { merchant_code: "..." },
  qi: { terminal_id: "..." },
});

// Same API for every gateway
const session = await pay.createPayment({
  gateway: "zaincash",     // or "fib", "asiacell", "fastpay", "qi"
  amount: 50000,            // in IQD (Iraqi Dinar)
  currency: "IQD",
  reference: "ORDER-12345",
  customer: {
    name: "قاسم محمد",
    phone: "+9647763695936",
    email: "customer@example.com",
  },
  redirect_url: "https://yoursite.com/thanks",
  webhook_url: "https://yoursite.com/api/webhook",
});

console.log(session.payment_url);   // Redirect customer here
```

---

## 📦 Supported gateways

| Gateway | Type | Status | Sandbox |
|---------|------|--------|---------|
| **ZainCash** | Mobile wallet | ✅ Production | ✅ |
| **FIB** (First Iraqi Bank) | Card + wallet | ✅ Production | ✅ |
| **AsiaCell Cash** | Mobile wallet | ✅ Production | ✅ |
| **FastPay** | Card + wallet | ✅ Production | ✅ |
| **Qi Card** | Government salaries + e-commerce | ✅ Production | ⚠️ Limited |
| **Switch** | Visa/Mastercard via Iraqi switch | 🚧 Beta | ✅ |
| **Nass Wallet** | New mobile wallet | 🚧 Beta | ✅ |

We add new gateways quarterly. [Request one](https://github.com/kasimmj/iraq-pay/issues/new).

---

## 🏗️ Architecture — why this works

```
            ┌─────────────────────────────────────────┐
            │           Your App                       │
            │  (one SDK call — gateway-agnostic)       │
            └───────────────┬─────────────────────────┘
                            │
                ┌───────────▼────────────┐
                │     iraq-pay SDK        │
                │  - Normalizes inputs    │
                │  - Routes to gateway    │
                │  - Validates response   │
                │  - Handles webhooks     │
                │  - Idempotency keys     │
                │  - Retry policy         │
                └───────────┬────────────┘
                            │
         ┌──────────┬───────┴───────┬──────────┐
         ▼          ▼               ▼          ▼
   ┌─────────┐ ┌──────┐    ┌──────────┐  ┌────────┐
   │ZainCash │ │ FIB  │... │FastPay   │  │  Qi    │
   │  API    │ │ API  │    │  API     │  │  API   │
   └─────────┘ └──────┘    └──────────┘  └────────┘
```

Each gateway has its own adapter under the hood. Your code never sees the gateway-specific quirks.

---

## 🛡️ Webhook handling

Every gateway sends webhooks differently. We unify them:

```typescript
import { IraqPay } from "iraq-pay";

const pay = new IraqPay({ /* ... */ });

app.post("/api/webhook", async (req, res) => {
  const event = pay.parseWebhook({
    headers: req.headers,
    body: req.body,
    signature: req.headers["x-iraq-pay-signature"],
  });

  // Same event format regardless of source
  switch (event.type) {
    case "payment.completed":
      await markOrderPaid(event.reference, event.amount);
      break;
    case "payment.failed":
      await markOrderFailed(event.reference, event.failure_reason);
      break;
    case "payment.refunded":
      await markOrderRefunded(event.reference, event.refund_amount);
      break;
  }

  res.status(200).send("ok");
});
```

Signature verification is automatic per-gateway.

---

## 🌐 Refunds, splits, recurring

```typescript
// Refund
await pay.refund({
  gateway: "fib",
  payment_id: "pay_xyz",
  amount: 25000,                    // Partial refund OK
  reason: "Customer cancelled",
});

// Marketplace split (for multi-vendor stores)
await pay.createPayment({
  gateway: "zaincash",
  amount: 100000,
  splits: [
    { recipient: "vendor_1", amount: 70000 },     // 70%
    { recipient: "vendor_2", amount: 25000 },     // 25%
    { recipient: "platform",  amount: 5000 },     // 5% (your cut)
  ],
});

// Subscriptions (where supported)
await pay.createSubscription({
  gateway: "fib",
  plan: "monthly_50k",
  customer_id: "user_123",
  start_date: new Date(),
});
```

---

## 🧪 Sandbox / Testing

Every gateway has a sandbox. We auto-route to sandbox when `NODE_ENV=test`:

```typescript
const pay = new IraqPay({
  mode: "sandbox",                  // or "production" or "auto"
  // ...
});

// Use these test phone numbers for ZainCash sandbox:
//   +9647712345671  → always approves
//   +9647712345672  → always declines
//   +9647712345673  → timeout
//   +9647712345674  → 3D Secure required
```

---

## 🖥️ Local Dashboard (no code required)

Don't want to integrate the SDK? Run our local control panel:

```bash
docker run -p 8080:8080 -v ~/.iraq-pay:/data ghcr.io/kasimmj/iraq-pay-dashboard
```

Open `http://localhost:8080` and you get a beautiful Arabic+English dashboard with:

- 📊 **Live transactions** — every payment as it happens
- 💸 **One-click refunds** — no code needed
- 📈 **Per-gateway analytics** — which converts best
- 🔔 **Webhook tester** — simulate callbacks
- 🛠️ **Manual reconciliation** — match payments to orders
- 📤 **CSV export** — for accounting
- 🇮🇶 **Full RTL Arabic** — for your finance team

Perfect for small shops, accountants, and customer support teams.

---

## 📊 Built-in analytics (SDK)

```typescript
const stats = await pay.stats({
  from: "2026-05-01",
  to: "2026-05-31",
  group_by: ["gateway", "currency"],
});

// Returns:
[
  { gateway: "zaincash", currency: "IQD", count: 124, total: 6_200_000, success_rate: 0.94 },
  { gateway: "fib",       currency: "IQD", count: 87,  total: 12_800_000, success_rate: 0.97 },
  // ...
]
```

---

## 🌍 Languages

### TypeScript / JavaScript (Node.js)
```bash
npm install iraq-pay
```

### Python
```bash
pip install iraq-pay
```
```python
from iraq_pay import IraqPay

pay = IraqPay(zaincash={"merchant_id": "...", "secret": "..."})
session = pay.create_payment(gateway="zaincash", amount=50000, ...)
```

### Flutter / Dart
```bash
flutter pub add iraq_pay
```
```dart
final pay = IraqPay(
  zaincash: ZainCashConfig(merchantId: '...', secret: '...'),
);

final session = await pay.createPayment(
  gateway: Gateway.zaincash,
  amount: 50000,
  ...
);
```

### PHP (for WordPress / WooCommerce)
```bash
composer require kasimmj/iraq-pay
```

All four implementations expose **the same API surface** — switch languages without re-learning.

---

## 🇮🇶 Why this matters for Iraqi devs

- Every Iraqi e-commerce site reinvents this wheel
- Most existing wrappers are abandoned (last commit 2022)
- Documentation in Arabic is rare and often outdated
- This SDK is **maintained**, **versioned**, and **tested in production** (Solar iQ uses it across 6 cities)

---

## 🤝 Adding a new gateway

We accept PRs for any registered Iraqi payment provider. To add one:

1. Read [docs/adding-a-gateway.md](docs/adding-a-gateway.md)
2. Implement the `GatewayAdapter` interface (8 methods)
3. Add tests against the gateway's sandbox
4. Submit PR

We help with the API contracts if you have access to a gateway we don't.

---

## 📜 License

MIT.

---

<div align="center">

**Star ⭐ if you've ever cursed at an Iraqi payment API.**

</div>
