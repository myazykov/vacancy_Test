# Agreement Management Process Optimization

> Анализ и оптимизация процесса хранения, согласования и контроля договоров (User Agreements) с целью создания единого портала для Legal, обеспечения прослеживаемой цепочки «договор → инвойс → оплата» для Finance и сокращения SLA с 2 недель до 1 недели.

---

## Содержание

| Артефакт | Описание |
|----------|----------|
| 📄 **README.md** | Обзор проекта, ключевые диаграммы, навигация *(вы здесь)* |
| 📋 [requirements.md](requirements.md) | Реестр требований: функциональные, нефункциональные, ограничения, трассируемость |
| 🔍 [diagrams_as_is.md](diagrams_as_is.md) | AS-IS: Sequence Diagrams трёх потоков (оферта, Self-form, оплаты) |
| 🎯 [diagrams_to_be.md](diagrams_to_be.md) | TO-BE: Sequence Diagrams с развилками решений |
| 🔄 [diagrams_state.md](diagrams_state.md) | State Diagrams: жизненный цикл договора (Self-form + Offer) |
| 🗄️ [data_model_and_exchanges.md](data_model_and_exchanges.md) | Модель данных (ER), схемы обмена, JSON-контракты, обработка ошибок |
| 🌐 [Интерактивный отчёт](https://myazykov.github.io/Emex_Test/) | **[Открыть →](https://myazykov.github.io/Emex_Test/)** Контекст, AS-IS, GAP, TO-BE, риски, roadmap, вопросы |

---

## Контекст задачи

Компания работает с ~200 поставщиками и ~500 покупателями. Договорные отношения оформляются двумя способами: акцепт публичной оферты на сайте (SellerDrive / Emexdwc) и подписание индивидуального договора (Self-form agreement) через DocuSign. Параллельно Finance ведёт оплаты в Airtable Payments, а инвойсы архивируются в Odoo Documents.

**Заказчики:** Legal (основной) и Finance (со-заказчик).

**Проблема:** процесс рассредоточен по 6 системам без интеграций. Реестр договоров покрывает ~40% контрагентов. Нет связи между договорами и оплатами. Нет единой точки входа для юристов. SLA не контролируется.

---

## Ландшафт систем

```mermaid
graph LR
    subgraph Порталы
        SD[SellerDrive<br/>поставщики]
        EM[Emexdwc<br/>покупатели]
    end

    subgraph SaaS
        DS[DocuSign<br/>подписание]
        AT_A[Airtable<br/>Agreements]
        AT_P[Airtable<br/>Payments]
        ODOO[Odoo<br/>Documents]
    end

    subgraph Каналы
        EMAIL[Email<br/>Sales → Legal]
    end

    SD -. "нет интеграции" .-> AT_A
    EM -. "нет интеграции" .-> AT_A
    EMAIL --> DS
    DS -- "ручная загрузка<br/>~40% покрытие" --> AT_A
    AT_A -. "нет связи" .-> AT_P
    ODOO -. "нет связи" .-> AT_P
```

---

## GAP-анализ (сводка)

```mermaid
graph TD
    G1["G1 · CRITICAL<br/>Данные оферты изолированы<br/>в БД порталов"]
    G2["G2 · CRITICAL<br/>Покрытие реестра ~40%"]
    G3["G3 · CRITICAL<br/>Нет связи<br/>договор ↔ оплата"]
    G4["G4 · HIGH<br/>Email без трекинга"]
    G5["G5 · HIGH<br/>Нет единой точки входа"]
    G6["G6 · HIGH<br/>Разрозненные архивы"]
    G7["G7 · MEDIUM<br/>Неполнота данных<br/>Beneficiary"]

    G1 --> S1["Интеграция порталов<br/>→ Airtable"]
    G2 --> S2["Автозагрузка<br/>из DocuSign"]
    G3 --> S3["Synced Table +<br/>Pre-payment check"]
    G4 --> S4["Dedicated mailbox<br/>+ AI-парсинг"]
    G5 --> S5["Airtable Interface<br/>для Legal"]
    G6 --> S6["Odoo ↔ Airtable<br/>sync (Phase 3)"]
    G7 --> S7["Validation rules +<br/>Lookup"]

    style G1 fill:#fee2e2,stroke:#ef4444,color:#7f1d1d
    style G2 fill:#fee2e2,stroke:#ef4444,color:#7f1d1d
    style G3 fill:#fee2e2,stroke:#ef4444,color:#7f1d1d
    style G4 fill:#fef9c3,stroke:#eab308,color:#713f12
    style G5 fill:#fef9c3,stroke:#eab308,color:#713f12
    style G6 fill:#fef9c3,stroke:#eab308,color:#713f12
    style G7 fill:#f3e8ff,stroke:#a855f7,color:#581c87
```

---

## Ключевые развилки TO-BE

Поскольку интервью со стейкхолдерами не проводились, решение построено с вариативностью. Каждая развилка содержит 2–3 варианта с аргументацией «за / против». Подробности — в [diagrams_to_be.md](diagrams_to_be.md) и [docs/analysis.html](docs/analysis.html).

| # | Развилка | Варианты | Рекомендация |
|---|----------|----------|--------------|
| 1 | Передача данных об акцепте оферты с порталов | A: Event-driven (webhook) · B: Scheduled sync (cron) · C: iPaaS (n8n/Zapier) | A — realtime, минимальная доработка бэкенда |
| 2 | Канал входа запросов от Sales | A: Dedicated mailbox + AI-парсинг · B: Airtable Form · C: Email-шаблон + парсинг | A — Sales не меняют привычку, данные структурируются через LLM |
| 3 | Загрузка подписанного документа из DocuSign | A: Полный авто → Registered · B: Авто → PendingReview → Registered · C: Ручная + напоминания | B — автоматизация + контроль юриста |
| 4 | Связь договоров с оплатами | A: Synced Table + Lookup (без изменения Beneficiary) · B: Добавление полей в Beneficiary | A — минимальное вмешательство в структуру Payments |
| 5 | Pre-payment check режим | A: Soft mode (warning) · B: Strict mode (block) | A→B — начать с soft, перейти в strict при покрытии ≥95% |

---

## Жизненный цикл договора

```mermaid
stateDiagram-v2
    direction LR

    [*] --> Requested: Sales запрос
    [*] --> Accepted: Акцепт оферты

    Accepted --> Active: Авто через sync

    Requested --> InReview: Юрист берёт
    InReview --> SentToSign: В DocuSign
    SentToSign --> Signed: Подписан
    Signed --> PendingReview: Авто-загрузка
    PendingReview --> Registered: Юрист подтвердил
    Registered --> Active: Действует

    Active --> Expiring: За 30 дн.
    Expiring --> Renewed: Продлён
    Expiring --> Expired: Истёк
    Renewed --> Active

    Expired --> [*]
```

---

## Целевая архитектура

```mermaid
graph TB
    subgraph Порталы
        SD[SellerDrive] -->|событие акцепта| MW
        EM[Emexdwc] -->|событие акцепта| MW
    end

    MW[n8n / middleware]

    subgraph Airtable Agreements
        OL[Offer Logs]
        TR[Self-form Tracker]
        REG[Registry]
        AUDIT[Audit Log]
        UI[Legal Interface]
    end

    subgraph DocuSign
        DS_SIGN[Подписание]
        DS_HOOK[Webhook completed]
    end

    subgraph Airtable Payments
        SYNC[Synced Table]
        CHK{Agreement Check}
        PAY[Approval flow]
        BEN[Beneficiary]
    end

    ODOO[Odoo Documents]

    MW -->|авто| OL
    MW -->|AI-parsed email| TR
    OL --> REG
    TR --> REG
    REG --> UI
    AUDIT --> UI

    UI -->|согласование| DS_SIGN
    DS_HOOK -->|авто| REG

    REG <-->|Synced Table| SYNC
    SYNC --> CHK
    CHK -->|Pass| PAY
    CHK -->|Warning/Block| PAY
    PAY --> BEN
    ODOO <-->|API sync| BEN
```

---

## KPI: до и после

| Метрика | AS-IS | TO-BE |
|---------|-------|-------|
| Покрытие реестра | ~40% | 100% |
| SLA согласования | 14 дней | ≤ 7 дней |
| Время «подписание → реестр» | 1–5 дней | < 5 мин |
| Оплат с проверкой договора | 0% | 100% |
| Потерянных запросов от Sales | не измеряется | 0 |
| Систем для ежедневной работы Legal | 3+ | 1 (+ DocuSign для подписания) |

---

## Roadmap

```mermaid
gantt
    title Roadmap внедрения
    dateFormat  YYYY-MM-DD
    axisFormat  %b %Y
    
    section Phase 1 — Фундамент
    Расширение Airtable Agreements       :p1a, 2026-04-01, 2w
    Airtable Interface для Legal          :p1b, after p1a, 1w
    Канал входа от Sales (mailbox)        :p1c, after p1a, 1w
    Automations (SLA, уведомления)        :p1d, after p1b, 1w
    Ретроспективная загрузка из DocuSign  :p1e, after p1a, 2w
    Data cleansing + Master ID            :p1f, 2026-04-01, 3w

    section Phase 2 — Интеграции
    Порталы → Airtable (событие акцепта)  :p2a, 2026-05-13, 3w
    DocuSign → Airtable (webhook)         :p2b, 2026-05-13, 2w
    Synced Table → Payments               :p2c, after p2b, 1w
    Pre-payment check (soft mode)         :p2d, after p2c, 1w
    AI-парсинг email (n8n + LLM)          :p2e, after p2b, 2w

    section Phase 3 — Прослеживаемость
    Odoo ↔ Airtable sync                  :p3a, 2026-06-22, 2w
    Strict mode + override                :p3b, after p3a, 1w
    Dashboard Finance                     :p3c, after p3a, 2w
    Reporting + validation rules          :p3d, after p3b, 1w
```

---

## Навигация по артефактам

### Анализ

- [📋 Реестр требований](requirements.md) — FR/NFR/Constraints, статусы Confirmed/Derived/Open, трассируемость
- [🌐 Интерактивный отчёт](https://myazykov.github.io/Emex_Test/) — полный анализ с навигацией (GitHub Pages)

### Диаграммы

- [🔍 AS-IS Sequence Diagrams](diagrams_as_is.md) — три потока: оферта, Self-form, оплаты
- [🎯 TO-BE Sequence Diagrams](diagrams_to_be.md) — целевые потоки с alt-блоками по развилкам
- [🔄 State Diagrams](diagrams_state.md) — жизненный цикл Self-form + Offer + общий вид

### Проектирование

- [🗄️ Модель данных и обмены](data_model_and_exchanges.md) — ER-диаграмма, JSON-контракты 5 точек интеграции, обработка ошибок, маппинг ID

---

## Подход к документированию

Проект использует **Mermaid-as-Code** для всех диаграмм. Обоснование подхода, типы используемых диаграмм и ограничения описаны в [data_model_and_exchanges.md → раздел 1](data_model_and_exchanges.md#1-обоснование-подхода-к-документированию).

**Типы диаграмм:**

- **Sequence Diagram** (UML) — взаимодействие между акторами и системами, временная последовательность, точки разрыва
- **State Diagram** (UML) — жизненный цикл сущностей, состояния, переходы, бизнес-правила
- **ER Diagram** — модель данных, связи между таблицами, ключи
- **Flowchart** — архитектура систем, потоки данных
- **Gantt** — roadmap внедрения

---

## Открытые вопросы

Полный список из 23 вопросов с привязкой к развилкам — в [интерактивном отчёте, раздел «Вопросы»](https://myazykov.github.io/Emex_Test/) и в [requirements.md, раздел 5](requirements.md#5-нерешённые-требования-зависят-от-open-questions).

Ключевые вопросы, блокирующие финализацию:

| OQ | Вопрос | К кому | Что определяет |
|----|--------|--------|----------------|
| OQ-1 | Проверяет ли Legal документ перед добавлением в реестр? | Legal | Развилка 3 (авто vs черновик) |
| OQ-7 | Как сейчас проверяется наличие договора перед оплатой? | Finance | Baseline для pre-payment check |
| OQ-9 | Кто владелец структуры Airtable Payments? | Finance/IT | Развилка 4 (Synced Table vs поля в Beneficiary) |
| OQ-14 | Готовы ли Sales отправлять на выделенный mailbox? | Sales | Развилка 2 (канал входа) |
| OQ-15 | SellerDrive и Emexdwc — одна кодовая база или разные? | Dev | Трудоёмкость интеграции (Развилка 1) |

---

## Стек и инструменты

| Компонент | Текущий | Целевой |
|-----------|---------|---------|
| Реестр договоров | Airtable Agreements (~40%) | Airtable Agreements расширенный (100%) |
| Подписание | DocuSign (ручной workflow) | DocuSign + webhook автозагрузка |
| Оплаты | Airtable Payments (изолирован) | Airtable Payments + Synced Table + Agreement Check |
| Инвойсы | Odoo Documents (изолирован) | Odoo Documents + API sync с Airtable |
| Middleware | — | n8n (self-hosted iPaaS) |
| AI-парсинг | — | LLM API (Anthropic/OpenAI) через n8n |
| Порталы | SellerDrive, Emexdwc (без интеграции) | + webhook/API для события акцепта |

---

*Business Analysis · 2026*
