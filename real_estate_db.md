# Лабораторна робота — База даних агентства нерухомості

---

## 1. Схема бази даних

База даних складається з 6 таблиць, що охоплюють повний цикл роботи агентства нерухомості.

---

### 1.1 PropertyType — Типи нерухомості

| Поле | Тип даних | Обмеження | Опис |
|------|-----------|-----------|------|
| type_id | SERIAL | PRIMARY KEY | Унікальний ідентифікатор типу |
| type_name | VARCHAR(50) | NOT NULL, UNIQUE | Назва типу (будинок, квартира, комерційна) |
| description | TEXT | — | Детальний опис типу нерухомості |

---

### 1.2 Agent — Агенти

| Поле | Тип даних | Обмеження | Опис |
|------|-----------|-----------|------|
| agent_id | SERIAL | PRIMARY KEY | Унікальний ідентифікатор агента |
| first_name | VARCHAR(100) | NOT NULL | Ім'я агента |
| last_name | VARCHAR(100) | NOT NULL | Прізвище агента |
| phone | VARCHAR(20) | UNIQUE | Контактний телефон |
| email | VARCHAR(150) | UNIQUE, NOT NULL | Електронна пошта |
| specialization | VARCHAR(100) | — | Спеціалізація (елітна, комерційна тощо) |
| hire_date | DATE | NOT NULL | Дата прийому на роботу |

---

### 1.3 Client — Клієнти

| Поле | Тип даних | Обмеження | Опис |
|------|-----------|-----------|------|
| client_id | SERIAL | PRIMARY KEY | Унікальний ідентифікатор клієнта |
| first_name | VARCHAR(100) | NOT NULL | Ім'я клієнта |
| last_name | VARCHAR(100) | NOT NULL | Прізвище клієнта |
| phone | VARCHAR(20) | UNIQUE | Контактний телефон |
| email | VARCHAR(150) | UNIQUE | Електронна пошта |
| client_type | VARCHAR(10) | CHECK IN ('buyer','seller') | Тип клієнта: покупець або продавець |
| registration_date | DATE | NOT NULL DEFAULT CURRENT_DATE | Дата реєстрації у системі |

---

### 1.4 Property — Нерухомість

| Поле | Тип даних | Обмеження | Опис |
|------|-----------|-----------|------|
| property_id | SERIAL | PRIMARY KEY | Унікальний ідентифікатор об'єкта |
| type_id | INT | FK → PropertyType | Тип нерухомості |
| agent_id | INT | FK → Agent | Відповідальний агент |
| owner_id | INT | FK → Client | Клієнт-власник (продавець) |
| address | VARCHAR(255) | NOT NULL | Повна адреса об'єкта |
| city | VARCHAR(100) | NOT NULL | Місто розташування |
| price | NUMERIC(15,2) | NOT NULL, CHECK > 0 | Ціна виставлення на продаж |
| rooms | SMALLINT | CHECK >= 0 | Кількість кімнат |
| area_sqm | NUMERIC(8,2) | — | Загальна площа (кв.м) |
| status | VARCHAR(20) | CHECK IN ('active','sold','withdrawn') | Поточний статус об'єкта |
| listed_date | DATE | NOT NULL DEFAULT CURRENT_DATE | Дата виставлення на продаж |

---

### 1.5 Viewing — Огляди нерухомості

| Поле | Тип даних | Обмеження | Опис |
|------|-----------|-----------|------|
| viewing_id | SERIAL | PRIMARY KEY | Унікальний ідентифікатор огляду |
| property_id | INT | FK → Property, NOT NULL | Об'єкт нерухомості |
| client_id | INT | FK → Client, NOT NULL | Клієнт-покупець |
| agent_id | INT | FK → Agent, NOT NULL | Агент, що проводив огляд |
| viewing_date | TIMESTAMP | NOT NULL | Дата та час огляду |
| notes | TEXT | — | Нотатки за результатами огляду |
| feedback | VARCHAR(20) | CHECK IN ('positive','neutral','negative') | Відгук клієнта після огляду |

---

### 1.6 Sale — Продажі

| Поле | Тип даних | Обмеження | Опис |
|------|-----------|-----------|------|
| sale_id | SERIAL | PRIMARY KEY | Унікальний ідентифікатор угоди |
| property_id | INT | FK → Property, UNIQUE | Проданий об'єкт нерухомості |
| buyer_id | INT | FK → Client, NOT NULL | Клієнт-покупець |
| agent_id | INT | FK → Agent, NOT NULL | Агент, що завершив угоду |
| sale_price | NUMERIC(15,2) | NOT NULL, CHECK > 0 | Фактична ціна продажу |
| sale_date | DATE | NOT NULL | Дата завершення угоди |
| commission_pct | NUMERIC(4,2) | DEFAULT 3.00 | Комісія агента (%) |

---

### 1.7 Зв'язки між таблицями (ERD)

```
PropertyType ──┐
               │ 1:N
Agent ─────────┼──────► Property ──────► Viewing
               │  1:N        │  1:N
Client ────────┘             │
  (owner)                    │ 1:1
                             ▼
                            Sale
                        (buyer, agent)
```

Зв'язки:
- `PropertyType` 1 ──── N `Property`
- `Agent` 1 ──── N `Property`
- `Client` 1 ──── N `Property` (власник)
- `Property` 1 ──── N `Viewing`
- `Client` 1 ──── N `Viewing` (покупець)
- `Agent` 1 ──── N `Viewing`
- `Property` 1 ──── 1 `Sale`
- `Client` 1 ──── N `Sale` (покупець)
- `Agent` 1 ──── N `Sale`

---

### 1.8 DDL — Створення таблиць

```sql
-- Типи нерухомості
CREATE TABLE PropertyType (
    type_id     SERIAL PRIMARY KEY,
    type_name   VARCHAR(50)  NOT NULL UNIQUE,
    description TEXT
);

-- Агенти
CREATE TABLE Agent (
    agent_id       SERIAL PRIMARY KEY,
    first_name     VARCHAR(100) NOT NULL,
    last_name      VARCHAR(100) NOT NULL,
    phone          VARCHAR(20)  UNIQUE,
    email          VARCHAR(150) NOT NULL UNIQUE,
    specialization VARCHAR(100),
    hire_date      DATE         NOT NULL
);

-- Клієнти
CREATE TABLE Client (
    client_id         SERIAL PRIMARY KEY,
    first_name        VARCHAR(100) NOT NULL,
    last_name         VARCHAR(100) NOT NULL,
    phone             VARCHAR(20)  UNIQUE,
    email             VARCHAR(150) UNIQUE,
    client_type       VARCHAR(10)  NOT NULL
        CHECK (client_type IN ('buyer', 'seller')),
    registration_date DATE NOT NULL DEFAULT CURRENT_DATE
);

-- Нерухомість
CREATE TABLE Property (
    property_id SERIAL PRIMARY KEY,
    type_id     INT           NOT NULL REFERENCES PropertyType(type_id),
    agent_id    INT           NOT NULL REFERENCES Agent(agent_id),
    owner_id    INT           NOT NULL REFERENCES Client(client_id),
    address     VARCHAR(255)  NOT NULL,
    city        VARCHAR(100)  NOT NULL,
    price       NUMERIC(15,2) NOT NULL CHECK (price > 0),
    rooms       SMALLINT      CHECK (rooms >= 0),
    area_sqm    NUMERIC(8,2),
    status      VARCHAR(20)   NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'sold', 'withdrawn')),
    listed_date DATE          NOT NULL DEFAULT CURRENT_DATE
);

-- Огляди
CREATE TABLE Viewing (
    viewing_id   SERIAL PRIMARY KEY,
    property_id  INT       NOT NULL REFERENCES Property(property_id),
    client_id    INT       NOT NULL REFERENCES Client(client_id),
    agent_id     INT       NOT NULL REFERENCES Agent(agent_id),
    viewing_date TIMESTAMP NOT NULL,
    notes        TEXT,
    feedback     VARCHAR(20)
        CHECK (feedback IN ('positive', 'neutral', 'negative'))
);

-- Продажі
CREATE TABLE Sale (
    sale_id        SERIAL PRIMARY KEY,
    property_id    INT           NOT NULL UNIQUE REFERENCES Property(property_id),
    buyer_id       INT           NOT NULL REFERENCES Client(client_id),
    agent_id       INT           NOT NULL REFERENCES Agent(agent_id),
    sale_price     NUMERIC(15,2) NOT NULL CHECK (sale_price > 0),
    sale_date      DATE          NOT NULL,
    commission_pct NUMERIC(4,2)  NOT NULL DEFAULT 3.00
);
```

---

## 2. OLTP запити

OLTP (Online Transaction Processing) — запити, орієнтовані на маніпуляцію окремими записами в реальному часі.

---

### 2.1 INSERT — Додавання даних

#### INSERT 1 — Реєстрація нового об'єкта нерухомості

Сценарій: агент (agent_id = 3) отримав нове доручення від клієнта-продавця (owner_id = 7) на продаж квартири у Львові за 85 000 USD.

```sql
INSERT INTO Property (
    type_id, agent_id, owner_id,
    address, city, price, rooms, area_sqm, status
)
VALUES (
    2,                               -- квартира
    3,                               -- відповідальний агент
    7,                               -- власник (продавець)
    'вул. Городоцька, 45, кв. 12',  -- адреса
    'Львів',
    85000.00,
    3,
    72.50,
    'active'
);
```

#### INSERT 2 — Запис нового огляду нерухомості

Сценарій: покупець (client_id = 12) запланував огляд квартири (property_id = 5) з агентом (agent_id = 3) на 15 квітня 2025 р. о 14:00.

```sql
INSERT INTO Viewing (
    property_id, client_id, agent_id,
    viewing_date, notes, feedback
)
VALUES (
    5,
    12,
    3,
    '2025-04-15 14:00:00',
    'Клієнт зацікавлений, просить уточнити стан сантехніки',
    'positive'
);
```

---

### 2.2 UPDATE — Оновлення даних

#### UPDATE 1 — Зміна ціни нерухомості

Сценарій: власник погодився знизити ціну об'єкта (property_id = 5) з 85 000 до 80 000 USD після переговорів.

```sql
UPDATE Property
SET
    price  = 80000.00,
    status = 'active'
WHERE property_id = 5;
```

#### UPDATE 2 — Оновлення спеціалізації агента

Сценарій: агент (agent_id = 3) підвищив кваліфікацію і отримав нову спеціалізацію.

```sql
UPDATE Agent
SET specialization = 'Елітна та комерційна нерухомість'
WHERE agent_id = 3;

-- Перепризначення активних об'єктів без агента
UPDATE Property
SET agent_id = 3
WHERE status = 'active'
  AND agent_id IN (
      SELECT agent_id FROM Agent
      WHERE hire_date < CURRENT_DATE - INTERVAL '5 years'
        AND specialization IS NULL
  );
```

---

### 2.3 DELETE — Видалення даних

#### DELETE 1 — Скасування запланованого огляду

Сценарій: покупець скасував огляд (viewing_id = 18), що ще не відбувся.

```sql
DELETE FROM Viewing
WHERE viewing_id = 18
  AND viewing_date > CURRENT_TIMESTAMP;
```

#### DELETE 2 — Видалення знятих з продажу об'єктів старше 2 років

Сценарій: архівна очистка — видалення об'єктів зі статусом `withdrawn`, знятих більш ніж 2 роки тому і без прив'язаних оглядів.

```sql
DELETE FROM Property
WHERE status = 'withdrawn'
  AND listed_date < CURRENT_DATE - INTERVAL '2 years'
  AND property_id NOT IN (
      SELECT DISTINCT property_id FROM Viewing
  );
```

---

### 2.4 SELECT — Вибірка даних

#### SELECT 1 — Вся нерухомість, яку веде конкретний агент

```sql
SELECT
    p.property_id,
    pt.type_name          AS property_type,
    p.address,
    p.city,
    p.price,
    p.rooms,
    p.area_sqm,
    p.status,
    p.listed_date,
    c.first_name || ' ' || c.last_name AS owner_name
FROM Property p
JOIN PropertyType pt ON pt.type_id  = p.type_id
JOIN Client       c  ON c.client_id = p.owner_id
WHERE p.agent_id = 3        -- конкретний агент
  AND p.status   = 'active'
ORDER BY p.listed_date DESC;
```

#### SELECT 2 — Список усіх оглядів для конкретної нерухомості

```sql
SELECT
    v.viewing_id,
    v.viewing_date,
    c.first_name || ' ' || c.last_name  AS buyer_name,
    c.phone                              AS buyer_phone,
    a.first_name || ' ' || a.last_name  AS agent_name,
    v.feedback,
    v.notes
FROM Viewing v
JOIN Client c ON c.client_id = v.client_id
JOIN Agent  a ON a.agent_id  = v.agent_id
WHERE v.property_id = 5     -- конкретний об'єкт
ORDER BY v.viewing_date DESC;
```

---

## 3. OLAP запити (аналітичні)

OLAP (Online Analytical Processing) — запити для аналізу агрегованих даних, виявлення трендів та підтримки управлінських рішень.

---

### 3.1 Середня ціна продажу за типом нерухомості

Аналізує реальні ціни угод у розрізі типів об'єктів. Дозволяє оцінити ринкові сегменти.

```sql
SELECT
    pt.type_name                       AS property_type,
    COUNT(s.sale_id)                   AS total_sales,
    ROUND(AVG(s.sale_price), 2)        AS avg_sale_price,
    ROUND(MIN(s.sale_price), 2)        AS min_sale_price,
    ROUND(MAX(s.sale_price), 2)        AS max_sale_price,
    ROUND(AVG(s.sale_price / p.price
              * 100), 1)               AS avg_price_pct_of_listed
FROM Sale         s
JOIN Property     p  ON p.property_id = s.property_id
JOIN PropertyType pt ON pt.type_id    = p.type_id
GROUP BY pt.type_name
ORDER BY avg_sale_price DESC;
```

---

### 3.2 Агенти з найбільшою кількістю успішних продажів за останній рік

Рейтинг агентів за кількістю та загальним обсягом угод за останні 12 місяців.

```sql
SELECT
    a.agent_id,
    a.first_name || ' ' || a.last_name  AS agent_name,
    a.specialization,
    COUNT(s.sale_id)                     AS sales_count,
    ROUND(SUM(s.sale_price), 2)          AS total_revenue,
    ROUND(AVG(s.sale_price), 2)          AS avg_deal_size,
    ROUND(SUM(s.sale_price *
              s.commission_pct / 100), 2) AS total_commission
FROM Agent a
JOIN Sale  s ON s.agent_id = a.agent_id
WHERE s.sale_date >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY a.agent_id, a.first_name, a.last_name, a.specialization
HAVING COUNT(s.sale_id) > 0
ORDER BY sales_count DESC, total_revenue DESC
LIMIT 10;
```

---

### 3.3 Середній час перебування нерухомості на ринку до продажу

Обчислює кількість днів від виставлення до продажу в розрізі типів та міст. Показник ліквідності ринку.

```sql
SELECT
    pt.type_name                          AS property_type,
    p.city,
    COUNT(s.sale_id)                      AS sales_count,
    ROUND(AVG(
        s.sale_date - p.listed_date
    ), 1)                                 AS avg_days_on_market,
    MIN(s.sale_date - p.listed_date)      AS min_days,
    MAX(s.sale_date - p.listed_date)      AS max_days
FROM Sale         s
JOIN Property     p  ON p.property_id = s.property_id
JOIN PropertyType pt ON pt.type_id    = p.type_id
GROUP BY pt.type_name, p.city
HAVING COUNT(s.sale_id) >= 2   -- статистично значущі вибірки
ORDER BY avg_days_on_market ASC;
```

---

### 3.4 Нерухомість з багатьма оглядами, але без продажу (CTE)

Виявляє «проблемні» об'єкти: ті, що мають значний інтерес покупців (багато оглядів), але так і не були продані. Допомагає переглянути ціну або стратегію просування.

```sql
WITH viewing_stats AS (
    -- Підраховуємо кількість оглядів для кожного об'єкта
    SELECT
        property_id,
        COUNT(*)                            AS viewing_count,
        COUNT(*) FILTER (WHERE feedback = 'positive')
                                            AS positive_count,
        MAX(viewing_date)                   AS last_viewing
    FROM Viewing
    GROUP BY property_id
),
avg_viewings AS (
    -- Середня кількість оглядів по всій базі
    SELECT AVG(viewing_count) AS mean_views
    FROM viewing_stats
)
SELECT
    p.property_id,
    pt.type_name                            AS property_type,
    p.address,
    p.city,
    p.price,
    p.status,
    p.listed_date,
    CURRENT_DATE - p.listed_date            AS days_listed,
    vs.viewing_count,
    vs.positive_count,
    vs.last_viewing,
    a.first_name || ' ' || a.last_name      AS agent_name
FROM Property     p
JOIN PropertyType pt ON pt.type_id      = p.type_id
JOIN Agent        a  ON a.agent_id      = p.agent_id
JOIN viewing_stats vs ON vs.property_id = p.property_id
CROSS JOIN avg_viewings av
-- Об'єкт без продажу:
WHERE p.property_id NOT IN (SELECT property_id FROM Sale)
  AND p.status <> 'withdrawn'
-- Кількість оглядів вища за середню:
  AND vs.viewing_count > av.mean_views
ORDER BY vs.viewing_count DESC, days_listed DESC;
```
