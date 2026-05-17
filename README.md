<img width="1682" height="779" alt="image" src="https://github.com/user-attachments/assets/0eeba4ef-aa5e-4d54-94b0-9776389bed0b" />

# 🎯 Job Detector — Автоматизований пошук вакансій на LinkedIn

Автоматизований n8n workflow для пошуку, аналізу та оцінки вакансій на LinkedIn з відправкою персоналізованих сповіщень у Telegram та Slack.

## Зміст

- [Як це працює](#як-це-працює)
- [Архітектура](#архітектура)
- [Можливості](#можливості)
- [Встановлення](#встановлення)
- [Налаштування облікових даних](#налаштування-облікових-даних)
- [Конфігурація під себе](#конфігурація-під-себе)
- [Вартість](#вартість)

## Як це працює

Workflow працює у двох режимах паралельно:

**Режим 1 — Планове сканування (кожні 4 години):** скрапить LinkedIn за заданими ключовими словами та локацією, фільтрує дублікати, оцінює кожну вакансію через LLM, аналізує відповідність резюме та надсилає результати у Telegram і Slack.

**Режим 2 — Gmail-тригер (реального часу):** перехоплює email-сповіщення LinkedIn Job Alerts, витягує URL вакансій, скрапить їх деталі через окремий Apify актор, приводить до єдиного формату та подає в основний потік обробки.

## Архітектура

```
                                ┌─────────────────────────────────────────────────────────────────────┐
                                                                                      ДЖЕРЕЛА ВАКАНСІЙ                             
                                                                                      
                                ⏰ Trigger every 4h                        📧 Gmail Trigger 
                                        |                                      (realtime)           
                                        │                                          │                                 
                                        ▼                                          ▼                                 
                                🔍 Find LinkedIn Jobs                   🔒 Remove Duplicated Mails               
                                (Apify: keyword search)                            │                                   
                                        │                                          ▼                                   
                                        │                                 📝 Parse Jobs URL                         
                                        │                                (Code: extract job IDs)                   
                                        │                                          │                                   
                                        │                                          ▼                                   
                                        │                              🔍 Find Alerted LinkedIn Jobs             
                                        │                                  (Apify: URL scraper)                      
                                        │                                          │                                   
                                        │                                          ▼                                   
                                        │                                   🔄 Map Fields                             
                                        │                               (нормалізація формату)                    
                                        │                                          │                                   
                                        └───────────────────|──────────────────────┘                                  
                                                            │                                                 
                                                            ▼                                                 
                                              🔒 Remove Duplicate Jobs (по jobId)                          
                                                            │                                                
                                                            |
                                ├───────────────────────────|─────────────────────────────────────────┤
                                                            ▼                         ОБРОБКА ВАКАНСІЇ                         
                                                    🔁 Loop Over Items                                        
                                                            │                                                
                                                ⏳ 5s delay (rate limiting)                               
                                                            │                                                
                                                    🔀 Skip irrelevant                                       
                                            (keywordMatch < 33% → пропустити)                         
                                                            │                                                
                                                            ▼                                                
                                                    🤖 LLM: calculate job relevance                          
                                             (Gemini Flash — оцінка 0-100, cover letter,               
                                                     Telegram/Slack повідомлення)                             
                                                            |                                                
                                               🔀 SMS Gateway (score ≥ 80)                             
                                                    ├── ❌ score < 80 → наступна вакансія                    
                                                    └── ✅ score ≥ 80 ↓                                      
                                                            |                                               
                                                 📊 Find CV suggestions                                   
                                             (Apify: аналіз резюме vs вакансія)                        
                                                            |                                                
                                                    🤖 LLM: normalize message                                
                                           (Gemini Flash — переклад, нормалізація,                   
                                                  об'єднання всіх даних)                                   
                                                            |                                                
                                ├───────────────────────────|─────────────────────────────────────────┤
                                                            ▼                    ДОСТАВКА ПОВІДОМЛЕННЯ                           
                                                    ┌───────┴────────┐                                       
                                                    ▼                ▼                                        
                                                  Slack           Telegram                                  
                                               (Markdown)          (HTML)                                       
                                                    │                │                                        
                                                    └───────┬────────┘                                        
                                                            ▼                                                 
                                              ✅ Finished iteration → наступна вакансія                 
                                └─────────────────────────────────────────────────────────────────────┘
```

## Можливості

**Пошук та збір:**
- Автоматичний пошук за 6 ключовими словами (React Native, Full Stack, Node.js, JavaScript, TypeScript, Front End)
- Gmail-тригер для миттєвої обробки LinkedIn Job Alert листів
- Дедуплікація вакансій між запусками (пам'ятає оброблені jobId)
- Дедуплікація email-листів (не обробляє один лист двічі)
- Keyword matching — автоматичне порівняння стеку вакансії з вашим профілем

**AI-аналіз:**
- Оцінка релевантності вакансії від 0 до 100 через LLM
- Генерація персоналізованого супровідного листа українською (для score ≥ 80)
- Аналіз резюме — прогалини навичок, ATS-сумісність, рекомендації
- Переклад та нормалізація всіх повідомлень українською

**Сповіщення:**
- Telegram — HTML-формат із клікабельними посиланнями
- Slack — Markdown-формат для командного каналу
- Включає: позицію, компанію, зарплату, досвід, кількість заявок, збіг стеку, аналіз резюме, дії для покращення

**Фільтрація:**
- Пре-фільтр по keyword match (< 33% → пропускається без LLM-виклику, економить токени)
- Score-based фільтр (< 80 → без CV-аналізу та сповіщень)

## Встановлення

**Передумови:**
- n8n інстанс (self-hosted або n8n cloud)
- Акаунти: Apify, Google Cloud (Gemini API), Telegram Bot, Slack App, Gmail

**Імпорт workflow:**

1. Скопіюйте файл `job_detector.json` з цього репозиторію
2. У n8n перейдіть до **Settings → Import from File**
3. Виберіть файл `job_detector.json`
4. Workflow з'явиться у вашому списку як "Job Detector"

Або через CLI:
```bash
# Якщо використовуєте n8n CLI
n8n import:workflow --input=job_detector.json
```

## Налаштування облікових даних

Після імпорту потрібно налаштувати credentials для кожного сервісу. Відкрийте workflow і пройдіть по нодах з помаранчевим індикатором (відсутні credentials).

### 1. Apify OAuth2 API

Використовується у 3 нодах: **Find LinkedIn Jobs**, **Find Alerted LinkedIn Jobs**, **Find CV suggestions**.

1. Зареєструйтесь на [apify.com](https://apify.com)
2. Перейдіть до **Settings → Integrations → API tokens**
3. Створіть новий токен
4. У n8n: **Credentials → New → Apify OAuth2 API** → вставте токен

**Apify актори, що використовуються:**
- `2rJKkhh7vjpX7pvjg` — LinkedIn Jobs Scraper (пошук вакансій по ключовим словам, є пошук і по URL, але він погано працює)
- `HxVzxLvPwxYkc6ndm` — LinkedIn Jobs Scraper - Professional Job Listings (отримати деталі вакансій пошук по URL)
- `dylMGHNi91mnRsuqB` — AI Resume Gap Analyzer (оцінити match вакансії та резюме)

### 2. Google Gemini (LLM)

Використовується у 2 нодах: **Google Gemini Chat Model**, **Google Gemini Chat Model1**.

1. Перейдіть до [Google AI Studio](https://aistudio.google.com/apikey)
2. Створіть API key
3. У n8n: **Credentials → New → Google Gemini API** → вставте ключ
4. Модель: `gemini-3.1-flash-lite` (швидка та дешева, але за бажанням можна обрати іншу)

### 3. Telegram Bot

Використовується у ноді **Send a text message**.

1. Напишіть [@BotFather](https://t.me/BotFather) у Telegram
2. Створіть нового бота: `/newbot`
3. Скопіюйте токен бота
4. Дізнайтесь свій Chat ID — напишіть [@userinfobot](https://t.me/userinfobot)
5. У n8n: **Credentials → New → Telegram API** → вставте токен
6. У ноді **Send a text message** замініть `chatId` на свій

### 4. Slack

Використовується у ноді **Send Notification to Slack**.

1. Створіть Slack App на [api.slack.com/apps](https://api.slack.com/apps)
2. Додайте OAuth Scope: `chat:write`, `chat:write.public`
3. Встановіть App у свій workspace
4. Скопіюйте Bot User OAuth Token
5. У n8n: **Credentials → New → Slack OAuth2 API** → авторизуйтесь
6. У ноді замініть канал `automation-alerts` на свій

### 5. Gmail OAuth2

Використовується у ноді **Received LinkedIn Job Alert**.

1. Створіть OAuth2 credentials у [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Увімкніть Gmail API
3. У n8n: **Credentials → New → Gmail OAuth2 API** → пройдіть OAuth flow
4. Переконайтесь, що у вас є LinkedIn Job Alerts на пошті

## Конфігурація під себе

### Ключові слова пошуку

У ноді **Find LinkedIn Jobs** → `customBody` → масив `keyword`:
```json
"keyword": [
    "React Native",
    "Full Stack",
    "Node.js",
    "JavaScript",
    "TypeScript",
    "Front End"
]
```

### Локація

Там же, поле `location`:
```json
"location": "Kyiv, Ukraine"
```

### Resume Keywords (для keyword matching)

Масив `resumeKeywords` у тому ж ноді — додайте свої ключові навички:
```json
"resumeKeywords": [
    { "keyword": "JavaScript", "aliases": ["JS"] },
    { "keyword": "TypeScript", "aliases": ["TS"] },
    { "keyword": "Python" }
]
```

### Резюме кандидата

У ноді **Find CV suggestions** → `customBody` → поле `resumeText` — замініть на свій текст резюме.

У ноді **LLM: calculate job relevance** → `text` → секція "Candidate Background" — замініть на свій опис.

### Пороги фільтрації

- **Skip irrelevant** (нод Switch): `keywordMatchScorePercentage >= 33` — мінімальний збіг ключових слів для обробки LLM
- **SMS Gateway** (нод IF): `closeness_score >= 80` — мінімальний score для CV-аналізу та сповіщень

### Gmail фільтр

У ноді **Received LinkedIn Job Alert** → `filters.q`:
```
subject:(React OR "React Native" OR "Node.js" OR "Full Stack" OR Frontend OR JavaScript OR TypeScript)
```

Додайте свої ключові слова пошуку LinkedIn Alerts.

## Вартість

| Сервіс | Вартість |
|--------|----------|
| LinkedIn Jobs Scraper | від $0.60 / 1,000 вакансій | Приблизно ~20-30 вакансій за запит |
| AI Resume Gap Analyzer | від $0.01 / 1,000 запитів |
| LinkedIn Jobs Scraper - Professional Job Listings | від $2 / 1,000 вакансій | Поставлено ліміт в одну вакансію за запит |
| Google Gemini Flash Lite | безкоштовно (free tier) |
| Telegram / Slack | безкоштовно |
| Gmail | безкоштовно |

При 6 запусках на добу (кожні 4 години): **~$0.10/день** або **~$3/місяць** на Apify.

Для зменшення витрат: зменшіть кількість ключових слів, збільшіть інтервал тригера, або використовуйте `publishedAt: "r43200"` (12 годин замість 24).

<img width="1684" height="777" alt="image" src="https://github.com/user-attachments/assets/5a40f3c1-fd32-47e0-b4a8-859c4d3115c4" />


# 📰 News Feed Automation

Автоматизований n8n-воркфлоу, який щоранку збирає статті з RSS-стрічок, створює короткі українськомовні анотації за допомогою Google Gemini і публікує їх у Telegram-канал із зображеннями.

---

## Архітектура

```
┌───────────────┐      ┌───────────┐      ┌──────────┐       ┌─────────────┐
   Schedule       ───▶   URL List   ───▶   Split Out   ───▶    RSS Read   
   (10:00 AM)            (9 feeds)        └──────────┘       └──────┬──────┘                        
└───────────────┘      └───────────┘                                |
                                                                    │
                    ┌───────────────────────────────────────────────┘
                    ▼
             ┌─────────────┐        ┌───────────┐        ┌──────────┐      ┌──────────────────┐
               Filter fresh   ───▶    Randomize   ───▶    Limit 15   ───▶  Remove Duplicates
               (this year)          └───────────┘        └───────────┘       (cross-execution)
             └─────────────┘                                               └───────┬──────────┘
                                                                                   │
                    ┌──────────────────────────────────────────────────────────────┘
                    ▼
          ┌──────────────────┐      ┌──────────────────┐       ┌─────────────┐
            Loop Over Items    ───▶     Wait 5s (rate    ───▶   Gemini 2.5  
             (batch by 1)               limit between             Flash LLM   
          └────────┬─────────┘           iterations)             (summarize) 
                   |                └────────┬─────────┘       └──────┬──────┘
                   ▲                                                  │
                   │                                                  ▼
          ┌────────┴─────────┐                                ┌──────────────┐
             Send Photo       ◀─────────────────────────────     Map Images  
             to Telegram                                         (fallbacks) 
          └──────────────────┘                                └──────────────┘
```

---

## Що робить воркфлоу

1. **Тригер за розкладом** — запускається щодня о 10:00.
2. **Збір RSS** — обходить 9 RSS-стрічок (tech, AI, українські медіа).
3. **Фільтрація** — залишає лише статті поточного року.
4. **Рандомізація + ліміт** — перемішує та обрізає до 15 статей.
5. **Дедуплікація** — прибирає статті, які вже надсилались у попередніх запусках (за ключем `creator + link`).
6. **LLM-саммарі** — Gemini 2.5 Flash генерує коротку анотацію українською (до 1024 символів, Telegram HTML).
7. **Підбір зображення** — бере `enclosure.url` зі статті або підставляє фолбек-логотип за доменом.
8. **Публікація** — надсилає фото з підписом у Telegram-чат.
9. **Rate limiting** — між ітераціями 5-секундна пауза, щоб не потрапити в ліміти API.

---

## RSS-джерела

| Джерело | URL |
|---------|-----|
| Hacker News (frontpage) | `https://hnrss.org/frontpage` |
| DEV.to | `https://dev.to/feed/` |
| OpenAI Blog | `https://openai.com/news/rss.xml` |
| Google DeepMind Blog | `https://deepmind.google/blog/rss.xml` |
| CSS-Tricks | `https://css-tricks.com/feed/` |
| AIN.UA | `https://ain.ua/feed/` |
| ITPro | `https://www.itpro.com/feeds.xml` |
| ITC.UA | `https://itc.ua/ua/feed/` |
| УНІАН | `https://rss.unian.net/site/news_ukr.rss` |

---

## Вимоги

- **n8n** — self-hosted або n8n Cloud
- **Google Gemini API** — ключ для моделі `gemini-2.5-flash`
- **Telegram Bot** — токен бота + ID чату для публікації

---

## Налаштування

### 1. Імпорт воркфлоу

1. Відкрийте n8n → **Workflows** → **Import from File**.
2. Завантажте JSON-файл воркфлоу.

### 2. Credentials

Потрібно створити два credentials у n8n:

**Google Gemini:**
- Перейдіть до **Settings** → **Credentials** → **New Credential**.
- Тип: `Google Gemini Chat Model`.
- Вставте ваш API-ключ Google AI Studio.

**Telegram Bot:**
- Тип: `Telegram API`.
- Вставте токен бота (отримайте через [@BotFather](https://t.me/BotFather)).

### 3. Налаштування чату Telegram

У ноді **"Send a post to TG"** замініть `chatId` на ID вашого каналу або чату

### 4. Кастомізація джерел

Відредагуйте масив URL-адрес у ноді **"Url sources"**. Додайте або видаліть RSS-стрічки за потреби.

### 5. Активація

Увімкніть воркфлоу тумблером — він запускатиметься автоматично щодня о 10:00.

---

## Структура нод

| Нода | Тип | Призначення |
|------|-----|-------------|
| Call every morning (10AM) | Schedule Trigger | Щоденний запуск |
| Url sources | Set | Масив RSS-стрічок |
| Split Out | Split Out | Розгортає масив в окремі items |
| RSS Read | RSS Feed Read | Зчитує статті з кожного RSS |
| Pick fresh posts | Filter | Фільтрує за поточним роком |
| Randomize | Sort (random) | Перемішує статті |
| Limit to 15 | Limit | Обмежує кількість |
| Remove Duplicates | Remove Duplicates | Виключає раніше надіслані |
| Loop Over Items | Split In Batches | Обробка по одній статті |
| If not the first iteration | If | Пропускає паузу для першої |
| Wait 5 seconds | Wait | Пауза між публікаціями |
| LLM: process a post | Chain LLM | Генерація анотації (Gemini) |
| Google Gemini Chat Model | LM Chat | Модель для LLM-ноди |
| Map images | Code | Вибір зображення або фолбеку |
| Send a post to TG | Telegram | Публікація в Telegram |

---

## Формат повідомлення у Telegram

Кожна публікація надсилається як фото з HTML-підписом:

```
<b>Заголовок статті</b>

<i>Автор:</i> Ім'я
<i>Дата:</i> 17.05.2026
<i>Категорія:</i> AI, Machine Learning

Коротка анотація українською — 2-3 речення
про що стаття та чому може бути корисна.

<a href="https://...">Читати повністю</a>
```

---

## Фолбек-зображення

Якщо стаття не містить `enclosure.url`, підставляється логотип джерела:

| Домен | Фолбек |
|-------|--------|
| openai.com | Логотип OpenAI |
| deepmind.google | Логотип DeepMind |
| ain.ua | Логотип AIN |
| itc.ua | Логотип ITC |
| dev.to | Логотип DEV |
| css-tricks.com | Логотип CSS-Tricks |
| itpro.com | Логотип ITPro |
| unian.net | Логотип УНІАН |
| (інше) | Стокова картинка новин |

---

## Можливі проблеми

- **RSS недоступний** — нода RSS Read має `onError: continueRegularOutput`, тому один збій не зупиняє весь воркфлоу.
- **Gemini rate limit** — 5-секундна пауза між ітераціями мінімізує ризик, але при великій кількості публікацій збільшіть паузу.
- **Telegram rate limit** — Telegram обмежує ботів до ~30 повідомлень на секунду; 15 повідомлень з паузами не повинні викликати проблем.
- **Фолбек-зображення недоступне** — замініть URL у ноді "Map images" на альтернативне.

---

## Ліцензія

MIT — використовуйте, модифікуйте, діліться вільно.
