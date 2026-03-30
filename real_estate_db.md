# Проєкт бази даних агентства нерухомості (Етап 1)

Цей файл містить повний набір SQL-скриптів для розгортання бази даних у схемі `kr`, наповнення її тестовими даними та виконання аналітичних запитів.

## 1. Підготовка схеми та створення таблиць

Використовується схема `kr`. Всі таблиці створені згідно з ER-діаграмою з використанням обмежень цілісності.

```sql
SET search_path TO kr;

-- Очищення схеми перед створенням
DROP TABLE IF EXISTS sales, viewings, properties, clients, agents, property_type CASCADE;
DROP TYPE IF EXISTS client_type_enum CASCADE;
DROP TYPE IF EXISTS property_status_enum CASCADE;

-- Створення перелічуваних типів
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
    hire_date DATE DEFAULT CURRENT_DATE NOT NULL -- NOT NULL згідно з вимогами бази
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
    feedback TEXT -- Тип TEXT для довгих відгуків
);

-- Таблиця продажів
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    property_id INT UNIQUE REFERENCES properties(property_id),
    buyer_id INT REFERENCES clients(client_id),
    agent_id INT REFERENCES agents(agent_id),
    sale_price NUMERIC(15, 2) NOT NULL,
    sale_date DATE NOT NULL DEFAULT CURRENT_DATE,
    commission_pct NUMERIC(4, 2)
);
