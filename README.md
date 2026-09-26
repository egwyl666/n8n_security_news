# CyberSec Digest — n8n → Telegram

> An autonomous SOC-flavored news desk that never sleeps: it watches the wires, filters the noise, asks Gemini to write the briefing, and delivers it straight to your phone.

[![n8n](https://img.shields.io/badge/n8n-workflow-EA4B71?logo=n8n&logoColor=white)](https://n8n.io)
[![Gemini](https://img.shields.io/badge/AI-Gemini-8E75B2?logo=googlegemini&logoColor=white)](https://aistudio.google.com)
[![Telegram](https://img.shields.io/badge/delivery-Telegram-26A5E4?logo=telegram&logoColor=white)](https://telegram.org)
[![License](https://img.shields.io/badge/license-MIT-informational)](#)

Read this in: **[English](#english)** · **[Українською](#українська)**

---

## English

### What this is

`CyberSec Digest` is a self-hosted **n8n** workflow that turns a firehose of security feeds into a short, readable Telegram briefing every 30 minutes — written in the tone of a SOC analyst, not a press release.

Every run, the workflow:

1. **Collects** from five independent sources in parallel:
   - [The Hacker News](https://thehackernews.com) (RSS)
   - [BleepingComputer](https://www.bleepingcomputer.com) (RSS)
   - [Krebs on Security](https://krebsonsecurity.com) (RSS)
   - [NVD](https://nvd.nist.gov) — freshly published CVEs
   - [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) — vulnerabilities confirmed as actively exploited in the wild
2. **Filters the signal from the noise**: only CVEs with CVSS ≥ 7 survive, KEV entries are limited to the last 3 days, and news older than 24 hours is dropped.
3. **Normalizes and de-duplicates** everything into one shape, keyed on the CVE ID (or link) so the same story never gets sent twice across runs.
4. **Asks Gemini** to write the actual digest: strict Telegram-HTML formatting, grouped by severity (KEV → CVE → News), each item with a plain-language summary *and* a concrete "what to check/patch" note for the analyst on shift.
5. **Delivers to Telegram** as one or more HTML messages, splitting long digests automatically so nothing gets cut off mid-sentence.
6. Falls back gracefully — if Gemini is unreachable or returns garbage, the workflow still sends a plain digest built from the raw feed data instead of failing silently.

### Architecture at a glance

```
Schedule (every 30 min)
        │
        ├── The Hacker News ─────┐
        ├── BleepingComputer ────┤
        ├── Krebs on Security ───┤
        ├── NVD (CVE ≥ 7 CVSS) ──┤──→ Merge → Normalize → Dedupe → Build prompt → Gemini → Parse → Telegram
        └── CISA KEV (last 3d) ──┘
```

### Repository contents

| File | Purpose |
|---|---|
| `CyberSec Digest(telegram).json` | The n8n workflow, ready to import |
| `README.md` | This file |

### Installation

1. **Import the workflow**
   - Open your n8n instance.
   - `Workflows → Add workflow → Import from File` (or `⋮ → Import from File`).
   - Select `CyberSec Digest(telegram).json`.

2. **Credentials are not exported with the workflow** — n8n never bundles secrets, so you'll create them fresh on your own instance:

   **Gemini (Google AI Studio)**
   - Get an API key at [aistudio.google.com](https://aistudio.google.com).
   - In the `Gemini` node, create a new **Header Auth** credential:
     - Name: `x-goog-api-key`
     - Value: your API key
   - Attach it to the `Gemini` node.

   **Telegram**
   - Create a bot via [@BotFather](https://t.me/BotFather) and copy its token.
   - Create a **Telegram API** credential in n8n using that bot token.
   - Attach it to the `Telegram` node.

   **Chat ID**
   - Message your bot directly, then read the chat id from the `getUpdates` API call, or ask [@userinfobot](https://t.me/userinfobot).
   - Paste it into the `chatId` field of the `Telegram` node (replacing the placeholder).

3. **Double-check the model name.** The `Gemini` node's URL points at a specific model version:
   ```
   https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent
   ```
   If that model isn't available or has been renamed on your account, the request will return a `404`. Swap in whatever current model your account exposes.

4. **Test before you flip it on:**
   - Run the workflow **manually once** — expect a large digest covering everything currently in the feeds.
   - Run it **manually a second time immediately after** — this run should produce **0 messages**, and the `Убрать уже отправленное` (dedup) node should report **0 kept items** on its output.
   - If the second run still sends messages, the dedup logic isn't matching correctly — check that node's settings before activating the schedule trigger.

5. **Activate** the workflow. It will now poll every 30 minutes on its own.

### Notes

- The severity threshold (CVSS ≥ 7) and lookback windows (24h news / 3d KEV) are set inside the `Code` nodes and can be tuned to taste.
- Every HTTP-fetching node uses `continueRegularOutput` on error, so one dead feed won't take the whole run down.

---

## Українська

### Що це

`CyberSec Digest` — це self-hosted воркфлоу для **n8n**, який перетворює потік розрізнених security-фідів на короткий, зручний для читання дайджест у Telegram кожні 30 хвилин — написаний у тоні аналітика SOC, а не прес-релізу.

На кожному запуску воркфлоу:

1. **Збирає дані** одразу з п'яти незалежних джерел:
   - [The Hacker News](https://thehackernews.com) (RSS)
   - [BleepingComputer](https://www.bleepingcomputer.com) (RSS)
   - [Krebs on Security](https://krebsonsecurity.com) (RSS)
   - [NVD](https://nvd.nist.gov) — щойно опубліковані CVE
   - [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) — вразливості, що підтверджено активно експлуатуються
2. **Відсіює шум**: залишаються лише CVE з CVSS ≥ 7, записи KEV обмежені останніми 3 днями, а новини старші за 24 години відкидаються.
3. **Нормалізує та дедублікує** усе до єдиного формату за ключем CVE-ідентифікатора (або посилання), щоб та сама новина не прийшла двічі в різних запусках.
4. **Звертається до Gemini**, щоб написати сам дайджест: сувора HTML-розмітка Telegram, групування за критичністю (KEV → CVE → Новини), кожен пункт із коротким поясненням суті *та* конкретною порадою для чергового аналітика — що перевірити чи запатчити.
5. **Надсилає в Telegram** одним або кількома HTML-повідомленнями, автоматично розбиваючи довгі дайджести, щоб текст не обривався на півслові.
6. Має «м'який» відкат — якщо Gemini недоступний або повернув сміття, воркфлоу все одно надішле простий дайджест, зібраний із сирих даних фідів, замість того щоб мовчки провалитись.

### Архітектура коротко

```
Тригер за розкладом (кожні 30 хв)
        │
        ├── The Hacker News ─────┐
        ├── BleepingComputer ────┤
        ├── Krebs on Security ───┤
        ├── NVD (CVE ≥ 7 CVSS) ──┤──→ Merge → Нормалізація → Дедуп → Промпт → Gemini → Парсинг → Telegram
        └── CISA KEV (3 дні) ────┘
```

### Вміст репозиторію

| Файл | Призначення |
|---|---|
| `CyberSec Digest(telegram).json` | Воркфлоу n8n, готовий до імпорту |
| `README.md` | Цей файл |

### Встановлення

1. **Імпортуйте воркфлоу**
   - Відкрийте свій інстанс n8n.
   - `Workflows → Add workflow → Import from File` (або `⋮ → Import from File`).
   - Оберіть файл `CyberSec Digest(telegram).json`.

2. **Credentials не експортуються разом із воркфлоу** — n8n ніколи не тягне за собою ключі й токени, тож їх треба буде створити наново на своєму інстансі:

   **Gemini (Google AI Studio)**
   - Отримайте API-ключ на [aistudio.google.com](https://aistudio.google.com).
   - У ноді `Gemini` створіть новий credential типу **Header Auth**:
     - Name: `x-goog-api-key`
     - Value: ваш API-ключ
   - Прив'яжіть його до ноди `Gemini`.

   **Telegram**
   - Створіть бота через [@BotFather](https://t.me/BotFather) і скопіюйте його токен.
   - Створіть credential типу **Telegram API** в n8n, використовуючи цей токен.
   - Прив'яжіть його до ноди `Telegram`.

   **Chat ID**
   - Найпростіше — написати боту в особисті повідомлення, а потім подивитись id через виклик `getUpdates`, або скористатися [@userinfobot](https://t.me/userinfobot).
   - Вставте його у поле `chatId` ноди `Telegram` (замінивши плейсхолдер).

3. **Перевірте назву моделі.** URL у ноді `Gemini` вказує на конкретну версію моделі:
   ```
   https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent
   ```
   Якщо ця модель недоступна або перейменована на вашому акаунті, запит поверне `404`. Підставте актуальну назву моделі зі свого акаунта.

4. **Перевірте перед активацією:**
   - Запустіть воркфлоу **вручну один раз** — прийде великий дайджест за весь наявний у фідах період.
   - Одразу після цього запустіть **вручну ще раз** — цей запуск має дати **0 повідомлень**, а нода `Убрать уже отправленное` (дедуп) на виході має показати **0 kept items**.
   - Якщо другий запуск все одно надсилає повідомлення — дедуп працює некоректно, перевірте налаштування цієї ноди перед активацією тригера за розкладом.

5. **Активуйте** воркфлоу. Далі він буде опитувати джерела самостійно кожні 30 хвилин.

### Примітки

- Поріг критичності (CVSS ≥ 7) та вікна перегляду (24 год для новин / 3 дні для KEV) задані всередині нод `Code` і легко налаштовуються під себе.
- Кожна нода, що робить HTTP-запит, використовує `continueRegularOutput` при помилці, тож одне непрацююче джерело не зупинить увесь запуск.
