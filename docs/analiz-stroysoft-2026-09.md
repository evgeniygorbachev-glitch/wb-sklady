# Аналитика строительного софта как основы для своего модуля

Дата: 18.09.2026. Контекст: дашборд «Склады Wildberries и Ozon» (`index.html`) уже содержит объекты
с координатами, статусом, площадью и кластером (`D`, `OZ`) и вкладку повреждённых складов с датами
событий, собственником, арендой и выплатами (`DMG`). Задача: выбрать, на чьи модели данных и
паттерны опираться при написании собственного блока.

## 0. Как прочитаны требования

| # | Формулировка | Индустриальный термин |
|---|---|---|
| 1 | Объекты и затраты в одном аналитическом блоке | Job costing / план-факт по объекту, cost codes |
| 2 | Фото из мессенджера, привязка к объекту по дате и изменениям | Photo documentation + change events, бот-инжест |
| 3 | «Вин учет» проектов с разрезами по регионам, размерам и т. д. | Прочитано как **единый** реестр проектов с атрибутами сопоставимости. Если имелся в виду **версионный** учёт: см. 2.3b |
| 4 | Сколько стоила прошлая похожая работа против новой | Cost benchmarking с нормализацией по времени и региону, объекты-аналоги |

## 1. Сводная матрица

Легенда: ●● сильно закрывает, ● частично, — нет. «Что брать» = конкретный паттерн или модель данных.

| Продукт | 1 Объекты+затраты | 2 Фото по дате/изменениям | 3 Реестр с разрезами | 4 Бенчмарк «было/стало» | Что брать |
|---|---|---|---|---|---|
| Procore | ●● | ● (Photos: альбом, Location, Trade, дата → Daily Log) | ●● (multi-tier Locations, WBS custom segments) | ● (только история бюджета) | WBS: cost code × cost type × сегменты; change event → change order; `origin_id` для внешних документов |
| Autodesk Build (Forma) | ●● | ● | ● | ● (forecast по истории) | Цепочка PCO → RFQ → SCO/OCO; budget transfers, связанные с CO; вложения из ответных писем автосохраняются |
| JobTread | ●● | ● (фото в daily logs, лента) | ● | ● (каталог cost items, история) | Budget-first: каждый документ привязан к строке бюджета 1:1; variance на любом уровне |
| Buildertrend | ●● | ● (только через приложение) | ● | ● | Cost codes сохраняют историю при деактивации; variance codes для незапланированных затрат; projected cost = max из источников |
| Fieldwire | — | ●● (фото = attachments/bubbles к задачам с x/y на плане, GPS сохраняется) | ● | — | Floorplan → Sheets (версии) + Sheet Compare; webhooks по сущностям; OpenAPI и llms.txt |
| PlanRadar | — | ●● (тикет с фото на плане) | ● | — | Webhook «photo attached», export тикетов; лимит 30 req/min |
| CompanyCam | — | ●● (лента по проекту, timeline, before/after, GPS+time stamp) | ● | — | Модель Photo: `captured_at`, `coordinates`, `tags`, `description`, `internal`; webhooks с backoff-ретраями |
| OpenSpace / HoloBuilder / Gmini | — | ●● (360°, TimeTravel: одна точка в разные даты) | — | — | Идея «сравнение по датам в одной точке»; сама 360-съёмка для складов избыточна |
| Gectaro (RU) | ●● (смета ↔ график ↔ закупки ↔ финансы) | ● (Telegram-бот прораба: выбор проекта → прогресс) | ● | ● | Схема бота: список проектов → ближайшие работы → факт; Open API (документация по запросу), 400+ интеграций через Albato |
| Pragmacore (RU) | ●● (БДР/БДДС, план-факт) | ● | ● | ●● (Оферта: предрасчёт стоимости; предиктивная аналитика на 300–600 проектах) | Идея сценариев (пессимистичный/оптимальный/оптимистичный); публичного API нет |
| Adept (RU) | ●● | ● (Стройконтроль: замечание + фото + место на чертеже) | ● | ● | Интеграции с Primavera/MS Project/Гранд-Смета; Р.3 общего журнала работ с телефона |
| 1С:Подрядчик строительства | ●● (затраты по стройкам/объектам/статьям, КС-2/3/6а, М-29) | — | ● | ● (план-факт, эффективность по объектам) | Разрезы затрат: стройка → объект → статья → технологическая структура → подрядчик |
| Техзор / MRS Стройконтроль / OSMO | — | ●● (замечания с фото, гео, шахматка) | ● | — | Public API у Техзора; экспорт замечаний с фото в docx/xlsx/pdf |
| Стройбот (RU) | ● (заявки на материалы) | ●● (фото к ежедневным отчётам мастера и к приходу материалов) | ● | — | Привязка фото сразу к трём сущностям: отчёт дня, работа, поставка |
| ПрорАКТ (open source, MIT, Go+Postgres) | ● (объекты, акты, позиции, оплаты, долги) | ●● (фото по `file_id` к объекту/акту, дата, подпись) | ● | — | Готовая референс-реализация «бот + объекты + фото + деньги»; структура `internal/domain`, `store`, `tg`, `app` |
| ERPNext (MIT) / Odoo CE | ●● (Project как хаб: timesheets, expense claims, purchase invoices, budget vs actual) | ● (Odoo construction apps: daily log с фото) | ●● | ● | Если нужен полноценный ERP-контур, а не тонкий модуль |
| PROBIS / Nomitech CosMO / Sage Estimating / RSMeans / BCIS | — | — | ● (comparison groups) | ●● | Нормализация: индекс времени × региональный фактор; ключевые показатели на м²; библиотека истории |
| Гранд-Смета / Smeta.ru | ● (сметы, акты) | — | — | ● (объекты-аналоги, индексы) | Импорт смет как источник затрат; сами бенчмарка не делают |

## 2. По каждому требованию: у кого что брать

### 2.1 Объекты и затраты в одном блоке

**Модель, которую стоит скопировать (Procore WBS + JobTread budget-first).**
- Затрата всегда имеет три координаты: объект → cost code (вид работ) → cost type (материалы / работы / техника / субподряд / прочее). Procore по умолчанию строит cost codes по CSI MasterFormat и добавляет кастомные сегменты (Location, Phase). Для складов достаточно 20–40 собственных кодов: кровля, фасад/сэндвич-панели, металлокаркас, полы, ворота/доки, спринклеры, электрика, ЛВС/СКС, благоустройство, охрана, демонтаж, вывоз мусора.
- Затраты рождаются из **событий изменений** (change events), а не появляются сами по себе: у Procore и Autodesk событие → оценка (PCO) → утверждённое изменение (CO) → факт. Для вашего кейса событие = «атака 18.07», «обследование», «начало восстановления», «ввод».
- Три колонки на строку бюджета: план / факт / прогноз. Buildertrend по умолчанию берёт в прогноз наибольшую из оценок, чтобы не занижать.
- История не удаляется: cost code деактивируется, но остаётся во всех прошлых объектах (Buildertrend). Внешние документы (счёт, акт КС-2, платёжка) хранятся ссылкой через `origin_id`/`origin_data` (Procore).
- Российский слой: разрезы 1С:Подрядчика (стройка → объект → статья затрат → технологическая структура → подрядчик) и формы КС-2/КС-3/КС-6а, если нужна сверка с бухгалтерией.

### 2.2 Фото из мессенджеров с привязкой к объекту, дате и изменениям

**Модель фото (CompanyCam) + привязка к событию (Procore Daily Log).**
- Поля Photo у CompanyCam: `captured_at` (unix), `coordinates {lat, lon}`, `tags[]`, `description`, `internal`, массив `uris` с размерами. Procore добавляет Location, Trade, Private и кнопку «добавить в журнал за день съёмки».
- Добавить поля источника: `channel` (telegram / max), `chat_id`, `sender`, `message_id`, `file_unique_id`, `sha256` (дедуп), `received_at` отдельно от `captured_at`, `exif_date` nullable.
- Привязка: `object_id` обязателен, `event_id` опционален. Таймлайн объекта группируется по дням; событие даёт «до/после» (паттерн before/after у CompanyCam и TimeTravel у HoloBuilder).

**Ограничения Telegram Bot API, которые влияют на схему.**
- Фото, отправленное как «photo», пережато и без EXIF: дата съёмки и GPS теряются. Берём `message.date` и просим отправлять «как файл» (document), где EXIF/GPS сохраняются.
- `getFile` отдаёт файлы до 20 МБ; локальный Bot API server снимает лимит. `file_name`/`mime_type` сохранять сразу, повторный `getFile` их не гарантирует.

**MAX Bot API (dev.max.ru).**
- Вложения приходят в `message.body.attachments[]` с `type: image` и `payload {photo_id, token, url}`; клавиатура тоже лежит в attachments, фильтровать по типу. Форматы до 50 МБ / 7680 px.
- Домен `platform-api.max.ru`, токен только в заголовке `Authorization`, вебхуки только по HTTPS с сертификатом доверенного ЦС (с 25.05.2026).
- Публиковать ботов могут только верифицированные юрлица РФ; ИП, самозанятые и физлица пока не могут. Это главный организационный барьер.

**Состояние мессенджеров в РФ на сентябрь 2026.** WhatsApp фактически не работает с февраля; Telegram под замедлением с перебоями (в апреле OONI фиксировал 95% аномалий, в августе давление усилилось), формального «полного» запрета нет; MAX предустановлен, 85 млн пользователей, обязателен для ссылок на сайтах. Вывод: инжест через ботов, а не через номер, и минимум два канала (Telegram-бот + MAX-бот) с общим обработчиком.

**Готовые RU-референсы.** Gectaro @GectaroBot (список проектов → работы → факт, всё в систему), ПрорАКТ (MIT, Go+Postgres, фото по `file_id` к объекту/акту с датой и подписью, mini app), Стройбот (фото к ежедневным отчётам и к приходу материалов), Service Inspector (отчёты PDF в Telegram в обратную сторону).

### 2.3 Единый реестр проектов с разрезами

**Атрибуты сопоставимости (RICS «Cost analysis and benchmarking», BCIS SFCA, PROBIS, IBC Real Estate).**
- Тип/функция объекта (ФЦ, СЦ, склад, фотостудия), класс (A: высота ≥ 12 м, нагрузка 8 т/м²), сухой / холодильный / фулфилмент.
- Площадь по единой методике измерения и полоса площади: до 10, 20, 30–70, 70+ тыс. м² (у IBC разная себестоимость по полосам).
- Регион, федеральный округ, кластер (уже есть в дашборде), координаты, адрес.
- Собственность / аренда / субаренда, собственник, арендаторы (уже есть в `DMG`).
- Даты: ввод, событие, начало/конец работ; статус объекта и статус проекта (оценка / тендер / работы / закрыт).
- Способ закупки (генподряд, свои силы, конкурс), число участников тендера, «абнормальные» затраты, что включено в границу стоимости (без земли, ТУ, проектирования). Без этого сравнение из п. 4 не работает.

**Паттерны.** Procore: multi-tier Locations внутри проекта (площадка → корпус → зона → пролёт), уникальные имена. PROBIS: «comparison groups» по категории, локации и стратегии. В дашборде массивы `D` (name, lon, lat, status, area) и `DMG` (name, address, lat, lon, area, owner, rented, note) нужно свести в одну таблицу объектов, а даты и заметки вынести в события.

**2.3b. Если имелся в виду версионный учёт.** Fieldwire: Floorplan → Sheets (версии), Sheet Compare с цветной разницей; Bluebeam: overlay/compare и slip-sheeting; Buildertrend: история изменений документов для клиента. Паттерн: неизменяемые снимки сметы/паспорта объекта с датой и автором, diff между снимками.

### 2.4 «Прошлая похожая работа vs новая»

**Метод (RICS/BCIS, CNBA guide 2026).**

```
Скорректированная стоимость аналога =
  историческая стоимость
  × (индекс сейчас / индекс на дату аналога)
  × (региональный фактор целевого региона / фактор региона аналога)
  + разница объёма работ + условия площадки + допуски на риск
```

- Сравнивать в стандартных единицах: ₽/м², ₽/т металла, ₽ на док-позицию, доля от стоимости объекта.
- Собственная история, размеченная по типу, региону, объёму, дате и способу закупки, ценнее внешних справочников; небольшого, но хорошо размеченного набора хватает для диапазона.
- ₽/м² на ранней стадии даёт ±20–30% (AACE class 5) и служит сигналом, а не выводом; для контрактных решений нужен постатейный расчёт.

**Источники индексов и региональных факторов для РФ.**
- Письма Минстроя об индексах изменения сметной стоимости по кварталам и регионам (3 кв. 2026: № 45358-ИФ/09 от 23.07.2026, № 46728-ИФ/09 от 30.07.2026, № 53691-АЛ/09 от 28.08.2026). Удобные таблицы: its.i-tat.ru/indexes.
- ФГИС ЦС: индексы для ресурсно-индексного метода, ФСНБ-2022, НЦС-2026 (приказы от 19.03.2026). Формула НЦС: `С = [(НЦС × M × Кпер × Кпер/зон × Крег) + Зр] × Ипр + НДС`. При отсутствии норматива Методика 421/пр (п. 13) прямо предписывает считать по объектам-аналогам.
- Отраслевой бенчмарк по складам: IBC Real Estate + MEGASTROY + Сбер, квартальные выпуски.

| Показатель (1 кв. 2026, генподряд без НДС, сухой склад 20 тыс. м², без земли и ТУ) | Значение |
|---|---|
| ЦФО, база | 69 100 ₽/м² |
| ДФО / СФО | 76 000 / 74 600 ₽/м² |
| УФО | 70 800 ₽/м² |
| СЗФО / СКФО | 67 700 / 66 500 ₽/м² |
| ЮФО / ПФО | 64 500 / 64 300 ₽/м² |
| Материалы / работы | 38 600 / 30 500 ₽/м² |
| Эффект масштаба 30–70 тыс. м² | −3…−5% |
| Эффект масштаба 70+ тыс. м² | −5…−10% |
| Фулфилмент-центр | +30…35% (83 200–86 600 ₽/м²) |
| Рост с 2021 г. | с 38 000 до 69 100 ₽/м² |
| Прогноз на конец 2026 (Коммерсантъ) | 75 000–78 000 ₽/м² |
| Оценка CORE.XP на начало 2026 | 71 400 ₽/м² |

**Инструменты-референсы.** PROBIS (собственная история + индексация + location factors, CPI), Nomitech CosMO, Sage Estimating / InEight / WinEst (библиотеки истории), RSMeans (970+ городов, предиктивные цены), BCIS (21 000+ проектов, TPI, location factors). В РФ: Pragmacore.Оферта, Adept; Гранд-Смета и Smeta.ru нужны как источник смет, бенчмарк они не делают.

## 3. Рекомендуемая основа для собственного модуля

Тяжёлую платформу не брать: ни один продукт не закрывает все четыре пункта, а российские API закрыты (Gectaro по запросу, Pragmacore без публичной документации). Собрать тонкое ядро из пяти таблиц, бота и существующего дашборда.

```sql
objects   (id, name, type, class, region, federal_district, cluster, area_m2,
           lat, lon, ownership, owner, tenants, status, commissioned_at)
events    (id, object_id, date, kind,        -- attack | inspection | repair_start | repair_done | commissioning
           description, source_ref)
photos    (id, object_id, event_id NULL, captured_at, received_at, exif_date NULL,
           lat NULL, lon NULL, channel, chat_id, sender, message_id,
           file_unique_id, sha256, storage_url, caption, tags[])
cost_codes(code, name, cost_type)            -- roof | facade | frame | floor | docks | fire | electric | ...
costs     (id, object_id, event_id NULL, cost_code, qty, unit, unit_cost, amount,
           currency, date, contractor, doc_ref, kind)   -- plan | actual | forecast
indices   (period, region, index_value, source)          -- Минстрой / IBC / свой
```

- Инжест: Telegram-бот (aiogram или python-telegram-bot) и MAX-бот (maxo) → один обработчик → оригиналы в S3/MinIO, метаданные в Postgres/SQLite → дашборд читает JSON или API. Сценарий бота как у Gectaro: выбрать объект → выбрать событие или «просто фото» → подпись → готово.
- Бенчмарк: функция `normalize(amount, date, region) → base` по таблице `indices`, поиск аналогов по (type, полоса площади ±30%, округ, cost_code), вывод медианы и диапазона аналогов рядом с новой оценкой и CPI.
- Открытые референсы кода и схем: ПрорАКТ (модель и FSM бота), CompanyCam OpenAPI YAML (Photo/Project/Webhook), Fieldwire OpenAPI (`console.fieldwire.net/oas/v3.json`, `developers.fieldwire.com/llms.txt`), Procore docs на GitHub, ERPNext (если позже понадобится ERP-контур).
- Не делать: 360-съёмку, BIM, сметные нормативы ФЕР/ГЭСН внутри модуля. Для складской аналитики это избыточно.

## 4. Риски и ограничения

- Мессенджеры: Telegram нестабилен, MAX только для юрлица; держать оба канала и кнопку «загрузить файлом» как запасной путь.
- Данные фото: у пережатых фото нет даты съёмки, полагаться на `message.date`; дедупликация по sha256; хранить оригиналы вне мессенджера.
- Бенчмарк: сравнение корректно только при единой границе стоимости (что включено) и полосе площади; ₽/м² даёт порядок величины.
- Вендорские API в РФ закрыты или ограничены; строить интеграции на своих данных, а не на чужих платформах.

## 5. Источники

Обзоры рынка: [PlanRadar, 15 tools 2026](https://www.planradar.com/us/the-15-best-construction-management-software-tools-in-2026-compared-honestly/) · [Fieldwire best CM software](https://www.fieldwire.com/blog/best-construction-management-software/) · [Construction Coverage](https://constructioncoverage.com/construction-project-management-software) · [G2](https://learn.g2.com/best-construction-project-management-software) · [a2is стройконтроль](https://a2is.ru/catalog/programmy-dlya-stroitelnogo-kontrolya) · [pickTech](https://picktech.ru/catalog/construction-management-software/) · [Cynteka обзор 2025](https://cynteka.ru/publikatsii/programmy-dlya-stroitelnykh-kompaniy-obzor-it-resheniy-dlya-avtomatizatsii-biznesa/)

Procore / Autodesk / JobTread / Buildertrend: [Procore cost codes](https://developers.procore.com/reference/rest/cost-codes?version=latest) · [Procore daily logs](https://procore.github.io/documentation/daily-logs) · [Procore attachments/images](https://procore.github.io/documentation/attachments) · [Procore change orders](https://procore.github.io/documentation/tutorial-change-orders) · [Procore multi-tier locations](https://support.procore.com/products/online/user-guide/project-level/admin/tutorials/add-multi-tiered-locations-to-a-project) · [Procore REST overview](https://github.com/procore/documentation/blob/main/getting_started/rest_api_overview.md) · [Autodesk cost mgmt Nov 2025](https://blogs.autodesk.com/bim360-release-notes/2025/11/18/bim-360-cost-management-november-2025/) · [Autodesk cost mgmt May 2025](https://blogs.autodesk.com/bim360-release-notes/2025/05/20/bim-360-cost-management-may-2025/) · [Autodesk cost workflow](https://construction.autodesk.com/workflows/construction-cost-management/) · [JobTread job costing](https://www.jobtread.com/features/job-costing) · [JobTread daily logs](https://www.jobtread.com/features/daily-logs) · [JobTread review 2026](https://stackvett.com/jobtread-review/) · [Buildertrend job costing](https://buildertrend.com/help-article/job-costing-budget-overview/) · [Buildertrend cost codes](https://buildertrend.com/help-article/cost-codes-overview/) · [Buildertrend daily logs mobile](https://buildertrend.com/help-article/daily-logs-on-mobile/)

Фото и поле: [Filio comparison](https://www.filio.io/blog/best-construction-photo-documentation-software/) · [CompanyCam timelines](https://companycam.com/features/galleries-timelines) · [CompanyCam Photo object](https://docs.companycam.com/reference/photo-1) · [CompanyCam webhooks](https://docs.companycam.com/docs/webhooks-1) · [CompanyCam tags on upload](https://docs.companycam.com/changelog/add-tags-when-uploading-a-photo-to-a-project) · [OpenSpace Capture](https://www.openspace.ai/products/capture/) · [HoloBuilder](https://www.faro.com/en/Products/Software/HoloBuilder-Software) · [PlanRadar vs OpenSpace vs Buildots](https://www.planradar.com/gb/360-construction-photo-documentation-software-planradar-vs-openspace-vs-buildots/) · [Gmini 360](https://gmini.tech/tpost/ps62538oe1-kak-fotofiksatsiya-360-prevraschaet-stro) · [Fieldwire getting started](https://developers.fieldwire.com/docs/getting-started) · [Fieldwire sheet versions](https://developers.fieldwire.com/docs/uploading-plans-via-api) · [Fieldwire webhook events](https://developers.fieldwire.com/docs/webhook-event-types) · [PlanRadar core API](https://help.planradar.com/hc/en-gb/articles/33438887517853-Core-API-Concepts) · [PlanRadar webhooks](https://help.planradar.com/hc/en-gb/articles/15480642226589-Setup-Webhooks) · [Bluebeam vs Fieldwire](https://www.selecthub.com/construction-management-software/bluebeam-revu-vs-fieldwire/) · [Bluebeam drawing mgmt](https://www.bluebeam.com/workflows/drawing-and-document-management/)

Российские продукты: [Adept УС](https://gk-adept.ru/programmy/adept_upravlenie_stroitelstvom/) · [Adept Стройконтроль](https://gk-adept.ru/programmy/adept_strojkontrol/) · [Pragmacore](https://pragma.info/) · [Pragmacore Оферта](https://pragma.info/offer) · [Pragmacore предиктивная аналитика](https://www.cnews.ru/news/line/2024-07-04_pragmacore_predstavila_modul) · [Pragmacore на pickTech](https://picktech.ru/product/pragmacore/) · [Gectaro Telegram-бот](https://gectaro.com/updates/tpost/7xuufogau1-telegram-bot-dlya-prorabov-papki-v-sprav) · [Gectaro Open API](https://gectaro.com/updates/tpost/o8m610kim1-open-api-svyazivaite-gectaro-s-lyubimi-s) · [Gectaro в Albato](https://albato.com/apps/gectaro) · [1С:Подрядчик. Финансы](https://solutions.1c.ru/catalog/finstroy/features) · [1С:Подрядчик. Производство](https://solutions.1c.ru/catalog/stroy/features) · [Техзор FAQ/API](https://tehzor.com/faqs/) · [MRS СтройКонтроль](https://mrspro.ru/strojkontrol/) · [Стройбот фотофиксация](https://stroybots.ru/photofixing_in_construction) · [ПрорАКТ (GitHub, MIT)](https://github.com/Sereban-glitch/proakt-build-bot) · [Service Inspector отчёты в Telegram](https://serviceinspector.ru/otchety_v_telegram) · [Photo-reports в строительстве](https://photo-reports.online/ru/docs/use-cases/construction-photo-report)

Мессенджеры: [Telegram Bot API](https://core.telegram.org/bots/api) · [Telegram и EXIF](https://www.sammapix.com/blog/does-telegram-strip-exif-metadata) · [getFile 20 MB, local server](https://github.com/tdlib/telegram-bot-api/issues/141) · [MAX docs-api](https://dev.max.ru/docs-api) · [MAX POST /uploads](https://dev.max.ru/docs-api/methods/POST/uploads) · [MAX: только юрлица](https://habr.com/ru/articles/951326/) · [MAX: грабли Bot API](https://habr.com/ru/articles/1060586/) · [maxo (Python)](https://github.com/K1rL3s/maxo) · [Блокировка Telegram 2026 (Wikipedia)](https://ru.wikipedia.org/wiki/Блокировка_Telegram_в_России_(2026)) · [CNews о WhatsApp](https://www.cnews.ru/news/top/2026-02-11_po_primeru_youtubevlasti_okonchatelno) · [Битрикс24 о Telegram](https://www.bitrix24.ru/journal/blokirovka-telegram/) · [Albato о WhatsApp для бизнеса](https://blog.albato.ru/whatsapp-dlya-biznesa-2026-blokirovka-alternativy/)

Бенчмаркинг: [RICS Cost analysis and benchmarking](https://www.rics.org/content/dam/ricsglobal/documents/standards/Cost-analysis-and-benchmarking_2nd-edition.pdf) · [BCIS six principles](https://www.bcis.co.uk/insight/bcis-basics-six-principles-of-benchmarking-in-construction/) · [CNBA guide 2026](https://cnba.us/2026/05/05/construction-cost-benchmarking-guide/) · [Rate QS benchmarking](https://rateqs.com/insights/construction-cost-benchmarking/) · [PROBIS benchmark](https://en.prob.is/feature/benchmark-analysis) · [Nomitech CosMO](https://www.nomitech.com/product/cosmo) · [RSMeans Online](https://www.rsmeans.com/products/online) · [Estimating tools 2026](https://thedigitalprojectmanager.com/tools/best-construction-estimating-software/) · [Estimating ranked by database](https://programminginsider.com/7-construction-cost-estimating-software-ranked-by-database-accuracy/)

Ценообразование РФ: [ФГИС ЦС](https://fgiscs.minstroyrf.ru/) · [НЦС на сайте Минстроя](https://minstroyrf.gov.ru/trades/tsenoobrazovanie/ukrupnennye-normativy-tseny-stroitelstva/) · [НЦС-2026 приказы](https://its.i-tat.ru/articles/opublikovany-prikazy-minstroya-ob-utverzhdenii-ukrupnennyh-normativov-ceny-stroitelstva-ncs-2026) · [Индексы Минстроя по кварталам](https://its.i-tat.ru/indexes/) · [Письмо на 3 кв. 2026](https://www.consultant.ru/law/hotdocs/94969.html) · [Методика 421/пр](https://base.garant.ru/74676474/) · [Расчёт по объектам-аналогам](https://expertsmet.ru/articles/raschet-stoimosti-stroitelstva-po-obektam-analogam/) · [Гранд-Смета](https://www.grandsmeta.ru/) · [Smeta.ru](https://smeta.ru/products/smeta-ru)

Себестоимость складов: [IBC Real Estate, 1 кв. 2026](https://ibcrealestate.ru/research/2026-q1-stoimost-stroitelstva-skladskaya-nedvizhimost/) · [IBC: региональная таблица](https://ibcrealestate.ru/blog/stoimost-stroitelstva-sklada-v-2026-godu-ot-chego-zavisit-czena-za-m/) · [Ведомости 12.05.2026](https://www.vedomosti.ru/realty/articles/2026/05/12/1196531-sebestoimost-stroitelstva-skladov-pochti-perestala-rasti) · [Коммерсантъ](https://www.kommersant.ru/doc/8533059) · [CORE.XP](https://core-xp.ru/research/stoimost-stroitelstva-sklada/)

Open source: [Fuzen: 7 OSS tools](https://www.fuzen.io/posts/open-source-construction-management-software) · [ERPNext for EPC](https://frappe.io/erpnext/for-EPC-engineering-contruction) · [Odoo vs OpenProject vs ERPNext](https://www.devolute.org/en/insights/open-source/odoo-vs-openproject-vs-erpnext) · [Odoo Construction Management app](https://apps.odoo.com/apps/modules/19.0/sdlc_construction_management) · [Odoo for general contractors](https://www.odoo.com/industries/construction/general-contractor)
