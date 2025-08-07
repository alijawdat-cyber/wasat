# وسط | **wasat**

> أول منصة وساطة تسويقية رقمية متخصصة في العراق، تربط أصحاب الأعمال بمقدّمي الخدمات التسويقية وتوفّر نظام دفع آمن (Escrow) يحفظ حقوق الجميع.

## 📑 الفهرس
1. نظرة سريعة
2. المميزات الرئيسية (MVP)
3. التقنيات والأدوات
4. تشغيل المشروع محليًا
5. هيكل المجلدات
6. التوثيق الإضافي
7. الرخصة

---

## 1️⃣ نظرة سريعة
"وسط" يسهّل على المطاعم، العيادات، ومراكز التجميل إيجاد مصممين، مصورين، ومودلز محترفين خلال دقائق، مع آلية دفع مؤمنة بالعُقد الذكية (Smart Quartez).

## 2️⃣ المميزات الرئيسية (MVP)
- تسجيل دخول برقم الهاتف أو الإيميل مع توثيق OTP.
- 4 فئات خدمات: **تصميم – تصوير – مودل – كتابة محتوى**.
- Gateways دفع مدمجة: Zain Cash، Visa/Master، FastPay، First Bank Iraq، Nass Wallet، **Cash on Delivery**.
- نظام Escrow أوتوماتيكي يفرج المبلغ بعد تأكيد العميل.
- نظام تقييم/مراجعات وشارة "موثّق" لمقدّمي الخدمة.
- دعم العربية (لهجة عراقية) والإنجليزية.

## 3️⃣ التقنيات والأدوات
| الطبقة | التقنية |
| --- | --- |
| Front-end | Flutter 3.22+ & Dart |
| Back-end | Firebase (Auth, Firestore, Cloud Functions, Storage, Analytics) |
| الدفع | تكامل بوابات محلية + عقد ذكي Smart Quartez |
| إدارة | لوحة تحكم Web (لاحقًا React أو Flutter Web) |
| DevOps | GitHub Actions، Firebase CLI |

## 4️⃣ تشغيل المشروع محليًا
```bash
# 1. استنساخ الريبو
git clone https://github.com/alijawdat-cyber/wasat.git
cd wasat

# 2. تثبيت الحزم
flutter pub get

# 3. ربط Firebase (مرة وحدة فقط)
flutterfire configure

# 4. شغّل التطبيق
flutter run
```

> 💡 متطلبات: Flutter SDK, Dart, Firebase CLI, Node 18.

## 5️⃣ هيكل المجلدات (مقترح)
```text
wasat/
├─ lib/
│  ├─ core/        # ثوابت وثيم ولغات
│  ├─ features/    # كل ميزة داخل مجلد مستقل (auth, orders, chat…)
│  └─ ui/          # Widgets مشتركة
├─ functions/      # Cloud Functions (TypeScript)
├─ docs/           # ملفات التوثيق المفصلة
└─ README.md
```

## 6️⃣ التوثيق الإضافي
- docs/architecture.md – بنية النظام
- docs/customer-flow.md – تدفق المستخدم
- docs/payment.md – تفاصيل الدمج مع بوابات الدفع

## 7️⃣ الرخصة
مشروع «وسط» مرخّص تحت رخصة **MIT**. انظر ملف `LICENSE` للمزيد من التفاصيل.
