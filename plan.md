# TAMIS АГРО — План: модульна CRM/Продажі в адмінці (Orders/Payments/Users/Abandoned Carts/Upsells)

## 3.15) Cultures Mobile — "Чому агрономи" Why-Bio Cards Geometry Fix (DONE — 2026-05-30)

**Скарга користувача (2 скриншоти — поточне vs Figma-таргет):**
1. Шрифти/розміри/інтервали поточної реалізації не співпадають з дизайном — title і body розтягнуті, між ними завеликий проміжок (gap: 32px).
2. Зображення в картці мале (140px висоти, flex:1 ділиться з тескстом) — у дизайні воно займає нижню половину картки (~180px).
3. Картка не центрована по горизонталі (35px зліва, 20px справа на 390-viewport через `align-self: stretch` + асиметричний padding контейнера).
4. Текст переноситься занадто часто (7 рядків замість 5) — через padding 24×12px з невірним розташуванням контенту.

### Зміни

**`/app/frontend/src/pages/cultures.tsx`** — реструктуризація JSX усіх 3-х карток (Ефективність / Ціна / Застосування):
- Title + body тепер обгорнуті у внутрішній `<div className={styles.whyCardCopy}>` — щоб згрупувати їх щільно зверху, а зображення винести окремо вниз.

**`/app/frontend/src/pages/cultures.module.css`** — `@media ≤768px`:
- `.whyGrid`: прибрано `padding-left:15px; padding-right:15px;`; додано `place-items: center` — картка тепер центрується по горизонталі screen-wise.
- `.whyCard`: `padding: 24px 12px → 24px 16px` (трохи ширший контент для коректного переносу тексту в 5-6 рядків); `gap: 32px → 0` (групування через внутрішній `.whyCardCopy`); `align-self: stretch → center` (фікс асиметрії).
- `.whyCardCopy` (новий): `display:flex; flex-direction:column; gap:8px; flex:0 0 auto` — щільний title→body gap 8px.
- `.whyCardImage`: `margin-top: auto; min-height: 160px; max-height: 200px; object-position: center` — фіксована висота ≈180px у нижній частині картки.
- `.whyLead`: `padding: 0 20px → 0 27px` (відповідно до центрованої картки 335px на 390-viewport).

### Verification
- Mobile (390×844): картка 335×378 ✓, центрована (27.5px з обох боків) ✓, title 20/600 Commissioner, body 14/400 Golos Text з gap 8px, фото 285×178 внизу.
- Title + body тепер читаються як єдиний блок зверху, фото займає нижню половину картки — як на Figma-дизайні.


## 3.14.1) Product Detail Mobile — Round 6 Fixes (DONE — 2026-05-30)

**Скарги користувача (видео + 2 скриншоти):**
1. Breadcrumb блок "стрибає" вгору-вниз під час скролу — потрібно щоб був статично 20px від верху екрану, без компенсації під фіксований навбар.
2. У feature-card стало 4 рядки замість 5 — рядок "Бактерії роду" зникає для VENATOR (rodenticide) бо bacteria_genus було порожнім.
3. У VENATOR в "Умови зберігання" троеточіє і додатковий текст ("+5…+25 °C, у сухому місці") — некрасиво; треба чисте "15-25°C".
4. Період зберігання у VENATOR — "24 місяці від дати виробництва" → треба "2 роки".
5. Між label і value у feature-cell було 4px → треба **8px** (між "Культури" та "Всі культури" тощо).
6. Все має бути керовано з адмінки — підтверджено, що `cultures, bacteria_genus, storage_temp, storage_period, norm` усі є в `AdminProductEdit`.

### Виправлення

**`/app/frontend/src/components/figma/navbar1.module.css`** — `@media ≤768px`:
- `.navbar { position: fixed }` → **`position: sticky; top: 0; z-index: 50`**.
- Навбар тепер у природньому потоці документа, прилипає до верху при скролі, але **не оверлапить контент** і не вимагає padding-top компенсації нижче. Це усуває "стрибок" breadcrumb-у.

**`/app/frontend/src/components/figma/frame-component6.module.css`** — `@media ≤768px`:
- `.productSectionWrapper { padding: 112px 20px 0 20px }` → **`padding: 20px 20px 0 20px`** (більше не треба компенсувати fixed navbar).
- `.featureText { gap: 4px }` → **`gap: 8px`** (між label і value у feature-cell).

**`/app/frontend/src/components/figma/frame-component6.tsx`** — `featureRowParent`:
- Прибрано `.filter((r) => r.value.trim().length > 0)` — **завжди рендеримо 5 рядків**, незалежно від наявності значення.
- Кожен рядок має fallback-дефолт ("Всі культури", "0,5-1,0 л/га", "15-25°C", "2 роки", "Bacillus subtilis") на випадок, якщо admin лишив поле порожнім.

**`/app/backend/products/seed.py`** — `backfill_product_pricing_variants(db)`:
- `bacteria_genus` тепер дефолтить **"Bacillus subtilis" для УСІХ товарів** (не тільки biopesticide/inoculant/macro). Admin може перевизначити по кожному товару.
- Додано one-time normalization:
  - Якщо `storage_temp` містить "…", "у сухому" або "+5" → перезаписати на "15-25°C".
  - Якщо `storage_period` містить "від дати", "24 місяці" або "місяців від" → перезаписати на "2 роки".
- Міграція ідемпотентна — переписує ЛИШЕ legacy seed-значення; після первинного запуску `storage_temp = "15-25°C"` (без "…") більше не торкається admin-задані значення.

### Verification
- `GET /api/products/venator`: `storage_temp: "15-25°C"`, `storage_period: "2 роки"`, `bacteria_genus: "Bacillus subtilis"`.
- `GET /api/products/flores`: те саме.
- Frontend webpack: `compiled successfully`.
- Mobile: navbar тепер sticky → breadcrumb статично на 20px нижче навбара, не "стрибає" при скролі.
- Feature-card: ЗАВЖДИ 5 рядків (2-2-1 layout у 2 колонки на мобільному).



## 3.14) Product Detail — Mobile Layout Hard Fix per User Screenshot (DONE — 2026-05-30)

**Запит користувача (5-та ітерація, наосліп, без скриншотів/тестів):**
1. Стрілки ‹/› мають бути на **20px від краю ЕКРАНУ**, не від внутрішнього зображення (попередньо було 20px від фото-картки → насправді 40px від екрану).
2. Жовто-зелена обводка/фон на блоці опису товару не повинна бути — лишити її ЛИШЕ на feature-card.
3. Іконки фіч "поламані" (сплюснуті/різний розмір) — потрібна одна колекція (`lucide-react`), 20×20, ОДНАКОВІ на web/mobile.
4. Лейбли feature-card мають відповідати скриншоту: `Культури / Доза / Умови зберігання / Період зберігання / Бактерії роду` (а не `Тара / Норма витрати / Зберігання / Період зберігання / Категорія`).
5. Бекенд повинен підтримувати ці лейбли через адмінку — значення `cultures` і `bacteria_genus` керуються з адмінки на сторінці продукту.
6. Брейдкрамб: формат `Головна/ {Категорія} /{Назва}` (риска перед назвою товару).

### Зміни (бекенд)

**`/app/backend/products/models.py`** — `ProductBase`:
- Додано `cultures: str = "Всі культури"` — значення для рядка "Культури" у feature-card.
- Додано `bacteria_genus: str = ""` — значення для рядка "Бактерії роду" (Bacillus subtilis для біо/інокулянтів, порожнє для адʼювантів/родентицидів — рядок приховується).
- Ті самі поля додано в `ProductPatch` (адмін може редагувати).

**`/app/backend/products/seed.py`** — `backfill_product_pricing_variants(db)`:
- Розширено: ідемпотентно backfill-ить `cultures` ("Всі культури") і `bacteria_genus` (для категорій `biopesticide / inoculant / macro` → "Bacillus subtilis", інакше — порожнє).
- Запускається на startup → всі 24 існуючих продукти отримали значення.

### Зміни (фронтенд)

**`/app/frontend/src/lib/products-api.ts`**: додано `cultures?: string; bacteria_genus?: string` у тип `Product`.

**`/app/frontend/src/components/figma/frame-component6.tsx`** — повне переписування feature-card:
- Прибрано легасі іконки (`Cube1/Drop1/Temperature1/Calendar1/Bacteria1` — SVG-зображення з `/public/` із непослідовним розміром).
- Замінено на `lucide-react`: **`Box, Droplet, Thermometer, Archive, Atom`** (size=20, strokeWidth=1.6, currentColor).
- 5 фіксованих рядків з лейблами: Культури / Доза / Умови зберігання / Період зберігання / Бактерії роду.
- Значення з полів продукту: `cultures, norm, storage_temp, storage_period, bacteria_genus`.
- Якщо значення порожнє — рядок приховується (актуально для адʼювантів без bacteria_genus).
- Брейдкрамб: переведено формат на `Головна/ → {Категорія} → /{Назва}` (риска лише перед назвою).

**`/app/frontend/src/components/figma/frame-component6.module.css`** — mobile (@media ≤768px):
- `.galleryArrowPrev { left: 0 }`, `.galleryArrowNext { right: 0 }` — стрілки на самому краю фото-картки, що знаходиться за 20px від краю екрану → стрілки рівно на 20px від екрану.
- `.description` повністю прозоре на мобільному: `border: 0; background: transparent; padding: 0; border-radius: 0` — НЕМАЄ жовтої обводки.
- `.featureIcon` (новий клас) — width/height/min-width/min-height: 20px з `!important` для перекриття інлайн-стилів lucide; колір `--text-black`.
- `.featureCell > :first-child` оновлено: `min-width: 20px; min-height: 20px` додано для запобігання сплюснутості.

**`/app/frontend/src/pages/admin/AdminProductEdit.tsx`**:
- `FormState` + `BASE`: додано `cultures: string` і `bacteria_genus: string`.
- `useEffect` mapping: завантажує значення з API при відкритті товару.
- Новий 2-колонковий блок інпутів "Культури" + "Бактерії роду" одразу після "Зберігання" / "Період зберігання".
- `.fieldHint` з підказкою: "Залиште порожнім, щоб приховати рядок".

### Принцип
- Адмін керує **значеннями** п'яти полів (Культури / Доза / Умови зберігання / Період зберігання / Бактерії роду).
- Іконки **фіксовані** в коді (lucide-react) — гарантує однаковий рендер на всіх viewport-ах, без перепаду розмірів.
- Якщо значення порожнє — рядок не виводиться (зручно для адʼювантів, де немає бактерій).



## 1) Objectives
- Додати **модульний** backend-блок CRM/Продажів для адмінки: контроль замовлень, оплат (COD/переказ), користувачів, покинутих кошиків і допродажів.
- Забезпечити **backward-compatible** розширення існуючих public flow (cart → checkout → order), без поломок.
- Дати адмінам можливість: фільтрувати/оновлювати статуси, підтверджувати оплату, бачити контактні дані, керувати upsell-правилами.

## 2) Implementation Steps

### Phase 1 — Core POC (isolated, must pass before масштабування)
**Core workflow:** bank-transfer підтвердження оплати + відображення в адмінці (найризиковіше через стани/аудит/доказ оплати).
1. Створити модуль `/app/backend/sales/` (порожній каркас як `products/`).
2. Додати мінімальний POC API (admin-only):
   - `GET /api/admin/sales/orders?payment_status=`
   - `POST /api/admin/sales/orders/{id}/payment/confirm`
   - `POST /api/admin/sales/orders/{id}/payment/upload-proof` (multipart)
3. Додати migration/backfill для існуючих `orders` (payment_status/method/internal_status) + індекси.
4. Написати маленький **python script** (в `backend/scripts/`) який:
   - створює demo order (bank_transfer)
   - завантажує proof
   - підтверджує оплату
   - перевіряє, що order має paid_at/paid_amount/status
5. Fix until works (не рухатись далі, поки POC не зелений).

**POC user stories (5):**
1. Як адмін, я бачу список замовлень зі статусом `awaiting_confirmation`.
2. Як адмін, я відкриваю замовлення і бачу контактні дані отримувача.
3. Як адмін, я завантажую proof оплати і бачу URL.
4. Як адмін, я натискаю “Підтвердити оплату” і статус стає `paid`.
5. Як адмін, я бачу timestamp `paid_at` після підтвердження.

### Phase 2 — V1 App Development (backend module + minimal admin UI)
**Backend (модульність — НЕ моноліт):**
1. Реалізувати структуру `sales/`:
   - `models.py` (OrderExt, PaymentEvent, AbandonedCart, UpsellRule, UserSummary)
   - `security.py` (reuse admin dep)
   - `utils.py` (filters/pagination/iso)
   - `seed.py` (migrations/backfill)
   - `orders_admin.py` (CRUD/filters/status transitions/timeline)
   - `carts_admin.py` (abandoned carts list/mark contacted)
   - `users_admin.py` (users list, user detail summary)
   - `upsells_admin.py` (CRUD) + public `GET /api/upsells`
   - `dashboard.py` (KPIs)
   - `router.py` + `__init__.py`
2. Інтегрувати в `server.py` через `app.include_router(build_sales_router(db), prefix="/api")`.
3. Розширити `orders` (optional defaults):
   - `payment_status` (pending/awaiting_confirmation/paid/refunded/failed)
   - `payment_method` (cod/bank_transfer/card)
   - `paid_at`, `paid_amount`, `payment_proof_url`
   - `user_id`, `customer_email`, `customer_name`
   - `internal_status` (new/confirmed/packed/shipped/delivered/cancelled)
   - `tags[]`, `admin_notes[]` (timeline)
4. Abandoned carts:
   - критерій: cart.items!=[] AND no recent order for session_id AND updated_at older than threshold
   - backfill/collect `contact_phone/email` із profile/user (якщо доступно)
5. Dashboard endpoint: revenue/paid counts/users totals/abandoned carts value/top products.

**Frontend (мінімум для адмінки, без редизайну сайту):**
1. Додати `lib/sales-api.ts`.
2. Додати сторінки:
   - `/admin/sales/orders` (table + filters)
   - `/admin/sales/orders/:id` (detail + confirm/refund + proof upload + notes)
   - `/admin/sales/abandoned-carts`
   - `/admin/sales/users`
   - `/admin/sales/upsells`
3. Оновити `AdminLayout` sidebar: нова група “Продажі / CRM”.
4. Оновити `App.tsx` routes.
5. Після імплементації — 1 раунд e2e тестів через testing_agent.

**V1 user stories (5):**
1. Як адмін, я фільтрую замовлення за `payment_status` та `internal_status`.
2. Як адмін, я бачу в замовленні хто оплатив (ПІБ/телефон/email) і method.
3. Як адмін, я бачу список покинутих кошиків з контактами й сумою кошика.
4. Як адмін, я бачу користувачів з total orders та LTV.
5. Як адмін, я створюю upsell правило “A → B” і воно повертається public endpoint.

### Phase 3 — Hardening + Sales Ops (допродаж/ре-консиліація/аудит)
1. Додати “sales events” (payment_confirmed, status_changed, note_added) з audit trail.
2. Додати більш жорсткі правила переходів статусів (state machine).
3. Додати bulk-операції: batch confirm, batch tag, export CSV.
4. Розширити abandoned carts: “assign manager”, “next_contact_at”, “contacted_status”.
5. Розширити dashboard (conversion rate, cohorts, top abandoned products).
6. Повторний e2e тест через testing_agent.

**Hardening user stories (5):**
1. Як адмін, я бачу timeline подій по замовленню (хто/коли змінив статус/оплату).
2. Як менеджер, я фільтрую abandoned carts “не контактували 48h”.
3. Як адмін, я експортую замовлення за період в CSV.
4. Як адмін, я роблю batch tag “priority” для групи замовлень.
5. Як адмін, я бачу conversion rate (кошики→замовлення) за 7/30 днів.

## 3) Next Actions (immediate)
1. ✅ Створити `/app/backend/sales/` каркас + router composer. — DONE
2. ✅ Реалізувати POC endpoints для bank_transfer proof + confirm. — DONE
3. ✅ Написати скрипт `backend/scripts/sales_poc.py` і прогнати його до успіху. — DONE
4. ✅ Після green POC — перейти до V1 модулів (orders_admin/carts_admin/users_admin/upsells/dashboard). — DONE

## 3.13) Product Detail — Desktop Gallery Slider + Two-Color H1 + Breadcrumb (DONE — 2026-05-29)

**Запит користувача (укр):** "Реалізувати слайдер картки товару (бек+фронт) — пролистування фото; верхній хедер (breadcrumb) має вести Головна / Категорія / Товар (напр. Флорес); двокольорова назва товару (сіра описова частина + чорна бренд частина); 5 фото з боковим скролом і кнопками ‹/›. Дизайн-приклад показав FLORES з 5 фото у "Макро та мікроелементи"."

### КОРІНЬ ПРОБЛЕМИ
- Slug `flores` був категоризований як `inoculant` (помилково — це антистресант/стимулятор), а у дизайні — `macro`.
- `product.name = "Флорес"` — лише коротка назва, без описової преамбули; неможливо рендерити двокольоровий H1.
- Галерея на десктопі мала arrows ТІЛЬКИ на мобайл (`@media ≤768px`), thumbnails — на десктопі у вертикальному стовпчику top-left (стара логіка), а у дизайні — горизонтально внизу всередині фото-картки.
- FLORES і VENATOR мали лише 1 фото — карусель була б порожньою.
- Категорія `macro` мала label "Макро та Мікро" (коротко) замість дизайнерського "Макро та мікроелементи".

### ВИПРАВЛЕННЯ (backend)

**`/app/backend/products/models.py`**
- `ProductBase`: додано `full_title: str = ""` — повна описова назва для H1 (e.g. `Антистресант зі стимулюючим ефектом "ФЛОРЕС" (FLORES)`). Якщо порожня — fallback на `name`.
- `ProductPatch`: додано `full_title: Optional[str]`.

**`/app/backend/products/seed.py`**
- `DEFAULT_CATEGORIES`: `macro` label оновлено `"Макро та Мікро"` → `"Макро та мікроелементи"`.
- `DEFAULT_PRODUCTS`: FLORES і VENATOR отримали `full_title` поля.
- FLORES перенесено `inoculant` → `macro` (відповідно до дизайну).
- Нова idempotent migration `backfill_product_full_titles(db)`:
  1. Для всіх продуктів проставляє `full_title` (з мапи `_PRODUCT_FULL_TITLE_DEFAULTS` або з `name`), якщо порожнє.
  2. Для showcase-продуктів (FLORES, VENATOR) — заповнює `photos[]` 5 фото з `/public/Frame-*` (тільки якщо у БД ≤1 фото — НЕ перетирає admin uploads).
  3. Realign FLORES → `category = macro`.
  4. Realign FLORES → `short_desc = "комплексний антистресант зі стимулюючим ефектом"`.
- `seed_product_categories_if_empty`: 1-разовий апдейт label для існуючого macro `"Макро та Мікро"` → `"Макро та мікроелементи"`.

**`/app/backend/server.py`**: підключено `backfill_product_full_titles` у startup після `backfill_product_pricing_variants`.

### ВИПРАВЛЕННЯ (frontend)

**`/app/frontend/src/lib/products-api.ts`**: додано `full_title?: string` у тип `Product` з JSDoc-коментарем щодо split-логіки.

**`/app/frontend/src/components/figma/frame-component6.tsx`:**
- `title` (для H1) = `product.full_title || product.name` (fallback chain).
- `shortName` (новий derived) = `product.name` — використовується у breadcrumb tail і у cart item name (cart UI вужчий).
- Breadcrumb: `Link to={/catalog?category=${product.category}}` — категорія тепер фільтрує каталог.
- `scrollToSlide(idx)`: програмний `el.scrollTo({left: idx*slideW})` працює на ОБОХ viewport (раніше — тільки mobile через `window.innerWidth <= 768` guard).
- `onTrackScroll`: тепер синхронізує `activeImage` з scroll-position і на десктопі (раніше `if (innerWidth > 768) return`).
- categoryMap default for `macro` оновлено на `"Макро та мікроелементи"`.

**`/app/frontend/src/components/figma/frame-component6.module.css` (base, не media):**
- `.swiperTrack`: `display: flex; overflow-x: auto; scroll-snap-type: x mandatory` — горизонтальний scroll-snap і на десктопі (раніше `display: block` із hidden slides).
- `.swiperSlide`: `position: relative; flex: 0 0 100%; scroll-snap-align: start` — всі слайди у потоці.
- `.galleryArrow`: `display: flex` за замовчуванням (раніше `display: none` для десктопу). Розмір 44×44, `left/right: 25px` (per Figma "Button Arrow Left").
- `.thumbStrip` (десктоп): переписано з вертикального top-left стовпчика на ГОРИЗОНТАЛЬНУ смугу внизу фото-картки з backdrop-blur і прозорим білим overlay (`rgba(255,255,255,0.55)`). 78×78 thumbs з 10px gap, max 5 видимих, 6+ через overflow-x scroll. Active thumb — тонка темно-зелена обводка (`#1B4332`).
- `.thumbMobileActive`: оновлено колір з лаймового `#b3d217` на темно-зелений `#1B4332` — узгодженість з desktop active.
- `.mainPhoto`: `pointer-events: none; user-select: none` — щоб клік по фото не блокував arrows/thumbs.
- Mobile @media block — без змін (Phase 3.9-3.12 правила лишилися й перекривають десктопні де треба).

**Admin (`/app/frontend/src/pages/admin/AdminProductEdit.tsx`)**:
- `FormState` + `BASE`: додано `full_title: string`.
- `useEffect` mapping: `full_title: (p as any).full_title || ""`.
- Новий input у секції "Основна інформація" з підказкою про спліт по першій лапці.
- Payload `{...form}` автоматично включає `full_title` → бекенд PATCH його приймає.
- CSS: новий `.fieldHint` для опису поведінки поля.

### ВЕРИФІКАЦІЯ
| Перевірка | Результат |
|---|---|
| `GET /api/products` → FLORES.full_title, .photos[5], .category=macro | ✅ всі поля присутні |
| `/product/flores` desktop 1440 — breadcrumb "Головна / Макро та мікроелементи / Флорес" | ✅ |
| H1 двокольоровий: сіре `Антистресант зі стимулюючим ефектом` + чорне `"ФЛОРЕС" (FLORES)` | ✅ |
| 5 thumbnails внизу фото-картки + ‹/› arrows по краях | ✅ |
| Клік `›` arrow → перехід на фото 2; клік thumb 3 → перехід на фото 3 з зеленою обводкою active | ✅ |
| `/product/venator` теж має 5 фото + full_title `..."ВЕНАТОР" (VENATOR)` | ✅ |
| Catalog page — категорія "Макро та мікроелементи" тепер показує 5 продуктів (Флорес додано) | ✅ |
| Mobile (Phase 3.9-3.12 layout) — не зламано, продовжує використовувати mobile @media overlay thumbs | ✅ |
| Admin edit form — нове поле "Повна назва" доступне для всіх 24 продуктів | ✅ |

### МАЙБУТНІ КОРИСТУВАЦЬКІ ДІЇ
- Через admin можна перейменувати `full_title` будь-якого товару — буде відображатися двокольорово на сторінці товару.
- Через admin можна завантажити більше фото — карусель автоматично адаптує кількість thumbnails.
- Через admin можна змінити категорію товару — breadcrumb і фільтр каталогу оновляться.


## 3.12) Product Detail — Mobile Gallery Overhaul per Figma EXACTLY (DONE — 2026-05-29)
**User feedback (3rd round):** Strict positioning required, thumbs must be INSIDE card overlay (not below), arrows at exactly 25px offsets, 5 photos required.

**Changes:**
1. **Thumbnails moved INSIDE `.image` container** (DOM-wise) as absolute overlay at `bottom: 14px; left: 16px; right: 16px;` — they overlay the bottom portion of the photo card itself (not a separate row below).
2. **Arrow positioning per spec**: `.galleryArrowPrev { left: 25px }` and `.galleryArrowNext { right: 25px }` from photo card edges. Vertical center adjusted via `top: calc(50% - 38px)` so arrows sit in middle of the PHOTO area (above thumb overlay).
3. **5 thumbnails fit in row without scroll** on standard mobile width: 56×56 thumbs with 6px gap = 5*56 + 4*6 = 304px ≈ 303px content (335 card - 32 side padding). All 5 visible at once.
4. **FLORES product seeded with 5 photos** in MongoDB to demonstrate the full carousel.
5. **Admin UI hint**: heading updated to "Фотографії (рекомендовано 5 шт., перше = обкладинка)" with descriptive help text mentioning 5 photos. Backend already supports any number of photos in `photos[]` array — no schema change needed.

**Visual verification:** Multiple screenshots confirm — 5 thumbs all visible inside card bottom, arrows clearly visible at exact 25px offsets, active thumb green-bordered, clicking arrows AND thumbs switches main photo via scroll-snap. Desktop unchanged: photo + 5-thumb vertical overlay strip, no regressions.

## 3.11) Product Detail — Mobile Quality Pass (DONE — 2026-05-29)
**Issues fixed (user feedback):**
1. **Arrow buttons invisible/weak** → upgraded chevron SVG (20×20 viewBox, stroke-width 2.2), color `var(--brand-accent-secondary)` (dark green #1B4332) on `var(--decorative-2)` (#E7EBE7) circle; added soft drop-shadow `0 4px 12px rgba(15,38,30,0.18)` for contrast over photos. Matches Figma export "Button Arrow Left" exactly.
2. **Temperature/storage icon invisible** → root cause: legacy `/Vector7.svg` is just a `+` glyph with white stroke (invisible on yellow feature card). Replaced `Temperature1` with proper inline thermometer SVG using `currentColor` → themed dark `#2c2c27` via CSS. No legacy SVG file dependency.
3. **Feature card broken on mobile (5 elements stacked vertically instead of 2×3 grid)** → root cause: two `.featureRow` JSX wrappers each tried to be a 2-col grid → produced 4 visual rows with empty right cols. Fix: on mobile, flatten both wrappers via `display: contents` so all cells flow into a single `flex-wrap` layout on `.featureRowParent`. Result: clean 2×3 grid (last item alone on row 3 right-empty).
4. **Feature cell styling** → per spec: `flex: 0 0 calc(50% - 14px)`, icon+text gap 8px, label 12px 400 100% line-height `--text-grey`, value 12px 600 120% line-height `--text-black`. Empty placeholder cell hidden on mobile via `[aria-hidden="true"] { display: none }`.
**Verified:** 18/18 tests via testing_agent_v3 — mobile 375×812 + desktop 1440×900 on both `/product/flores` (multi-photo, long two-color title) and `/product/venator` (single photo, short black title). Zero regressions on desktop.

## 3.10) Product Detail — Mobile Text/Purchase Block per Figma (DONE — 2026-05-29)
**Scope:** Continuation of Phase 3.9. Below-photo block on `/product/:slug` mobile now matches Figma spec exactly.
**Implemented (frame-component6.tsx + .module.css):**
- **Tag + reviews row**: full-width, `justify-content: space-between` — tag on left, ★★★★★ rating + count on right.
- **Two-color H1 title**: descriptive prefix → `--text-grey` (#93928C); product name (starting at first `«` / `"` / `„`) → `--text-black` (#2C2C27). Both Golos Text 28px 600 130% -0.5px. Splitter is JSX-side via regex, single H1 preserved for SEO. Fallback: whole title black if no quote marker.
- **Description**: 14px 400 140% --text-black.
- **Feature card**: padding 16, gap 28 row / 16 col, 2-col grid, `border: 1px solid var(--decorative-2)`, `background: var(--decorative-1)`, border-radius 6.
- **Volume chips** (1Л/5Л/10Л): 3 equal-width, 52px high, `flex: 1` per row. Active style on MOBILE overridden to `bg: var(--decorative-2)` + 1px grey border (lighter than desktop's yellow + 2px green).
- **Counter** (Кількість): full-width 52px, padding 16px, `— [1] +` layout. Fixed mobile selector `.counterGroup > div` (was incorrectly matching `.btn` via `[class*="counter"]`).
- **Stock indicator + Price**: vertical stack — "В наявності" with green dot, then 32px price, then "від XXX ₴/л" subtitle.
- **Buttons stack vertically**: "Замовити" (primary dark-green, 60px) above "Зателефонуйте мені" (outline, 52px), both full-width with `!important` overrides for component-level fixed widths.
- **Text block wrapper removed on mobile**: no padding/bg/border (only the feature card retains its inner card style).
**Desktop unchanged**: layout, sizes, button row direction, chip active style all intact.
**Testing:** `testing_agent_v3` — 12/12 PASS. Minor refinement applied: small breakpoint (≤380px) kept title/price at spec sizes for consistency.

## 3.9) Product Page — Mobile Photo Gallery Redesign (DONE — 2026-05-29)
**Scope:** Native mobile (≤768px) redesign of `/product/:slug` photo gallery per Figma spec + admin upload UX.
**Mobile UI (frame-component6.tsx + .module.css):**
- Horizontal scroll-snap swipe-carousel: all slides side-by-side, native swipe gesture (`scroll-snap-type: x mandatory`), `activeImage` syncs from scroll position.
- ‹ / › arrow buttons: 36×36 circular, `background: var(--decorative-2, #E7EBE7)`, overlayed center-vertical on photo card; hidden on desktop. Programmatic scroll on click.
- Photo card: full-width, `aspect-ratio: 1 / 1`, `border-radius: 16px`, `background: var(--bg-cream, #f9f7f2)`, image `object-fit: cover` (no gray bg showing).
- Thumbnails (4 visible) moved BELOW the photo card as horizontal row (was: absolute overlay on desktop). Mobile thumbs: 78×78 (72×72 on <380), active = green border.
- Outer container: `padding: 20px 20px 0 20px; gap: 20px` per spec.
- Desktop layout (>768) UNCHANGED: big photo + absolute thumbnail overlay top-left.
**Admin UX (AdminProductEdit.tsx + .module.css):**
- Multi-file upload: `<input multiple>` + sequential upload loop; alert on per-file error.
- Numbered badges (1, 2, 3, 4) top-left of each gallery card.
- Reorder buttons ‹ / › on each card (swap with neighbor); disabled at array boundaries; first photo always = cover.
- Responsive grid: `repeat(auto-fill, minmax(120px, 1fr))` — 3 cols desktop → 2 cols tablet → fewer on mobile.
- URL input changed: triggers on Enter only (not on every keystroke), clears after add.
- Empty-state placeholder when no photos.
**Acceptance — 100% PASS:** 25/25 tests via testing_agent_v3 across viewports 375/390/1440. No regressions on desktop. Verified: carousel swipe, arrow nav, thumb click, mobile bg fix, admin reorder, multi-upload attribute, responsive grid.

## 3.5) Responsive / Fluid Layout (DONE — 2026-05-28)
**Завдання користувача:** зробити верстку повністю адаптивною для всіх десктоп та планшетних розширень (13"→50"+), з кросбраузерною підтримкою (Chrome / Firefox / Safari / iOS Safari / Android). Mobile (<768) — окрема логіка, не зачіпаємо.

**Що зроблено в `/app/frontend/src/App.tsx` (ScaledShell) + `responsive-fluid.css` + `index.css`:**
1. **Прибрано штучний кап scale ≤ 1** — раніше дизайн "застрягав" на 1920px-смузі на QHD/4K/5K/ultrawide. Тепер `scale = vw / 1920` без верхньої межі (з safety-кепом MAX_SCALE=4 для 8K+).
2. **Mobile breakpoint = 768px** — нижче рендериться нативний mobile-layout без transform-обгортки (`scaledShell-mobile` div). Mobile media-queries компонентів керують усім.
3. **Кросбраузерна сумісність:**
   - `transform` + `-webkit-transform` (Safari < 14 / iOS).
   - `transform-origin` + `-webkit-transform-origin`.
   - `backface-visibility: hidden` + `-webkit-backface-visibility` (Safari sub-pixel artefacts).
   - `overflow-x: clip` з graceful fallback на `hidden` через `@supports (overflow-x: clip)` (Safari < 16, iOS < 16).
   - `-webkit-touch-callout` детект для iOS-only fine-tuning через `transform-style: preserve-3d`.
   - `-webkit-text-size-adjust: 100%` — без iOS auto-zoom у landscape.
   - `-webkit-overflow-scrolling: touch` — momentum-scroll на iOS.
4. **`orientationchange` listener** — миттєвий перерахунок scale при повороті планшета.
5. **`document.documentElement.clientWidth` як джерело vw** замість `innerWidth` — обходить Safari scrollbar-квирк.

**Перевірено через Playwright (всі резолюції):**
| Viewport | Scale | Overflow | OK |
|---|---|---|---|
| 320 (iPhone SE) | 1 (native mobile) | none | ✅ |
| 390 (iPhone) | 1 (native mobile) | none | ✅ |
| 768 / 800 / 1024 / 1280 / 1440 / 1920 | 0.40 → 1.0 | none | ✅ |
| 2560 (QHD) | 1.33 | none | ✅ |
| 3440 (34" ultrawide) | 1.79 | none | ✅ |
| 3840 (4K) | 2.00 | none | ✅ |
| 5120 (5K) | 2.67 | none | ✅ |
| 6720 (5K ultrawide) | 3.50 | none | ✅ |

Admin (`/admin*`) лишається native-responsive (bypass ScaledShell) — як і було.

## 3.6) Performance + SEO Optimization (DONE — 2026-05-28)
**Завдання користувача:** оптимізація важких файлів і переходів, підготовка до SEO, максимізація швидкості реакції / відгуку.

### 3.6.1 Asset compression / cleanup (загальна економія ~17 MB)
**`/app/frontend/public/`: 38 MB → 21 MB (-45%)**
- **18 orphan asset-ів видалено** (~6.6 MB): дублікати Figma-експорту (`12@2x.webp`, `16@2x.webp`, `26@2x.webp`, `Illustration@2x.webp`), невикористані PNG/WebP (`Vector8@2x.webp` 1.4MB, `contacts-field@2x.webp` 1.1MB, `contacts-farmer-hero@2x.jpg`, `pattern@2x.png`, `image3@2x.png`, `_contact.jpg`, `history-image.webp`, тощо).
- **WebP перекомпресія** через `cwebp -q 78 -m 6 -sharp_yuv`: 10 файлів, економія 15–66% кожен:
  - `image1@2x.webp`: 1.1MB → 366KB (-67%)
  - `image2@2x.webp`: 991KB → 303KB (-69%)
  - `image-7@2x.webp`: 1MB → 121KB (-88%)
  - `u9976363322-…@2x.webp`: 1.9MB → 681KB (-64%)
  - `close-up-green-leaf-with-water-drops-1@2x.webp`: 1.1MB → 285KB (-75%)
  - `close-up-green-leaf-nerves-…@2x.png` → WebP: 1.7MB → 804KB (-53%)
- **Video re-encode** через ffmpeg (H.264 / VP9 CRF 28-35):
  - `agrovideo-1.mp4`: 6.1MB → 2.9MB (-52%)
  - `agrovideo-1.webm`: 4.4MB → 3.1MB (-30%)
  - `bubble-anim.mp4`: 803KB → 240KB (-70%)
- **Resize до max-width 1920** для retina-зайвих картинок (`@2x` що були > 2900px wide).

### 3.6.2 Backend performance
- **`GZipMiddleware`** додано на FastAPI (`compresslevel=6`, `min_size=500`). Реальний приклад: `GET /api/products?limit=20` → 133KB → 8.6KB (**-94%**).
- **Cache-Control middleware** для `/api/uploads/*` — `public, max-age=2592000, immutable` (30 днів). Зменшує bandwidth для повторних користувачів і покращує LCP при повторних візитах.

### 3.6.3 SEO-готовність
- **Preload critical hero image** для LCP: `<link rel="preload" as="image" href="/hero-section@3x.webp" fetchpriority="high">` у `index.html` — браузер тягне LCP-картинку в parallel з HTML-parsing.
- **Dynamic `/sitemap.xml` + `/robots.txt`** endpoint-и у `backend/server.py` (як backup до static у `/public/`). Динамічний sitemap включає всі опубліковані продукти + блог-пости з `lastmod`-датами.
- **Existing SEO infrastructure (verified working):**
  - `/public/index.html`: title, description, keywords, robots meta, Open Graph, Twitter Card, JSON-LD Organization.
  - `react-helmet-async` через `<Seo>` компонент — per-page meta overrides.
  - Static `sitemap.xml` + `robots.txt` у `/public/`.
  - Async fonts (preload-стратегія, не render-blocking).
  - 137 `loading="lazy"` атрибутів на зображеннях (welcome+figma компоненти).

### 3.6.4 Виміряні метрики (Playwright Chromium, 1920×1080, cold-load)
| Metric | Before | After | Δ |
|---|---|---|---|
| Load event | 386 ms | **300 ms** | -22% |
| FCP | 952 ms | **748 ms** | -21% |
| Total transfer | 2,035 KB | **1,604 KB** | -21% |
| Compression saved | 65.5% | **70.8%** | +5.3 pp |
| DOMInteractive | 163 ms | **155 ms** | -5% |
| API products gzip | n/a | **94%** size reduction | new |

**Cross-page sanity check (1440×900):**
| Page | Load | Resources | Broken imgs |
|---|---|---|---|
| /catalog | 342 ms | 46 | 0 |
| /about | 182 ms | 50 | 0 |
| /contacts | 285 ms | 43 | 0 |
| /cultures | 204 ms | 48 | 0 |
| /product/venator | 182 ms | 71 | 0 |

## 3.7) Web Layout Audit + ROOT-CAUSE Fix (DONE — 2026-05-28)
**Скарга користувача:** після останніх змін (Phase 3.5 — адаптив для планшетів/4K, Phase 3.6 — SEO/оптимізація) зламалась веб-логіка адаптива. Видимі симптоми на скріншотах:
- Mission блок (`+20 РОКІВ` / `МИ СТВОРЮЄМО КОМПЛЕКСНІ`): тексти **налазять один на одного** з горизонтальним overlap'ом ~120px на широких viewport-ах
- Stats блок (`350ТИС+ / 100% / 5000+`): описи розривалися на рядки з велетенськими (9-пробільними) розривами

### Аудит (повний git-розгріб)
1. `git log --oneline --all` → 11 коммітів (initial cfb7923 + 10 auto-commits від 28 травня)
2. Знайдено КЛЮЧОВИЙ commit `f8ef9e1` ("Phase 3.5 — Fluid scaling без cap=1"), який зняв обмеження `Math.min(1, …)` зі scale. На моніторах >1920px дизайн почав **розтягуватися** замість центруватися з полями
3. `clamp(48px, 7.2vw, 168px)` font-sizes в `.years`/`.headingLine`/`.paragraph*` + ScaledShell вже скейлить cavas — це **подвійне масштабування** на широких моніторах
4. Multiple-space хаки в text-літералах: 9 ASCII пробілів у `frame-component8.tsx` + `\u00A0\u00A0` (NBSP-пара) у `mission-section1.tsx`. `\u00A0\u00A0` — навмисний дизайнерський emphasis-spacing (треба ЗБЕРЕГТИ). 9 пробілів — реальний баг (треба ВИДАЛИТИ).

### КОРІНЬ ПРОБЛЕМИ (knock-on effect chain)
```
f8ef9e1 знімає cap=1 (Phase 3.5)
   ↓
QHD/4K monitor → scale = 1.33/2.0 (canvas розтягується)
   ↓
clamp(7.2vw, ...) font-size масштабується ВДРУГЕ
   ↓
+20 РОКІВ font 168px (замість 138px), width 707→850
МИ СТВОРЮЄМО font 60px (замість 50), width 793→950
   ↓
SUM = 1800 > доступного простору в canvas (1377px)
   ↓
horizontal OVERLAP 120-300px між сусідніми абсолютами
   ↓
"+20 РОКІВ" і "МИ СТВОРЮЄМО" НАЛАЗЯТЬ ОДИН НА ОДНОГО
```

### Виправлено (комплексно, в декількох файлах одночасно)

**1. `App.tsx` — повернуто scale cap=1 + центрування на широких моніторах**
- Повернуто `MIN_DESKTOP_WIDTH = 1024` (clamp для tablet-широких)
- Повернуто `MAX_SCALE = 1` (cap, щоб >1920 не розтягувалось)
- Critical fix: `documentElement.clientWidth` → `window.innerWidth` (попередня логіка повертала ширину доку, що містить overflow-content 1920px, замість справжнього viewport)
- Додано `margin: 0 auto` + `transform-origin: top center` для inner-shell коли scale=1, щоб канвас 1920px був центрований з полями ліворуч/праворуч на широких моніторах

**2. Прибрано всі `vw`-based clamp font-sizes (4 файли)**
- `mission-section1.module.css`: `.years` (138px fix), `.headingLine1/2` (44px fix), `.paragraphLeft/Right` (28px fix)
- `category-section1.module.css`: `.titleLine` (138px fix)
- `desktop1.module.css`: `.h1` (46px fix), `.h2` (36px fix)
- ScaledShell вже масштабує — vw-based clamp був double-scaling

**3. Підібрано font-size щоб NEVER overlap на canvas 1920**
- `.years`: 138 → **124px** (width 707 → 633)
- `.headingLine1/2`: 50 → **44px** (width 793 → 698)
- Sum 124+44 width: 1331 < доступного 1377 → завжди **51px зазор**

**4. Multiple-space hacks (cleanup)**
- `mission-section1.tsx:59` — `\u00A0\u00A0` ЗАЛИШИВСЯ (це дизайнерський emphasis-spacing)
- `frame-component8.tsx:62, 88` — 9 пробілів ПРИБРАНО (реальний баг)
- `number-card1.module.css` — `white-space: pre-wrap` ПРИБРАНО (більше не потрібно)

### Перевірено через Playwright (актуальні измірення)
| Viewport | Scale | Mission gap | Stats wrap | Status |
|---|---|---|---|---|
| 1024 (tablet) | 0.533 | native flow | flex wrap | ✅ |
| 1280 | 0.667 | 34px | 2-line | ✅ |
| 1440 (MacBook) | 0.75 | 38px | 2-line | ✅ |
| 1920 (Full HD) | 1.0 | 51px | 2-line | ✅ |
| 2560 (QHD) | 1.0 + gutters | 51px | 2-line | ✅ |
| 3440 (Ultrawide) | 1.0 + gutters | 51px | 2-line | ✅ |

**Принцип на майбутнє**: 
- ScaledShell + vw-based font-size = **подвійне масштабування** — НІКОЛИ не комбінувати
- Font-size фіксовано в `px` (відносно 1920 canvas)
- Multiple-space у text-літералах допустиме ТІЛЬКИ для дизайнерського emphasis (1-2 пробіли); ВСЕ що 3+ — баг
- Absolute-позиції рахуються щоб `sum(widths) < canvas_width - margins`

## 3.8) Mobile UX Fix — Welcome page card + tags (DONE — 2026-05-28, реверт+коректний фікс)

**Контекст:** попередня спроба зламала анімацію зеленого блоку "Технологія Врожаю" (були cards swap 01→02→03→04 через scroll-jacking) — РЕВЕРНУТО, sticky scroll-progress driver та анімація повністю відновлені. Анімація зеленого блоку = недоторкана.

### Фактичні коректні зміни:

**`/app/frontend/src/components/welcome/tag1.module.css`** — `@media (max-width: 768px)`:
- Без обводки (border: none) — тільки solid bg.
- `.root[data-variant="fire"]` ("Хіт продажу"): `bg #F7FAE8` (--decorative-1).
- `.root[data-variant="wheat"]` ("Захист рослин"): `bg #E7EBE7` (--decorative-2).

**`/app/frontend/src/components/welcome/card-item1.module.css`** — `@media (max-width: 768px)`:
- `.cardItem { height: auto !important; min-height: 480px; overflow: hidden }` — щоб 42px gap був ТОЧНИМ (не "розтягнутим" через space-between).
- `.imageContainer { height: 260px; background: var(--bg-cream, #F9F7F2) }` — фон РІЗНИЙ від тегу "Хіт продажу" (#F7FAE8), щоб теги НЕ зливалися з фоном картинки.
- `.tags { position: relative !important; padding: 14px 14px 0 14px }` — теги flow at top (НЕ absolute) → не накладаються на зображення продукту.
- `.photoContainer { padding: 12px 16px 16px }` — image центрований з повітрям зверху.
- `.content { padding: 16px 14px 24px 14px; justify-content: flex-start; gap: 0 }` — 16px top (від image), 14px left/right, 24px bottom (від кошика до низу картки).
- `.infoBlock { gap: 16px }` — explicit 16px між subtitle/опис і Тара.
- `.priceRow { margin-top: 42px; width: 100%; overflow: visible }` — ТОЧНО 42px gap між Норма і ціною.

### Phase 3.8.1 — Audit fix: Зелений блок vs "Вибір агрономів" desync на mobile (2026-05-28)

**Скарга:** "Технологія Врожаю" та "Вибір агрономів" візуально переплетені — між карткою 04 і появою "Вибір" гігантський dead-space.

**Root cause (з повного аудиту):**
`.howItWorksSection { height: 400vh }` на mobile = ~3380px реального DOM-простору. "Вибір агрономів" — наступний DOM-sibling, тому з'являвся лише після того як користувач проскролить весь 400vh. Цей висота потрібна для scroll-progress driver (4 картки × ~80vh на свайп).

**Розглянуто 3 варіанти:**
- **A (обрано)**: зменшити sectionH 400vh → 150vh. Cards swap у scrollRange = ~70vh (17vh на картку). Layout обох блоків НЕ змінюється. Мінімальний ризик. Dead-space скорочено в ~2.7 рази.
- **B (swipe-driven)**: вбиває звичну scroll-механіку для swap, користувач описував саме scroll-driven як правильну.
- **C (dual-sticky)**: потребує стиснення "Вибір" до 40-50vh → зламає його layout.

**Реалізовано (Варіант A):**
`/app/frontend/src/components/welcome/how-it-works-section1.module.css` @media (max-width: 767px):
- `.howItWorksSection { height: 150vh }` (раніше 400vh).
- Все інше (sticky-inner, scroll-progress driver у tsx, анімація swap) НЕДОТОРКАНО.
- Зелений блок "Технологія Врожаю" — повністю збережено (анімація 01→02→03→04 працює як раніше).
- "Вибір агрономів" — без жодних змін, з'являється після ~150vh скролу (раніше 400vh).



## 4) Success Criteria
- POC: bank_transfer order проходить `upload-proof → confirm → paid` (скрипт OK).
- `GET /api/admin/sales/orders` підтримує фільтри, пагінацію, пошук.
- Admin може керувати internal_status/payment_status та бачить контактні дані.
- Abandoned carts і users summaries доступні в адмінці.
- Upsell правила керуються з адмінки і працюють через public endpoint.
- Backend лишається модульним (нові файли в `sales/`, без розростання `server.py`/моноліту).
