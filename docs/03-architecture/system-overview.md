# نظرة عامة على المعمارية (System Overview)

> هذا الرسم يبيّن شلون تتواصل مكوّنات «وسط» ويّا بعض من لحظة تسجيل الدخول لحد تحرير الدفعة عبر العقد الذكي.

```mermaid
graph TD
  subgraph موبايل
    A[Flutter App\n\niOS & Android]
  end

  subgraph Firebase
    B1[Auth]
    B2[Firestore]
    B3[Storage]
    B4[Cloud Functions (Node18)]
    B5[Analytics]
  end

  subgraph دفع محلي
    C1[Wayl Gateway]
    C2[ZainCash / FastPay\nVisa/Master / Nass Wallet]
  end

  subgraph بلوكچين
    D[Smart Contract\n(Smart Quartez)\nPolygon]
  end

  subgraph ويب
    E[Admin Panel\n(React)]
  end

  %% الاتصالات
  A -- تسجيل / OTP --> B1
  A -- عمليات CRUD --> B2
  A -- رفع صور / فيديو --> B3
  A -- Event Tracking --> B5

  A -- طلب إنشاء دفعة --> B4
  B4 -- Hold العمولة --> D
  D -- OK Hash --> B4

  B4 -- استدعاء API --> C1
  C1 -- يوجّه للبوابات --> C2
  C2 -- إشعار دفع --> C1
  C1 -- Webhook --> B4
  B4 -- Capture باقي المبلغ --> D

  E -- استعلامات إدارة --> B2
  E -- تقرير مالي --> C1
```

---

## شرح المكوّنات

| المكوّن | الدور الرئيسي | ملاحظات تقنية |
| --- | --- | --- |
| **Flutter App** | واجهة المستخدم للعميل ومقدّم الخدمة. | Riverpod للإدارة؛ يمكن استخدام `dart:ffi` للتكامل مع SDKات الدفع عند الحاجة. |
| **Firebase Auth** | تسجيل دخول رقم هاتف أو إيميل مع OTP. | يفعّل مزوّدَي `phone` و`email/password`. |
| **Firestore** | قاعدة بيانات NoSQL (users, services, orders...). | Security Rules مع Roles؛ فهارس حسب الاستعلامات الحرجة. |
| **Storage** | حفظ الصور والفيديوهات (مؤقتًا بعد التسليم). | مهام حذف تلقائي بعد 30 يوم عبر Cloud Tasks. |
| **Cloud Functions** | منطق السيرفر (Hold/Capture، Webhooks، إشعارات). | Node 18 + TypeScript؛ يستدعي Wayl والعقد الذكي ويرسل Push. |
| **Analytics** | تتبع أحداث (login, order_create, escrow_release…). | Export إلى BigQuery للتحليلات المتقدمة. |
| **Wayl Gateway** | وسيط مالي يوجه المعاملات للبوابات المحلية. | REST API + Webhooks. |
| **بوابات الدفع** | ZainCash، FastPay، Visa/Master، Nass Wallet، COD. | COD يُسجل كـ Payment Intent بدون معالجة إلكترونية. |
| **Smart Contract (Polygon)** | يحتجز العمولة (Hold) ويطلقها (Capture) بعد الموافقة. | ERC-20 USDT؛ Events تُرسل لـ Cloud Functions. |
| **Admin Panel (React)** | إدارة مستخدمين، طلبات، نزاعات، تقارير مالية. | يعتمد Firebase Admin SDK + صلاحية “admin”. |

---

## تدفق أساسي (مختصر)

1. **العميل** ينشئ طلب ويحدد السعر → يُحفظ Doc في `orders/` وتستدعي **Function** عملية Hold للعمولة على العقد الذكي.
2. Function تنشئ رابط دفع عبر **Wayl** → العميل يدفع بالبوابة المناسبة.
3. **Webhook Wayl** يرجع نجاح الدفع → Function تغير حالة الطلب إلى `in_progress`.
4. بعد التسليم والموافقة، العميل يضغط “استلمت” → Function تعمل **Capture** لباقي المبلغ وترسله للمزوّد.
5. **Analytics** تسجل الحدث، و**Firestore** يحدث رصيد المزوّد.
6. **Admin Panel** تراقب كل العمليات وتتدخل عند أي نزاع.

> *أي إضافة مكوّن جديد (مثل CDN للفيديو) تستدعي تحديث هذا الملف والرسم أعلاه.*
