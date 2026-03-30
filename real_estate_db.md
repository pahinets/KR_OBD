# Звіт з проєктування та розробки бази даних "Агентство нерухомості"

Цей документ містить повний набір SQL-інструкцій для розгортання, наповнення та аналізу бази даних. Всі операції виконуються в окремій схемі `kr`.

## 1. Проєктування схеми (DDL)

Схема побудована на основі ER-діаграми. Для забезпечення цілісності використані первинні та зовнішні ключі (`PRIMARY KEY`, `REFERENCES`), а також перелічувані типи (`ENUM`) для категоріальних даних.

```sql
-- Налаштування робочого середовища
SET search_path TO kr;

-- Очищення схеми
DROP TABLE IF EXISTS sales, viewings, properties, clients, agents, property_type CASCADE;
DROP TYPE IF EXISTS client_type_enum CASCADE;
DROP TYPE IF EXISTS property_status_enum CASCADE;

-- Створення користувацьких типів даних
CREATE TYPE client_type_enum AS ENUM ('покупець', 'продавець');
CREATE TYPE property_status_enum AS ENUM ('доступна', 'зарезервована', 'продана');

-- Таблиця типів нерухомості
CREATE TABLE property_type (
    type_id SERIAL PRIMARY KEY,
    type_name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT
);

-- Таблиця агентів
CREATE TABLE agents (
    agent_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(100) UNIQUE,
    specialization VARCHAR(100),
    hire_date DATE DEFAULT CURRENT_DATE NOT NULL
);

-- Таблиця клієнтів
CREATE TABLE clients (
    client_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(100) UNIQUE,
    client_type client_type_enum NOT NULL,
    registration_date DATE DEFAULT CURRENT_DATE
);

-- Таблиця об'єктів нерухомості
CREATE TABLE properties (
    property_id SERIAL PRIMARY KEY,
    type_id INT REFERENCES property_type(type_id),
    agent_id INT REFERENCES agents(agent_id),
    owner_id INT REFERENCES clients(client_id),
    address VARCHAR(255) NOT NULL,
    city VARCHAR(100),
    price NUMERIC(15, 2) NOT NULL,
    rooms SMALLINT NOT NULL CHECK (rooms > 0),
    area_sqm NUMERIC(10, 2),
    status property_status_enum DEFAULT 'доступна',
    listed_date DATE NOT NULL DEFAULT CURRENT_DATE
);

-- Таблиця оглядів
CREATE TABLE viewings (
    viewing_id SERIAL PRIMARY KEY,
    property_id INT REFERENCES properties(property_id) ON DELETE CASCADE,
    client_id INT REFERENCES clients(client_id),
    agent_id INT REFERENCES agents(agent_id),
    viewing_date TIMESTAMP NOT NULL,
    notes TEXT,
    feedback TEXT
);

-- Таблиця транзакцій продажу
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    property_id INT UNIQUE REFERENCES properties(property_id),
    buyer_id INT REFERENCES clients(client_id),
    agent_id INT REFERENCES agents(agent_id),
    sale_price NUMERIC(15, 2) NOT NULL,
    sale_date DATE NOT NULL DEFAULT CURRENT_DATE,
    commission_pct NUMERIC(4, 2)
);
```
## 2. Операції маніпулювання даними (DML / OLTP)

Нижче наведено приклади базових CRUD-операцій: додавання, оновлення, видалення та пошуку даних.

## 2.1. Додавання даних (INSERT)

```sql
-- Додавання типів та персоналу
INSERT INTO property_type (type_name, description) VALUES ('Квартира', 'Житло у багатоповерхівці');
INSERT INTO agents (first_name, last_name, email, specialization) 
VALUES ('Петро', 'Шевченко', 'petro.agent@example.com', 'Центральний район');

-- Додавання клієнтів
INSERT INTO clients (first_name, last_name, client_type) 
VALUES ('Олексій', 'Коваленко', 'продавець'), ('Марія', 'Ткачук', 'покупець');

-- Додавання нерухомості
INSERT INTO properties (type_id, agent_id, owner_id, address, city, price, rooms)
VALUES (1, 1, 1, 'вул. Хрещатик, 10', 'Київ', 150000.00, 3),
       (1, 1, 1, 'вул. Дерибасівська, 1', 'Одеса', 120000.00, 2);

-- Реєстрація оглядів та продажів
INSERT INTO viewings (property_id, client_id, agent_id, viewing_date, feedback)
VALUES (1, 2, 1, CURRENT_DATE, 'Чудовий стан');
INSERT INTO sales (property_id, buyer_id, agent_id, sale_price, commission_pct)
VALUES (1, 2, 1, 145000.00, 5.00);
```
## 2.2. Оновлення та видалення (UPDATE / DELETE)
```sql
-- Зміна ціни на доступні об'єкти
UPDATE properties SET price = price * 0.98 WHERE status = 'доступна';

-- Оновлення відгуку
UPDATE viewings SET feedback = 'Замала кухня' WHERE viewing_id = 1;

-- Видалення застарілих оглядів
DELETE FROM viewings WHERE viewing_date < '2024-01-01';

-- Видалення клієнтів без контактів
DELETE FROM clients WHERE phone IS NULL AND email IS NULL;
```
## 2.3. Прості вибірки (SELECT)
```sql
-- Нерухомість конкретного агента
SELECT address, price, status FROM properties WHERE agent_id = 1;

-- Огляди для конкретної нерухомості
SELECT v.viewing_date, c.last_name as client, v.feedback 
FROM viewings v 
JOIN clients c ON v.client_id = c.client_id 
WHERE v.property_id = 1;
```
## 3. Аналітичні запити (OLAP)

Використання агрегатних функцій, групування та спільних табличних виразів (CTE) для аналізу бізнес-показників.
```sql
-- 1. Середня ціна продажу за типом нерухомості
SELECT pt.type_name, ROUND(AVG(s.sale_price), 2) as avg_price
FROM sales s
JOIN properties p ON s.property_id = p.property_id
JOIN property_type pt ON p.type_id = pt.type_id
GROUP BY pt.type_name;

-- 2. Рейтинг агентів за кількістю продажів за останній рік
SELECT a.first_name, a.last_name, COUNT(s.sale_id) as sales_count
FROM sales s
JOIN agents a ON s.agent_id = a.agent_id
WHERE s.sale_date >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY a.agent_id, a.first_name, a.last_name
ORDER BY sales_count DESC;

-- 3. Середній час знаходження об'єкта на ринку (у днях)
SELECT ROUND(AVG(s.sale_date - p.listed_date), 0) as avg_days_to_sale
FROM sales s
JOIN properties p ON s.property_id = p.property_id;

-- 4. Нерухомість з оглядами, яка ще не була продана (CTE)
WITH property_view_counts AS (
    SELECT property_id, COUNT(*) as total_views
    FROM viewings
    GROUP BY property_id
)
SELECT p.address, p.price, pvc.total_views
FROM properties p
JOIN property_view_counts pvc ON p.property_id = pvc.property_id
LEFT JOIN sales s ON p.property_id = s.property_id
WHERE s.sale_id IS NULL AND p.status = 'доступна';
```
