# مخطط قاعدة البيانات (Firebase Firestore Schema)

> التصميم يعتمد Collection/Document. كل مستند رئيسي ممكن يحتوي Sub-collection (مثل الرسائل داخل order).

---

## 1. المجموعات الرئيسية (Collections)

| Collection | مثال المستند | أهم الحقول |
| --- | --- | --- |
| **users** | `users/{uid}` | `role` (`client` / `provider` / `admin`)<br>`fullName`, `phone`, `email`<br>`businessType` (مطعم، صالون…)
 `skills` (تصوير، تصميم…)<br>`rating` (float، افتراضي 0)<br>`verified` (bool) |
| **services** | `services/{serviceId}` | `providerId` (`uid`)<br>`category` (`photography` / `design` / `model`)<br>`title`, `description`, `basePrice`<br>`mediaUrls` (صور/فيديو)<br>`avgRating`, `reviewsCount` |
| **orders** | `orders/{orderId}` | `clientId`, `providerId`<br>`serviceId`, `category`<br>`price`, `commissionPct`<br>`status` (`pending`, `in_progress`, `review`, `completed`, `disputed`)<br>`milestones` (array) |
| **payments** | `payments/{paymentId}` | `orderId`, `amount`, `currency`<br>`method` (ZainCash…)<br>`waylRef` (string)<br>`escrowStatus` (`hold`, `captured`, `refunded`)<br>`createdAt` |
| **disputes** | `disputes/{disputeId}` | `orderId`, `openedBy` (`client`/`provider`)<br>`reason`, `status` (`open`,`resolved`,`rejected`)<br>`decision` |
| **chats** | `chats/{orderId}/messages/{msgId}` | `senderId`, `text`, `type` (`text`,`image`...)<br>`timestamp` |
| **reviews** | `reviews/{orderId}` | `clientId`, `providerId`, `rating`, `comment` |

---

## 2. الفهارس المقترَحة (Indexes)

1. `orders` – مركّب: `providerId` + `status`
2. `services` – مفرد: `category`
3. `payments` – مركّب: `orderId` + `escrowStatus`
4. `reviews` – مركّب: `providerId` + `rating`

> إضافة أي استعلام جديد يتطلب Composite Index يجب تحديث هذا القسم.

---

## 3. قواعد الأمان (Security Rules – ملخّص)

```firebase
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ==[ Users ]==
    match /users/{userId} {
      allow read: if isSignedIn();
      allow write: if isOwner(userId) || isAdmin();
    }

    // ==[ Services ]==
    match /services/{sid} {
      allow read: if true;
      allow create, update: if isProvider() &&
        request.auth.uid == request.resource.data.providerId;
      allow delete: if isAdmin();
    }

    // ==[ Orders ]==
    match /orders/{oid} {
      allow read: if isOrderParticipant(oid) || isAdmin();
      allow create: if isClient();
      allow update: if isOrderParticipant(oid);
    }

    // ==[ Payments ]==
    match /payments/{pid} {
      allow read: if isOrderParticipant(resource.data.orderId) || isAdmin();
      allow write: if isFunction();
    }

    // ==[ Chats ]==
    match /chats/{orderId}/{msg=**} {
      allow read, write: if isOrderParticipant(orderId);
    }

    // ==[ Reviews ]==
    match /reviews/{orderId} {
      allow create: if request.auth.uid == getOrder(orderId).clientId;
      allow read: if true;
    }

    /* Helper functions */
    function isSignedIn()  { return request.auth != null; }
    function isAdmin()     { return request.auth.token.role == 'admin'; }
    function isClient()    { return request.auth.token.role == 'client'; }
    function isProvider()  { return request.auth.token.role == 'provider'; }
    function isOwner(uid)  { return request.auth.uid == uid; }
    function getOrder(oid) { return get(/databases/$(database)/documents/orders/$(oid)).data; }
    function isOrderParticipant(oid) {
      let ord = getOrder(oid);
      return request.auth.uid == ord.clientId || request.auth.uid == ord.providerId;
    }
    function isFunction()  {
      return request.auth.token.firebase.sign_in_provider == 'custom';
    }
  }
}
```

> `isFunction()` تتأكد إن الكتابة على `payments` تتم فقط من Cloud Functions باستخدام Custom Token.

---

## 4. سياسات الاحتفاظ (Retention)

| Collection | مدة الاحتفاظ | آلية الحذف |
| --- | --- | --- |
| chats | 6 أشهر | Cloud Scheduler + Function (batch delete) |
| storage/files | 30 يوم بعد `completed` | Trigger بعد تغيير حالة الطلب |
| disputes | دائم | محفوظ لأغراض قانونية |
| analytics | BigQuery export | تُدار خارج Firestore |

---

## 5. نقاط بحاجة تأكيد

1. إضافة Collection `portfolios` منفصلة أو استخدام `services` فقط؟
2. أرشفة أو حذف أوامر أقدم من سنة لتقليل التكلفة؟
3. هل نضيف حقول `companyName`, `location.lat/lng` داخل `users`؟

> جاوب على الأسئلة أعلاه حتى نجمّد المخطط ونطبّق الـ Rules.

---

## ملحق توضيحي (أمثلة من واقع التطبيق)

> حتى تتخيّل شلون ترتبط الـ Collections ويّا شاشات الموبايل، خلي نستعرض أمثلة مبسّطة باللهجة العراقية.

### 1. مثال تسجيل مستخدم جديد
- شاشة “إنشاء حساب”.
- بعد ما يدخل الرقم والـ OTP ينجح، تنكتب وثيقة:
  ```json
  users/{uid} = {
    "role": "client",
    "fullName": "أبو حيدر برغر",
    "phone": "+9647701234567",
    "businessType": "مطعم",
    "rating": 0,
    "verified": false
  }
  ```

### 2. مزوّد يضيف خدمة تصوير داخلية
- يختار “إضافة خدمة” → يملأ العنوان “تصوير أطباق احترافي”.
- تترفع صورتين ويرسل.
- تتولد وثيقة:
  ```json
  services/{serviceId} = {
    "providerId": "UidOfProvider",
    "category": "photography",
    "title": "تصوير أطباق احترافي",
    "basePrice": 75000,
    "mediaUrls": ["https://.../1.jpg", "https://.../2.jpg"],
    "avgRating": 0,
    "reviewsCount": 0
  }
  ```

### 3. العميل يحجز الخدمة
- على شاشة تفاصيل الخدمة يضغط “احجز” ويحدد السعر 80,000.
- ينخزن:
  ```json
  orders/{orderId} = {
    "clientId": "UidClient",
    "providerId": "UidOfProvider",
    "serviceId": "serviceId",
    "category": "photography",
    "price": 80000,
    "commissionPct": 18,
    "status": "pending",
    "milestones": [
      {"title": "العربون", "amount": 14400},
      {"title": "باقي المبلغ", "amount": 65600}
    ]
  }
  ```

### 4. الدفع عبر ZainCash
- Cloud Function تحجز العربون (Hold) وتسجّل:
  ```json
  payments/{pid} = {
    "orderId": "orderId",
    "amount": 14400,
    "currency": "IQD",
    "method": "ZainCash",
    "waylRef": "W-123456",
    "escrowStatus": "hold",
    "createdAt": 1691420000
  }
  ```

### 5. الدردشة
- العميل يبعث رسالة “هلا بالمصور”.
- تتخزن:
  ```json
  chats/orderId/messages/msgId = {
    "senderId": "UidClient",
    "text": "هلا بالمصور",
    "type": "text",
    "timestamp": 1691420300
  }
  ```

### 6. المراجعة بعد الإكمال
- بعد ما يضغط “استلمت”، العميل يقيّم 5 نجوم.
  ```json
  reviews/orderId = {
    "clientId": "UidClient",
    "providerId": "UidOfProvider",
    "rating": 5,
    "comment": "شغل مرتب!",
    "createdAt": 1691430000
  }
  ```

> بهاي الأمثلة تلاحظ كل حركة بالموبايل تتحول لوثيقة/تحديث واضح بالقاعدة، ويسهّل نتابع الطلبات ونبني التقارير.

---

### 7. المزوّد يحدّث مرحلة (Milestone)
- من شاشة "تفاصيل الطلب" يضغط زر **ابدأ المرحلة**.
- التطبيق يغيّر أول عنصر داخل `milestones[0].status = "in_progress"`:
  ```json
  orders/{orderId}/milestones/0 = {
    "title": "العربون",
    "amount": 14400,
    "status": "in_progress"
  }
  ```
- لما يضغط **تمَّت المرحلة**، تتغيّر إلى `completed` ويصعد تنبيه للعميل.

### 8. فتح نزاع وحلّه
- العميل من زر "🚩 عندي مشكلة" يختار السبب "تأخير".
  ```json
  disputes/{disputeId} = {
    "orderId": "orderId",
    "openedBy": "client",
    "reason": "delay",
    "status": "open"
  }
  ```
- داخل لوحة الـ Admin، الموظف يضغط **حل النزاع** ويقرّر رد نصف المبلغ:
  ```json
  disputes/{disputeId}.status   = "resolved"
  disputes/{disputeId}.decision = "refund_50"
  payments/{pid}.escrowStatus   = "refunded"
  ```

### 9. إيقاف حساب مزوّد مخالف
- الـ Admin من صفحة المستخدم يضغط **تعليق الحساب**.
  ```json
  users/{uid} = {
    ...existingFields,
    "suspended": true,
    "suspendedAt": 1691500000
  }
  ```
- بعدها أي محاولة دخول من هالـ UID ترفضها القاعدة `rules` لأن الحقل `suspended` = true.

### 10. بانر ترويجي (Promo Banner)
- إدارة المحتوى تضيف عرض جديد "خصم 30% على التصوير".
  ```json
  promos/{promoId} = {
    "title": "خصم 30% على التصوير",
    "imageUrl": "https://.../promo.png",
    "startsAt": 1691600000,
    "endsAt": 1692200000
  }
  ```
- واجهة Home Screen تجيب `promos` الفعّالة وتعرضها سلايد.

### 11. شاشة "أرباحي" للمزوّد
- من Bottom Bar يضغط **💰 أرباحي** → التطبيق يستعلم عن:
  ```javascript
  db.collection('payments')
    .where('providerId', '==', uid)
    .where('escrowStatus', '==', 'captured')
  ```
- ويحسب مجموع `amount` لعرضه ضمن Card "رصيد قابل للسحب".

> بهالأمثلة الإضافية صار عندنا سيناريوهات تشمل التقدم بالمراحل، النزاعات، إدارة النظام، والعروض الترويجية.
