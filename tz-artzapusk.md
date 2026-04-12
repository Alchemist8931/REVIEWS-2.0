# Техническое задание: «АртЗапуск» — CRM-система управления доставкой отзывов

**Версия:** 2.0  
**Дата:** 12 апреля 2026 г.  
**Платформа:** PWA (Progressive Web Application)  
**Стек:** HTML/CSS/JS (Vanilla) + Tailwind CSS + Supabase (PostgreSQL + Auth + Storage + Edge Functions + Realtime)

---

## 1. ОБЩЕЕ ОПИСАНИЕ ПРОЕКТА

### 1.1 Назначение
CRM-приложение для компании, оказывающей услуги по доставке отзывов на различных площадках (Яндекс Карты, 2ГИС, Авито, Профи.ру, фриланс-площадки и др.). Приложение автоматизирует полный цикл сделки: от создания карточки клиента до выполнения задания автором с имитацией естественного поведения на площадке.

### 1.2 Ключевые принципы
- **Single-Page Application** — вся навигация без перезагрузки страницы
- **PWA** — установка на мобильные устройства, offline-ready shell, push-уведомления
- **Responsive Design** — адаптивный интерфейс для Desktop (1024px+), Tablet (768px–1023px), Mobile (320px–767px)
- **Real-time** — Supabase Realtime для мгновенных обновлений статусов между ролями
- **Action Logging** — каждое действие каждого пользователя логируется в таблицу `action_logs`

### 1.3 Дизайн-система
Светлая тема, вдохновлённая приложённым примером Alchemist Cloud (инвертированная — светлая):

```
ПАЛИТРА (СВЕТЛАЯ ТЕМА):
--bg:             #F8F9FA    (основной фон)
--bg-secondary:   #FFFFFF    (карточки, модалки)
--bg-tertiary:    #F0F1F3    (вложенные блоки, инпуты)
--fg:             #1A1A1A    (основной текст)
--fg-secondary:   #6B7280    (вторичный текст)
--fg-muted:       #9CA3AF    (приглушённый текст)
--muted:          #D1D5DB    (разделители, бордеры)
--card:           #FFFFFF    (фон карточек)
--border:         #E5E7EB    (бордеры)
--border-light:   #F3F4F6    (лёгкие бордеры)
--accent:         #111827    (акцент, кнопки)
--accent-hover:   #374151    (ховер акцент)
--success:        #10B981    (успех)
--warning:        #F59E0B    (предупреждение)
--error:          #EF4444    (ошибка)
--info:           #3B82F6    (информация)

ШРИФТ: 'SN Pro', sans-serif
СКРУГЛЕНИЯ: 12px (кнопки), 16px (карточки), 20px (модалки)
ТЕНИ: box-shadow: 0 1px 3px rgba(0,0,0,0.05) (карточки), 0 20px 40px rgba(0,0,0,0.1) (модалки)
```

**Элементы из примера (адаптировать в светлую тему):**
- Corner-marks (декоративные уголки на карточках авторизации)
- Blueprint-grid фон (очень лёгкий, едва заметный паттерн)
- Плавные анимации появления (reveal, fade-in, slide-in)
- Scan-line анимация на экране авторизации
- Glass-кнопки (dark glass — тёмные полупрозрачные кнопки на светлом фоне)
- Toast-уведомления (нижний правый угол)
- Status-pulse анимация для индикаторов
- Pin-dots стиль (для индикации шагов/этапов)

---

## 2. РОЛИ И ПРАВА ДОСТУПА

### 2.1 Роли

| Роль | Код в БД | Описание |
|------|----------|----------|
| Супер Админ | `super_admin` | Полный доступ ко всем данным и функциям. Управление пользователями, просмотр всей статистики, управление настройками |
| Менеджер клиента | `manager` | Создаёт сделки, управляет отзывами, взаимодействует с клиентами и авторами, видит свои сделки и назначенных авторов |
| Клиент | `client` | Заполняет бриф, согласовывает/корректирует отзывы, видит статус своих сделок и задач |
| Автор | `author` | Берёт задания в работу, выполняет 3-шаговый процесс оставления отзыва, загружает скриншоты подтверждения |

### 2.2 Матрица доступа

| Функция / Страница | Super Admin | Менеджер | Клиент | Автор |
|---------------------|:-----------:|:--------:|:------:|:-----:|
| Дашборд (общая статистика) | ✅ | ✅ (свои) | ❌ | ❌ |
| Список сделок | ✅ (все) | ✅ (свои) | ✅ (свои) | ❌ |
| Создание сделки | ✅ | ✅ | ❌ | ❌ |
| Карточка сделки (полная) | ✅ | ✅ (свои) | ✅ (частично) | ❌ |
| Бриф клиента | ✅ (просмотр) | ✅ (просмотр) | ✅ (заполнение) | ❌ |
| Создание текстовок отзывов | ✅ | ✅ | ❌ | ❌ |
| Согласование отзывов | ✅ | ❌ | ✅ | ❌ |
| Назначение задач авторам | ✅ | ✅ | ❌ | ❌ |
| Доска задач (Kanban) | ❌ | ❌ | ❌ | ✅ |
| Мои задания (автор) | ❌ | ❌ | ❌ | ✅ |
| Управление пользователями | ✅ | ❌ | ❌ | ❌ |
| Логи действий | ✅ | ❌ | ❌ | ❌ |
| Настройки профиля | ✅ | ✅ | ✅ | ✅ |
| Уведомления | ✅ | ✅ | ✅ | ✅ |

---

## 3. СТРУКТУРА БАЗЫ ДАННЫХ (SUPABASE / POSTGRESQL)

### 3.1 Таблица `users`
Основная таблица пользователей (заменяет `reviews_users`).

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,       -- bcrypt hash (хешировать через Edge Function)
  role VARCHAR(20) NOT NULL CHECK (role IN ('super_admin', 'manager', 'client', 'author')),
  status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'blocked')),
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  middle_name VARCHAR(100),
  phone VARCHAR(20),
  city VARCHAR(100),
  avatar_url TEXT,
  must_change_password BOOLEAN DEFAULT false, -- для авто-созданных клиентов
  last_login TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Индексы
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_status ON users(status);
```

### 3.2 Таблица `counterparties` (контрагенты)
```sql
CREATE TABLE counterparties (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  inn VARCHAR(12) UNIQUE NOT NULL,
  opf VARCHAR(20),                           -- ОПФ: ИП, ООО, АО и т.д.
  name_company VARCHAR(500) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_counterparties_inn ON counterparties(inn);
```

### 3.3 Таблица `clients` (карточки клиентов)
```sql
CREATE TABLE clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE SET NULL, -- связь с аккаунтом (если создан)
  counterparty_id UUID REFERENCES counterparties(id),
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  middle_name VARCHAR(100),
  phone VARCHAR(20),
  email VARCHAR(255),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_clients_user_id ON clients(user_id);
CREATE INDEX idx_clients_email ON clients(email);
```

### 3.4 Таблица `deals` (сделки)
```sql
CREATE TABLE deals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  deal_number SERIAL,                        -- автоинкремент номер сделки
  manager_id UUID NOT NULL REFERENCES users(id),  -- менеджер-создатель
  client_id UUID NOT NULL REFERENCES clients(id),
  counterparty_id UUID REFERENCES counterparties(id),
  
  -- Договор
  contract_number VARCHAR(100),
  contract_date DATE,
  sum_deal NUMERIC(12,2) DEFAULT 0,
  
  -- Статус сделки (этапы)
  status VARCHAR(30) NOT NULL DEFAULT 'brief' 
    CHECK (status IN ('brief', 'brief_completed', 'review_creation', 'approval', 'in_work', 'completed', 'cancelled')),
  
  -- Бриф
  brief_data JSONB,                          -- ответы на бриф
  brief_submitted_at TIMESTAMPTZ,
  
  -- Документы (массивы ссылок)
  documents_agreement JSONB DEFAULT '[]'::jsonb,   -- [{name, url, path, uploaded_at}]
  documents_closing JSONB DEFAULT '[]'::jsonb,
  
  -- Комментарий
  help_text TEXT,
  
  -- Метаданные
  is_new_client BOOLEAN DEFAULT true,        -- флаг: новый ли клиент
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_deals_manager_id ON deals(manager_id);
CREATE INDEX idx_deals_client_id ON deals(client_id);
CREATE INDEX idx_deals_status ON deals(status);
CREATE INDEX idx_deals_created ON deals(created_at DESC);
```

### 3.5 Таблица `reviews` (отзывы = задания)
```sql
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  deal_id UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
  
  -- Контент
  review_text TEXT NOT NULL,                 -- текст отзыва
  review_text_original TEXT,                 -- оригинальный текст (до правки клиентом)
  platform_link TEXT,                        -- URL площадки
  
  -- Статусы
  status VARCHAR(30) NOT NULL DEFAULT 'draft'
    CHECK (status IN (
      'draft',              -- черновик (менеджер создал)
      'pending_approval',   -- отправлен клиенту на согласование
      'client_edited',      -- клиент отредактировал, ждёт подтверждения менеджера
      'approved',           -- согласован (готов к работе)
      'available',          -- опубликован для авторов (можно брать)
      'in_progress',        -- взят автором в работу
      'contact_made',       -- автор сделал первый контакт
      'review_posted',      -- автор оставил отзыв (ждёт 48ч)
      'completed',          -- задача выполнена
      'cancelled'           -- отменена
    )),
  
  -- Назначение автору
  assigned_author_id UUID REFERENCES users(id),
  assigned_at TIMESTAMPTZ,
  
  -- Шаги автора
  contact_screenshot_url TEXT,               -- скриншот первого контакта
  contact_made_at TIMESTAMPTZ,               -- время первого контакта
  contact_pause_until TIMESTAMPTZ,           -- до какого времени пауза после контакта (48ч)
  
  review_screenshot_url TEXT,                -- скриншот опубликованного отзыва
  review_posted_at TIMESTAMPTZ,              -- время публикации отзыва
  review_pause_until TIMESTAMPTZ,            -- до какого времени пауза после отзыва (48ч)
  
  completed_at TIMESTAMPTZ,                  -- время закрытия задачи
  
  -- Дедлайн
  deadline DATE,
  
  -- Метаданные
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_reviews_deal_id ON reviews(deal_id);
CREATE INDEX idx_reviews_status ON reviews(status);
CREATE INDEX idx_reviews_assigned ON reviews(assigned_author_id);
CREATE INDEX idx_reviews_available ON reviews(status) WHERE status = 'available';
```

### 3.6 Таблица `action_logs` (лог действий)
```sql
CREATE TABLE action_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  user_role VARCHAR(20),
  action_type VARCHAR(100) NOT NULL,         -- тип действия (см. справочник ниже)
  entity_type VARCHAR(50),                   -- тип сущности: deal, review, user, client
  entity_id UUID,                            -- ID сущности
  details JSONB,                             -- дополнительные данные
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_action_logs_user ON action_logs(user_id);
CREATE INDEX idx_action_logs_type ON action_logs(action_type);
CREATE INDEX idx_action_logs_entity ON action_logs(entity_type, entity_id);
CREATE INDEX idx_action_logs_created ON action_logs(created_at DESC);
```

**Справочник action_type:**
```
-- Авторизация
user.login, user.logout, user.register, user.password_change

-- Сделки
deal.create, deal.update, deal.status_change, deal.document_upload, deal.document_delete

-- Бриф
brief.submit, brief.view

-- Отзывы
review.create, review.edit, review.send_for_approval, review.approve, review.reject, 
review.client_edit, review.send_to_authors, review.publish_for_authors

-- Задания автора
task.take, task.contact_upload, task.review_upload, task.complete

-- Пользователи
user.create, user.activate, user.block, user.role_change

-- Уведомления
notification.send, notification.read
```

### 3.7 Таблица `notifications` (уведомления)
```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type VARCHAR(50) NOT NULL,                 -- тип уведомления
  title VARCHAR(255) NOT NULL,
  message TEXT,
  link TEXT,                                 -- ссылка для перехода в приложении
  is_read BOOLEAN DEFAULT false,
  related_entity_type VARCHAR(50),
  related_entity_id UUID,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read);
CREATE INDEX idx_notifications_created ON notifications(created_at DESC);
```

### 3.8 Таблица `brief_templates` (шаблоны брифов)
```sql
CREATE TABLE brief_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(200) NOT NULL,
  sections JSONB NOT NULL,  -- [{title, questions: [{label, type: 'text'|'textarea'|'checkbox_group', options?: [...]}]}]
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### 3.9 Таблица `platforms` (справочник площадок)
```sql
CREATE TABLE platforms (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,                -- Яндекс Карты, 2ГИС, Авито и т.д.
  icon_url TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### 3.10 RLS (Row Level Security)
```sql
-- Включить RLS на всех таблицах
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE deals ENABLE ROW LEVEL SECURITY;
ALTER TABLE reviews ENABLE ROW LEVEL SECURITY;
ALTER TABLE clients ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;
ALTER TABLE action_logs ENABLE ROW LEVEL SECURITY;

-- Примеры политик (реализовать для каждой таблицы):

-- Super Admin видит всё
CREATE POLICY "super_admin_all" ON deals FOR ALL 
  USING (EXISTS (SELECT 1 FROM users WHERE id = auth.uid() AND role = 'super_admin'));

-- Менеджер видит свои сделки
CREATE POLICY "manager_own_deals" ON deals FOR ALL
  USING (manager_id = auth.uid());

-- Клиент видит свои сделки
CREATE POLICY "client_own_deals" ON deals FOR SELECT
  USING (client_id IN (SELECT id FROM clients WHERE user_id = auth.uid()));

-- Автор видит доступные и свои задания
CREATE POLICY "author_reviews" ON reviews FOR SELECT
  USING (status = 'available' OR assigned_author_id = auth.uid());
```

### 3.11 Supabase Storage Buckets
```
deals-documents/      — документы сделок (agreement, closing)
  {deal_id}/agreement/
  {deal_id}/closing/

task-attachments/     — скриншоты авторов
  {review_id}/contact/
  {review_id}/review/

avatars/              — аватары пользователей
  {user_id}/
```

---

## 4. АВТОРИЗАЦИЯ И РЕГИСТРАЦИЯ

### 4.1 Экран авторизации
Дизайн в стиле Alchemist Cloud (светлая тема):
- Центрированная карточка с corner-marks
- Blueprint-grid фон (едва заметный)
- Scan-line анимация
- Reveal-анимации при появлении

**Вход:**
- Поля: Email (или телефон), Пароль
- Кнопка «Войти» (glass-btn стиль)
- Ссылка «Забыли пароль?» (через Supabase Auth reset)

**Регистрация (только для Авторов и Менеджеров):**
- Табы: Вход / Регистрация (radio-inputs стиль)
- При регистрации: подтаб выбора роли (Автор / Менеджер)
- Поля: Фамилия, Имя, Отчество, Город, Телефон, Email, Пароль
- После регистрации: статус `pending`, баннер «Ожидайте подтверждения администратором»
- Super Admin подтверждает через панель управления

**Клиенты НЕ регистрируются сами** — их создаёт Менеджер при создании сделки:
- Если клиент новый → создаётся аккаунт с auto-generated паролем, на email уходит уведомление с логином/паролем + флаг `must_change_password = true`
- Если клиент существует → на email уходит уведомление о новой сделке

### 4.2 Принудительная смена пароля
При первом входе клиента (если `must_change_password = true`):
- Модальное окно с полями «Новый пароль» и «Подтверждение пароля»
- Нельзя закрыть без смены пароля
- После смены → `must_change_password = false`

### 4.3 Логирование
При каждом входе/выходе → запись в `action_logs`:
```json
{ "action_type": "user.login", "user_id": "...", "details": { "method": "email" } }
```

---

## 5. СТРУКТУРА ПРИЛОЖЕНИЯ (APP SHELL)

### 5.1 Общий Layout
```
┌──────────────────────────────────────────────────────┐
│ HEADER: Логотип | Навигация (desktop) | Уведом. | Профиль │
├──────────────┬───────────────────────────────────────┤
│   SIDEBAR    │                                       │
│   (навигация │         ОСНОВНОЙ КОНТЕНТ              │
│    иконки)   │         (страницы)                    │
│              │                                       │
│   Desktop:   │                                       │
│   84px fixed │                                       │
│              │                                       │
│   Mobile:    │                                       │
│   hamburger  │                                       │
│   overlay    │                                       │
└──────────────┴───────────────────────────────────────┘
```

### 5.2 Навигация по ролям

**Super Admin:**
1. 📊 Дашборд (статистика по всем)
2. 📋 Сделки (все сделки)
3. ✍️ Отзывы (все отзывы/задания)
4. 👥 Пользователи (управление)
5. 📈 Аналитика (логи, счётчики)
6. ⚙️ Настройки

**Менеджер:**
1. 📊 Дашборд (мои показатели)
2. 📋 Мои сделки
3. ✍️ Задания (отзывы моих сделок)
4. ⚙️ Настройки профиля

**Клиент:**
1. 📋 Мои сделки
2. 📝 Бриф (если нужно заполнить)
3. ✍️ Согласование отзывов
4. 📊 Статус задач
5. ⚙️ Настройки профиля

**Автор:**
1. 📋 Доступные задания (Kanban-доска)
2. 🔧 Мои задания (взятые в работу)
3. 📊 Мои результаты
4. ⚙️ Настройки профиля

---

## 6. БИЗНЕС-ЛОГИКА: СЦЕНАРИЙ СДЕЛКИ

### 6.1 Этапы сделки (Timeline)

```
[1. БРИФ] → [2. СОЗДАНИЕ ОТЗЫВОВ] → [3. СОГЛАСОВАНИЕ] → [4. В РАБОТЕ] → [5. ВЫПОЛНЕНО]
```

**Визуализация:** прогресс-бар с 5 этапами (t-bar стиль из текущего кода), кнопки действий под каждым этапом.

### 6.2 Этап 1: Создание сделки и Бриф

**Менеджер создаёт сделку:**
1. Нажимает «Создать сделку» → модальное окно
2. Заполняет:
   - Блок «Контрагент»: ИНН (автоподбор из `counterparties`), ОПФ, Наименование
   - Блок «Клиент»: Фамилия (автоподбор из `clients`), Имя, Отчество, Телефон, Email
   - Блок «Договор»: Номер, Дата, Сумма, Комментарий
3. При сохранении:
   - Если клиент новый → создаётся запись в `clients` + аккаунт в `users` (role=client, must_change_password=true)
   - Отправляется email (через Supabase Edge Function):
     - Новый клиент: «Вам создана сделка. Логин: {email}, Пароль: {auto_pass}. Необходимо заполнить бриф.»
     - Существующий клиент: «Вам создана новая сделка #{deal_number}.»
   - Сделка создаётся со статусом `brief` (если клиент новый) или `review_creation` (если существующий)

**Клиент заполняет бриф:**
1. Клиент входит → видит сделку со статусом «Требуется заполнить бриф»
2. Нажимает «Пройти бриф» → модальное окно с формой
3. Секции брифа (из `brief_templates` или зашитые):
   - Общие сведения (стаж, специализация, ресурсы, ЦА)
   - Результаты и преимущества (кейсы, конкуренты)
   - Условия сотрудничества (стоимость, формат, консультации)
   - Доп. информация (персоналии, пожелания к содержанию)
   - Присутствие на площадках (чекбоксы)
4. После отправки:
   - `deals.brief_data` = JSON с ответами
   - `deals.brief_submitted_at` = now()
   - `deals.status` = `brief_completed`
   - Уведомление менеджеру: «Клиент {ФИО} заполнил бриф по сделке #{deal_number}»
   - Лог: `brief.submit`

### 6.3 Этап 2: Создание текстовок отзывов

**Менеджер создаёт отзывы:**
1. В карточке сделки менеджер видит этап «Создание отзывов» → кнопка «Создать отзывы»
2. Открывается модальное окно (task modal):
   - Поля: Ссылка на площадку (URL), Количество отзывов, Дедлайн
   - Textarea: текст отзыва
   - Кнопка «Создать отзыв» → анимация «стирания» текста из textarea + появление карточки отзыва в левом окне «Черновики»
3. Менеджер создаёт N отзывов (каждый — отдельная запись в `reviews` со статусом `draft`)
4. Два окна:
   - **Левое «Черновики / На мне»**: карточки со статусом `draft` и `approved` (согласованные клиентом)
   - **Правое «У клиента»**: карточки со статусом `pending_approval`
5. Кнопка «Отправить на согласование» — переводит все `draft` → `pending_approval`
6. При переводе → уведомление клиенту
7. `deals.status` = `approval`

### 6.4 Этап 3: Согласование клиентом

**Клиент согласовывает отзывы:**
1. Клиент видит сделку со статусом «Ожидают согласования»
2. Нажимает «Согласовать отзывы» → модальное окно
3. Два окна:
   - **Левое «На согласовании»**: карточки отзывов
   - **Правое «Согласовано»**: утверждённые
4. Для каждого отзыва клиент может:
   - **Согласовать** → `reviews.status` = `approved`
   - **Изменить** → редактирование текста → сохранение (оригинал сохраняется в `review_text_original`, новый текст в `review_text`, статус = `client_edited`)
5. Отредактированные отзывы уходят менеджеру на подтверждение
6. Менеджер видит правки клиента, может:
   - Принять → `approved`
   - Вернуть на доработку клиенту → `pending_approval`

**Когда все отзывы согласованы:**
- Кнопка «Отправить в работу авторам» становится доступной
- При нажатии: все `approved` → `available`
- `deals.status` = `in_work`
- Логи, уведомления

### 6.5 Этап 4: Работа авторов

**КРИТИЧЕСКИ ВАЖНОЕ БИЗНЕС-ПРАВИЛО:**
> Один автор может взять только ОДИН отзыв в работу по одной и той же ссылке (`platform_link`). Это обеспечивает уникальность аккаунтов на площадке и предотвращает бан за накрутку.

**Проверка при взятии задания:**
```sql
-- Проверить, что автор ещё не брал задание по этой ссылке
SELECT COUNT(*) FROM reviews 
WHERE assigned_author_id = {author_id} 
AND platform_link = {platform_link}
AND status NOT IN ('cancelled');
-- Если > 0 → отказ с сообщением «Вы уже работали с этой площадкой»
```

**Автор видит доступные задания:**
- Kanban-доска или список карточек с фильтрацией
- Каждая карточка: название площадки, текст отзыва (первые 100 символов), дедлайн, контакты менеджера
- Кнопка «Взять в работу» (только если проходит проверку уникальности)

**3-шаговый процесс выполнения:**

**Шаг 1: Первый контакт**
- Автор видит: URL площадки, полный текст отзыва, контакты менеджера-куратора
- Действие: автор переходит на площадку, пишет клиенту первое сообщение
- Загружает скриншот переписки → `reviews.contact_screenshot_url`
- При загрузке:
  - `reviews.status` = `contact_made`
  - `reviews.contact_made_at` = now()
  - `reviews.contact_pause_until` = now() + 48 часов
  - Уведомление клиенту: «Автор написал вам на площадке {platform_name}»
  - Клиент в ЛК видит индикатор «Вам написали на {platform_name}»

**Шаг 2: Пауза 48 часов**
- Автор видит обратный отсчёт (таймер)
- Кнопка перехода к Шагу 3 заблокирована до истечения таймера
- В это время автор и клиент имитируют общение на площадке

**Шаг 3: Публикация отзыва**
- После истечения 48ч → кнопка «Оставить отзыв» становится доступной
- Автор оставляет отзыв на площадке
- При загрузке скриншота:
  - `reviews.status` = `review_posted`
  - `reviews.review_posted_at` = now()
  - `reviews.review_pause_until` = now() + 48 часов
- **Вторая пауза 48 часов** перед закрытием

**Закрытие задачи:**
- После второй паузы → кнопка «Закрыть задачу» становится доступной
- ОБЯЗАТЕЛЬНО: загрузка скриншота опубликованного отзыва (валидация!)
- При закрытии:
  - `reviews.status` = `completed`
  - `reviews.completed_at` = now()
  - Уведомления менеджеру и клиенту
  - Лог: `task.complete`

### 6.6 Этап 5: Завершение сделки
- Когда все отзывы по сделке имеют статус `completed`:
  - `deals.status` = `completed`
  - Уведомление менеджеру и клиенту
  - Менеджер может загрузить закрывающие документы

---

## 7. СТРАНИЦЫ ПРИЛОЖЕНИЯ

### 7.1 Страница «Дашборд» (Менеджер / Super Admin)

**Счётчики (карточки-виджеты):**
- Всего сделок / Активных / Завершённых
- Отзывов в работе / На согласовании / Выполнено
- Авторов онлайн / Задач без исполнителя
- Выручка (сумма сделок за период)

**Графики:**
- Динамика сделок по месяцам (line chart)
- Распределение статусов (donut chart)
- Топ-5 площадок (bar chart)

**Фильтры:** период (сегодня, неделя, месяц, квартал, год, произвольный)

### 7.2 Страница «Сделки» (Менеджер / Super Admin)

**Список сделок** — карточки с:
- ID сделки, номер и дата договора
- ОПФ + Наименование, ИНН
- ФИО клиента, телефон
- Сумма
- Timeline-прогресс (5 этапов)
- Кнопки действий на текущем этапе
- По клику на timeline → раскрываются документы + история действий

**Фильтры:** по статусу, по менеджеру (для SA), по дате, поиск по ИНН/названию/ФИО

**Кнопка «Создать сделку»** (glass-btn)

### 7.3 Страница «Мои сделки» (Клиент)

**Список карточек:**
- Статус-чип (цветной): «Заполните бриф», «На согласовании», «В работе», «Завершено»
- Данные: организация, ИНН, номер договора, сумма
- Кнопки: «Пройти бриф», «Согласовать отзывы», «Посмотреть статус»
- Индикаторы: «Вам написали на площадке X» (когда автор сделал первый контакт)

### 7.4 Страница «Доступные задания» (Автор)

**Kanban-доска с колонками:**
1. **Доступные** — задания со статусом `available`
2. **Контакт** — `in_progress`, `contact_made`
3. **Отзыв** — `review_posted`
4. **Завершено** — `completed`

**Карточка задания:**
- Площадка (иконка + название)
- Текст отзыва (первые 100 символов, hover → полный)
- Дедлайн
- Контакты менеджера
- Кнопка «Взять» / Timeline прогресса / Таймер паузы

**Фильтры:** по площадке, по дедлайну

### 7.5 Страница «Пользователи» (Super Admin)

**Таблица/список пользователей:**
- ФИО, Email, Телефон, Роль, Статус, Дата регистрации, Последний вход
- Действия: Активировать / Заблокировать / Изменить роль / Удалить

**Фильтры:** по роли, по статусу, поиск

### 7.6 Страница «Аналитика / Логи» (Super Admin)

**Лог действий:**
- Таблица: Дата/время, Пользователь, Роль, Действие, Сущность, Детали
- Фильтры: по пользователю, по типу действия, по дате
- Экспорт в CSV

**Счётчики:**
- Действий за сегодня / неделю / месяц
- По типам действий
- По пользователям

### 7.7 Страница «Настройки профиля» (все роли)

- Редактирование: ФИО, телефон, город, аватар
- Смена пароля
- Настройки уведомлений (email вкл/выкл)

### 7.8 Уведомления (все роли)

**Иконка колокольчика в хедере** с badge-счётчиком непрочитанных.

**Dropdown / Страница уведомлений:**
- Список с группировкой «Сегодня / Вчера / Ранее»
- Типы: новая сделка, бриф заполнен, отзывы на согласование, отзыв согласован, задание взято, контакт совершён, задача выполнена
- Кнопка «Отметить все как прочитанные»
- Клик → переход к соответствующей сущности

---

## 8. УВЕДОМЛЕНИЯ (EMAIL)

### 8.1 Триггеры отправки email

| Событие | Получатель | Тема письма |
|---------|-----------|-------------|
| Создана сделка (новый клиент) | Клиент | «Ваш аккаунт в АртЗапуск: логин и пароль» |
| Создана сделка (существующий) | Клиент | «Новая сделка #{N} в АртЗапуск» |
| Бриф заполнен | Менеджер | «Клиент {ФИО} заполнил бриф» |
| Отзывы отправлены на согласование | Клиент | «Отзывы по сделке #{N} ожидают согласования» |
| Клиент отредактировал отзыв | Менеджер | «Клиент внёс правки в отзыв» |
| Все отзывы согласованы | Менеджер | «Все отзывы согласованы, можно передать в работу» |
| Задание взято автором | Менеджер | «Автор {ФИО} взял задание в работу» |
| Первый контакт совершён | Клиент | «Вам написали на площадке {platform}» |
| Задача завершена | Менеджер, Клиент | «Отзыв на площадке {platform} опубликован» |
| Сделка завершена | Менеджер, Клиент | «Сделка #{N} завершена» |
| Регистрация подтверждена | Пользователь | «Ваш аккаунт в АртЗапуск активирован» |

### 8.2 Реализация
Через **Supabase Edge Functions** (Deno) + SMTP или Supabase Auth emails.

---

## 9. REAL-TIME ОБНОВЛЕНИЯ

### 9.1 Supabase Realtime подписки

Использовать Supabase Realtime для мгновенных обновлений:

```javascript
// Подписка на изменения сделок (для менеджера)
supabase.channel('deals-changes')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'deals', filter: `manager_id=eq.${userId}` }, 
    payload => handleDealChange(payload))
  .subscribe();

// Подписка на уведомления
supabase.channel('notifications')
  .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'notifications', filter: `user_id=eq.${userId}` },
    payload => handleNewNotification(payload))
  .subscribe();

// Подписка на доступные задания (для авторов)
supabase.channel('available-tasks')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'reviews', filter: `status=eq.available` },
    payload => handleTaskUpdate(payload))
  .subscribe();
```

---

## 10. PWA

### 10.1 manifest.json
```json
{
  "name": "АртЗапуск — CRM управления отзывами",
  "short_name": "АртЗапуск",
  "description": "CRM-система управления доставкой отзывов",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#F8F9FA",
  "theme_color": "#111827",
  "orientation": "any",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### 10.2 Service Worker
- Кеширование App Shell (HTML, CSS, JS, шрифты, иконки)
- Стратегия: Network First для API, Cache First для статики
- Offline fallback: показ сообщения «Нет подключения к интернету»

### 10.3 Push-уведомления (опционально, фаза 2)
- Web Push API + Supabase Edge Functions
- Запрос разрешения при первом входе

---

## 11. АДАПТИВНОСТЬ (RESPONSIVE)

### 11.1 Breakpoints
```css
/* Mobile First */
@media (min-width: 768px)  { /* Tablet */ }
@media (min-width: 1024px) { /* Desktop */ }
@media (min-width: 1440px) { /* Wide Desktop */ }
```

### 11.2 Адаптация компонентов

| Компонент | Desktop | Tablet | Mobile |
|-----------|---------|--------|--------|
| Sidebar | Фиксированная, 84px | Hamburger overlay | Hamburger overlay |
| Header | Полная навигация | Сокращённая | Hamburger + логотип |
| Карточки сделок | Полная информация | Компактная | Стэк, свайп |
| Модальные окна | 600–900px, по центру | 90% ширины | Fullscreen |
| Kanban-доска | 4 колонки | 2 колонки | 1 колонка (табы) |
| Таблицы | Полные | Горизонтальный скролл | Карточки вместо таблиц |
| Timeline | Горизонтальный | Горизонтальный | Вертикальный |
| Формы | 2 колонки | 2 колонки | 1 колонка |

### 11.3 Touch-оптимизация
- Минимальный размер кнопок: 44×44px
- Swipe-жесты для Kanban (mobile)
- Pull-to-refresh
- Bottom sheet вместо модалок (mobile)

---

## 12. БЕЗОПАСНОСТЬ

### 12.1 Аутентификация
- Пароли хешируются через bcrypt (Supabase Edge Function)
- **НЕ хранить** plain-text пароли (в текущем коде `password_hash` содержит plain-text — исправить!)
- JWT-токены через Supabase Auth
- Сессии с автоматическим refresh

### 12.2 Авторизация
- RLS на всех таблицах
- Проверка роли в middleware (Edge Functions)
- Клиент не может видеть данные других клиентов
- Автор не может видеть данные сделок (только задания)

### 12.3 Защита данных
- HTTPS only
- CSP headers
- Sanitization всех пользовательских входов
- Rate limiting на API (через Supabase)
- Файлы: проверка MIME-type, максимальный размер 10MB

### 12.4 Скрытие credentials
- Supabase URL и anon key — через environment variables (не хардкод в HTML!)
- Или через Supabase Auth + RLS (anon key безопасен при правильном RLS)

---

## 13. ТЕХНИЧЕСКИЙ СТЕК И АРХИТЕКТУРА

### 13.1 Frontend
```
/
├── index.html              — SPA entry point + App Shell
├── manifest.json           — PWA manifest
├── sw.js                   — Service Worker
├── css/
│   ├── variables.css       — CSS-переменные, палитра
│   ├── base.css            — Сброс, типографика
│   ├── components.css      — Компоненты (кнопки, карточки, инпуты, модалки)
│   ├── layout.css          — Layout (header, sidebar, content)
│   ├── animations.css      — Анимации
│   └── responsive.css      — Адаптивность
├── js/
│   ├── app.js              — Инициализация, роутинг, состояние
│   ├── auth.js             — Авторизация / регистрация
│   ├── api.js              — Обёртки над Supabase
│   ├── realtime.js         — Realtime-подписки
│   ├── logger.js           — Логирование действий
│   ├── notifications.js    — Уведомления (in-app + push)
│   ├── router.js           — SPA-роутер (hash-based)
│   ├── utils.js            — Утилиты (форматирование, валидация)
│   └── pages/
│       ├── dashboard.js    — Дашборд
│       ├── deals.js        — Сделки
│       ├── deal-card.js    — Карточка сделки
│       ├── reviews.js      — Управление отзывами
│       ├── client-lk.js    — ЛК клиента
│       ├── author-board.js — Доска заданий автора
│       ├── author-task.js  — Задание автора
│       ├── users.js        — Управление пользователями
│       ├── analytics.js    — Аналитика / логи
│       └── settings.js     — Настройки
├── pages/                   — HTML-шаблоны страниц
│   ├── dashboard.html
│   ├── deals.html
│   ├── client-lk.html
│   ├── author-board.html
│   ├── users.html
│   ├── analytics.html
│   └── settings.html
└── icons/
    ├── icon-192.png
    └── icon-512.png
```

### 13.2 Backend (Supabase)
- **PostgreSQL** — таблицы, RLS, функции, триггеры
- **Supabase Auth** — JWT, sessions
- **Supabase Storage** — файлы, скриншоты
- **Supabase Edge Functions** (Deno):
  - `send-email` — отправка email-уведомлений
  - `hash-password` — хеширование паролей (bcrypt)
  - `create-client-account` — создание аккаунта клиента с auto-паролем
  - `check-author-eligibility` — проверка уникальности автор/площадка
- **Supabase Realtime** — подписки на изменения

### 13.3 Триггеры PostgreSQL
```sql
-- Автообновление updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$ BEGIN NEW.updated_at = now(); RETURN NEW; END; $$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at BEFORE UPDATE ON deals FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON reviews FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON clients FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Автосоздание уведомления при смене статуса сделки
CREATE OR REPLACE FUNCTION notify_deal_status_change()
RETURNS TRIGGER AS $$
BEGIN
  IF OLD.status IS DISTINCT FROM NEW.status THEN
    -- Логика создания уведомлений в зависимости от нового статуса
    INSERT INTO action_logs (user_id, action_type, entity_type, entity_id, details)
    VALUES (auth.uid(), 'deal.status_change', 'deal', NEW.id, 
      jsonb_build_object('old_status', OLD.status, 'new_status', NEW.status));
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER deal_status_change AFTER UPDATE ON deals FOR EACH ROW EXECUTE FUNCTION notify_deal_status_change();

-- Проверка завершения сделки (все отзывы completed)
CREATE OR REPLACE FUNCTION check_deal_completion()
RETURNS TRIGGER AS $$
DECLARE
  total_reviews INT;
  completed_reviews INT;
BEGIN
  IF NEW.status = 'completed' THEN
    SELECT COUNT(*), COUNT(*) FILTER (WHERE status = 'completed')
    INTO total_reviews, completed_reviews
    FROM reviews WHERE deal_id = NEW.deal_id;
    
    IF total_reviews > 0 AND total_reviews = completed_reviews THEN
      UPDATE deals SET status = 'completed' WHERE id = NEW.deal_id;
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_deal_on_review_complete AFTER UPDATE ON reviews FOR EACH ROW EXECUTE FUNCTION check_deal_completion();
```

---

## 14. ЛОГИРОВАНИЕ ДЕЙСТВИЙ

### 14.1 Функция логирования (frontend)
```javascript
// js/logger.js
async function logAction(actionType, entityType, entityId, details = {}) {
  try {
    await supabase.from('action_logs').insert({
      user_id: currentUser.id,
      user_role: currentUser.role,
      action_type: actionType,
      entity_type: entityType,
      entity_id: entityId,
      details: details,
      ip_address: null,  // заполняется через Edge Function если нужно
      user_agent: navigator.userAgent
    });
  } catch (e) {
    console.error('Log error:', e);
  }
}
```

### 14.2 Точки вызова
Каждое действие пользователя должно вызывать `logAction()`:
- Вход / Выход
- Создание / Редактирование / Удаление сделки
- Создание / Редактирование отзыва
- Отправка на согласование / Согласование / Отклонение
- Передача в работу
- Взятие задания / Загрузка скриншота / Закрытие задания
- Загрузка / Удаление файлов
- Изменение профиля / Смена пароля
- Активация / Блокировка пользователя

---

## 15. МИГРАЦИЯ С ТЕКУЩЕГО КОДА

### 15.1 Что сохранить
- Общую концепцию визуала (glass-btn, radio-inputs, timeline, card-layout)
- Структуру данных (адаптировав к новой схеме)
- Логику автокомплита ИНН/клиентов
- Анимацию создания отзыва (стирание из textarea)

### 15.2 Что переделать
- **Авторизация**: убрать plain-text пароли → bcrypt через Edge Function
- **Роутинг**: вместо загрузки HTML-фрагментов через `fetch` → JS-роутер с lazy-загрузкой
- **Структура**: разделить монолитный HTML на модульную структуру
- **RLS**: настроить Row Level Security вместо клиентской фильтрации
- **Уведомления**: реализовать полноценную систему (email + in-app + realtime)
- **Адаптивность**: добавить полную mobile-поддержку
- **PWA**: service worker, manifest, offline shell
- **Логирование**: добавить `action_logs` на каждое действие

### 15.3 Маппинг таблиц (старые → новые)
```
reviews_users        → users
reviews_counterparty → counterparties
reviews_human_contact → clients
reviews_list_deals   → deals
reviews_deal_tasks   → reviews
(новая)              → action_logs
(новая)              → notifications
(новая)              → brief_templates
(новая)              → platforms
```

---

## 16. ПЛАН РАЗРАБОТКИ (ФАЗЫ)

### Фаза 1: Ядро (MVP)
1. Структура БД + миграции + RLS
2. App Shell (header, sidebar, routing, responsive)
3. Авторизация (вход, регистрация, роли, защита маршрутов)
4. Страница «Сделки» (CRUD, timeline, документы)
5. Бриф клиента
6. Создание и согласование отзывов
7. Логирование действий
8. PWA (manifest, service worker)

### Фаза 2: Полный функционал
1. ЛК клиента (полный)
2. Доска заданий автора (Kanban) + 3-шаговый процесс
3. Email-уведомления (Edge Functions)
4. In-app уведомления + Realtime
5. Управление пользователями (Super Admin)
6. Система проверки уникальности автор/площадка

### Фаза 3: Аналитика и оптимизация
1. Дашборд со статистикой и графиками
2. Страница аналитики / логов
3. Push-уведомления
4. Оптимизация производительности
5. Полное тестирование

---

## 17. КРИТИЧЕСКИЕ ТРЕБОВАНИЯ (ЧЕКЛИСТ)

- [ ] **Пароли НЕ хранятся в plain-text** — только bcrypt hash
- [ ] **RLS на всех таблицах** — клиент не видит чужие данные
- [ ] **Один автор = один отзыв на площадку** — серверная проверка
- [ ] **48-часовые паузы** — серверная валидация (не только клиентская)
- [ ] **Обязательный скриншот при закрытии** — валидация наличия файла
- [ ] **Логирование КАЖДОГО действия** — в `action_logs`
- [ ] **Email-уведомления** — при всех ключевых событиях
- [ ] **Responsive** — работает на телефоне и десктопе
- [ ] **PWA** — устанавливается как приложение
- [ ] **Realtime** — обновления без перезагрузки

---

## 18. SUPABASE КОНФИГУРАЦИЯ

### 18.1 Текущий Supabase проект
```
URL: https://ncdwlmgwljjwpuawlssr.supabase.co
ANON KEY: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 18.2 Необходимо настроить:
1. **Auth**: включить Email Provider, настроить email templates на русском
2. **Database**: выполнить SQL-миграции из раздела 3
3. **Storage**: создать бакеты `deals-documents`, `task-attachments`, `avatars`
4. **Edge Functions**: развернуть `send-email`, `hash-password`, `create-client-account`
5. **Realtime**: включить для таблиц `deals`, `reviews`, `notifications`
6. **RLS**: настроить политики из раздела 3.10

---

*Конец технического задания.*
