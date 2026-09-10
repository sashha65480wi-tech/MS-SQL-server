# MS-SQL-server
# 📘 Документация базы данных «Офисные работники»

**СУБД:** SQLite  
**Название БД:** `office.db`  
**Кодировка:** UTF-8  
**Назначение:** учёт отделов, должностей, сотрудников, проектов и их участия в проектах.

---

## 📑 Содержание

1. Общее описание
    
2. ER-схема связей
    
3. Таблица `departments`
    
4. Таблица `positions`
    
5. Таблица `employees`
    
6. Таблица `projects`
    
7. Таблица `project_members`
    
8. Связи и ограничения
    
9. Полный SQL создания
    
10. Примеры запросов
    
11. Рекомендации по эксплуатации
    

---

## 1. Общее описание

База данных описывает структуру вымышленной IT-компании:

- **Отделы** (`departments`) — подразделения, в которых работают сотрудники.
    
- **Должности** (`positions`) — список возможных ролей с вилкой зарплат.
    
- **Сотрудники** (`employees`) — люди, работающие в компании.
    
- **Проекты** (`projects`) — проекты компании с бюджетом и сроками.
    
- **Участие в проектах** (`project_members`) — связь «многие ко многим» между сотрудниками и проектами.
    

### Ключевые особенности

- Сотрудник может быть **руководителем** других сотрудников (иерархия через `manager_id`).
    
- Сотрудник может участвовать в **нескольких проектах**, а проект — включать нескольких сотрудников.
    
- У каждого сотрудника есть **ровно один отдел** и **одна должность**.
    
- Сотрудников **не удаляют физически** — вместо этого используется флаг `is_active`.
    

---

## 2. ER-схема связей

text

 ┌───────────────┐        ┌───────────────┐
 │ departments   │        │  positions    │
 ├───────────────┤        ├───────────────┤
 │ id (PK)       │        │ id (PK)       │
 │ title         │        │ title         │
 │ ceiling       │        │ min_salary    │
 │ phone         │        │ max_salary    │
 └───────┬───────┘        └───────┬───────┘
         │ 1                    1 │
         │                        │
         │ N                     N│
         ▼                        ▼
 ┌──────────────────────────────────────┐
 │             employees                │
 ├──────────────────────────────────────┤
 │ id (PK)                              │
 │ first_name, last_name, email, phone  │
 │ birth_date, hire_date                │
 │ department_id (FK → departments.id)  │
 │ position_id   (FK → positions.id)    │
 │ manager_id    (FK → employees.id)  ◄─┐
 │ salary, is_active                    │
 └───────────────┬──────────────────────┘
                 │ 1                  │
                 │                    │ self-FK
                 │ N                  │
                 ▼                    │
 ┌──────────────────────────────┐     │
 │      project_members         │     │
 ├──────────────────────────────┤     │
 │ employee_id (PK, FK) ────────┼─────┘
 │ project_id  (PK, FK)         │
 │ role                         │
 └───────────────┬──────────────┘
                 │ N
                 │
                 │ 1
                 ▼
 ┌──────────────────────────────┐
 │         projects             │
 ├──────────────────────────────┤
 │ id (PK)                      │
 │ title                        │
 │ start_date, end_date         │
 │ budget                       │
 └──────────────────────────────┘

---

## 3. Таблица `departments`

**Назначение:** список подразделений (отделов) компании.

|Поле|Тип|Ограничения|Описание|
|---|---|---|---|
|`id`|INTEGER|PRIMARY KEY|Уникальный идентификатор отдела|
|`title`|VARCHAR(100)|NOT NULL, UNIQUE|Название отдела|
|`ceiling`|INTEGER|—|Этаж (по антониму — «потолок»), где находится отдел|
|`phone`|VARCHAR(20)|—|Внутренний номер телефона|

**Ограничения:**

- `title` уникально — нельзя создать два отдела с одинаковым названием.
    

**Текущее содержимое таблицы:**

![[Pasted image 20260910141454.png]]


|id|title|ceiling|phone|
|---|---|---|---|
|1|Разработка|3|101|
|2|Маркетинг|2|205|
|3|Бухгалтерия|1|310|
|4|Отдел кадров|1|312|
|5|Продажи|4|401|

**Всего записей:** 5.

---

## 4. Таблица `positions`

**Назначение:** справочник должностей с указанием вилки зарплат.

|Поле|Тип|Ограничения|Описание|
|---|---|---|---|
|`id`|INTEGER|PRIMARY KEY|Уникальный идентификатор должности|
|`title`|VARCHAR(100)|NOT NULL, UNIQUE|Название должности|
|`min_salary`|INTEGER|—|Минимальная зарплата|
|`max_salary`|INTEGER|—|Максимальная зарплата|

**Особенности:**

- Не участвует в связях напрямую, кроме FK из `employees`.
    
- Вилка зарплат используется для проверки «не выходит ли сотрудник за потолок».
    

**Текущее содержимое таблицы:**
![[Pasted image 20260910141622.png]]


|id|title|min_salary|max_salary|
|---|---|---|---|
|1|Стажёр|30000|50000|
|2|Разработчик|80000|200000|
|3|Старший разработчик|150000|300000|
|4|Менеджер|70000|150000|
|5|Бухгалтер|60000|120000|
|6|HR-специалист|50000|100000|
|7|Директор|200000|500000|

**Всего записей:** 7.

---

## 5. Таблица `employees`

**Назначение:** основная таблица — сотрудники компании.

|Поле|Тип|Ограничения|Описание|
|---|---|---|---|
|`id`|INTEGER|PRIMARY KEY|Уникальный идентификатор сотрудника|
|`first_name`|VARCHAR(50)|NOT NULL|Имя|
|`last_name`|VARCHAR(50)|NOT NULL|Фамилия|
|`email`|VARCHAR(100)|UNIQUE|Электронная почта|
|`phone`|VARCHAR(30)|—|Телефон|
|`birth_date`|VARCHAR(10)|—|Дата рождения (`YYYY-MM-DD`)|
|`hire_date`|VARCHAR(10)|NOT NULL|Дата приёма на работу|
|`department_id`|INTEGER|FK → `departments.id`|Отдел|
|`position_id`|INTEGER|FK → `positions.id`|Должность|
|`manager_id`|INTEGER|FK → `employees.id`|Руководитель|
|`salary`|INTEGER|—|Текущая зарплата|
|`is_active`|INTEGER|DEFAULT 1|1 — работает, 0 — уволен|

### Особенности

- **`manager_id`** — ссылка на саму таблицу (self-reference). У топов значение `NULL`.
    
- **`is_active`** вместо физического удаления: исторические данные и участие в проектах сохраняются.
    
- **Даты хранятся как текст** в формате `YYYY-MM-DD` — это стандарт SQLite, удобно сравнивать лексикографически.
    

**Текущее содержимое таблицы:**

![[Pasted image 20260910141636.png]]

|id|first_name|last_name|email|department_id|position_id|manager_id|salary|is_active|
|---|---|---|---|---|---|---|---|---|
|1|Иван|Петров|ivan.petrov@company.ru|1|3|NULL|250000|1|
|2|Мария|Сидорова|maria.sidorova@company.ru|1|2|1|160000|1|
|3|Алексей|Кузнецов|alex.kuznetsov@company.ru|1|2|1|140000|1|
|4|Ольга|Смирнова|olga.smirnova@company.ru|2|4|NULL|180000|1|
|5|Дмитрий|Волков|dmitry.volkov@company.ru|2|2|4|120000|1|
|6|Анна|Морозова|anna.morozova@company.ru|3|5|NULL|110000|1|
|7|Сергей|Новиков|sergey.novikov@company.ru|3|1|6|45000|1|
|8|Екатерина|Фёдорова|katya.fedorova@company.ru|4|6|NULL|90000|1|
|9|Павел|Егоров|pavel.egorov@company.ru|5|4|NULL|150000|1|
|10|Наталья|Павлова|natalia.pavlova@company.ru|5|2|9|110000|1|

**Всего записей:** 10.

---

## 6. Таблица `projects`

**Назначение:** проекты, выполняемые компанией.

|Поле|Тип|Ограничения|Описание|
|---|---|---|---|
|`id`|INTEGER|PRIMARY KEY|Уникальный идентификатор проекта|
|`title`|VARCHAR(150)|NOT NULL|Название проекта|
|`start_date`|VARCHAR(10)|—|Дата начала|
|`end_date`|VARCHAR(10)|—|Дата окончания|
|`budget`|INTEGER|—|Бюджет проекта|

**Текущее содержимое таблицы:**

![[Pasted image 20260910141653.png]]

|id|title|start_date|end_date|budget|
|---|---|---|---|---|
|1|CRM-система|2024-01-10|2024-09-30|5000000|
|2|Мобильное приложение|2024-03-01|2025-01-15|3500000|
|3|Ребрендинг|2024-02-01|2024-06-30|1200000|
|4|Автоматизация бухгалтерии|2024-04-15|2024-12-01|800000|

**Всего записей:** 4.  
**Общий бюджет:** 10 500 000.

---

## 7. Таблица `project_members`

**Назначение:** связующая таблица «многие ко многим» между `employees` и `projects`.

|Поле|Тип|Ограничения|Описание|
|---|---|---|---|
|`employee_id`|INTEGER|NOT NULL, FK → `employees.id`, часть PK|Сотрудник|
|`project_id`|INTEGER|NOT NULL, FK → `projects.id`, часть PK|Проект|
|`role`|VARCHAR(50)|—|Роль сотрудника в проекте|

**Особенности:**

- **Составной первичный ключ** `(employee_id, project_id)` — один сотрудник не может участвовать в одном проекте дважды.
    
- При удалении сотрудника или проекта записи лучше удалять вручную (SQLite не поддерживает `ON DELETE CASCADE` по умолчанию — только если включить `PRAGMA foreign_keys = ON`).
    

**Текущее содержимое таблицы:**
![[Pasted image 20260910141704.png]]

|employee_id|project_id|role|
|---|---|---|
|1|1|Руководитель|
|2|1|Разработчик|
|2|2|Тимлид|
|3|1|Разработчик|
|3|2|Разработчик|
|4|3|Менеджер проекта|
|5|3|Маркетолог|
|6|4|Бухгалтер|
|7|4|Ассистент|

**Всего записей:** 9.

> 💡 Обратите внимание: сотрудник **Мария Сидорова (id=2)** участвует сразу в **двух проектах** — это иллюстрация связи «многие ко многим».

---

## 8. Связи и ограничения

### Внешние ключи (Foreign Keys)

|Таблица|Поле|Ссылается на|Тип связи|
|---|---|---|---|
|`employees`|`department_id`|`departments.id`|N:1|
|`employees`|`position_id`|`positions.id`|N:1|
|`employees`|`manager_id`|`employees.id`|N:1 (self)|
|`project_members`|`employee_id`|`employees.id`|N:1|
|`project_members`|`project_id`|`projects.id`|N:1|

### Кардинальности

- **Отдел → Сотрудники** — 1:N
    
- **Должность → Сотрудники** — 1:N
    
- **Сотрудник → Подчинённые** — 1:N (self-ref)
    
- **Сотрудник ↔ Проекты** — M:N (через `project_members`)
    

### Ограничения целостности

- `UNIQUE` на `departments.title`, `positions.title`, `employees.email`.
    
- `NOT NULL` на ключевых бизнес-полях.
    
- `DEFAULT 1` на `is_active`.
    
- Составной `PRIMARY KEY` в `project_members`.
    

> ⚠️ В SQLite **внешние ключи по умолчанию отключены**! Чтобы они работали, нужно выполнить:
> 
> sql
> 
> PRAGMA foreign_keys = ON;

---

## 9. Полный SQL создания
```sql
```-- Отделы
CREATE TABLE departments (
    id       INTEGER PRIMARY KEY,
    title    VARCHAR(100) NOT NULL UNIQUE,
    ceiling  INTEGER,
    phone    VARCHAR(20)
);
-- Должности
CREATE TABLE positions (
    id         INTEGER PRIMARY KEY,
    title      VARCHAR(100) NOT NULL UNIQUE,
    min_salary INTEGER,
    max_salary INTEGER
);
-- Сотрудники
CREATE TABLE employees (
    id            INTEGER PRIMARY KEY,
    first_name    VARCHAR(50)  NOT NULL,
    last_name     VARCHAR(50)  NOT NULL,
    email         VARCHAR(100) UNIQUE,
    phone         VARCHAR(30),
    birth_date    VARCHAR(10),
    hire_date     VARCHAR(10)  NOT NULL,
    department_id INTEGER,
    position_id   INTEGER,
    manager_id    INTEGER,
    salary        INTEGER,
    is_active     INTEGER DEFAULT 1,
    FOREIGN KEY (department_id) REFERENCES departments(id),
    FOREIGN KEY (position_id)   REFERENCES positions(id),
    FOREIGN KEY (manager_id)    REFERENCES employees(id)
);
-- Проекты
CREATE TABLE projects (
    id         INTEGER PRIMARY KEY,
    title      VARCHAR(150) NOT NULL,
    start_date VARCHAR(10),
    end_date   VARCHAR(10),
    budget     INTEGER
);
-- Участие в проектах
CREATE TABLE project_members (
    employee_id INTEGER NOT NULL,
    project_id  INTEGER NOT NULL,
    role        VARCHAR(50),
    PRIMARY KEY (employee_id, project_id),
    FOREIGN KEY (employee_id) REFERENCES employees(id),
    FOREIGN KEY (project_id)  REFERENCES projects(id)
);```
```

---

## 10. Примеры запросов

### 10.1 Полная карточка сотрудника



```sql
SELECT
    e.first_name || ' ' || e.last_name AS ФИО,
    d.title  AS отдел,
    p.title  AS должность,
    e.salary AS зарплата,
    e.hire_date AS принят
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
LEFT JOIN positions   p ON e.position_id   = p.id
WHERE e.id = 1;```
```


### 10.2 Организационная иерархия (кто кому подчиняется)

```sql
SELECT
    e.first_name || ' ' || e.last_name AS сотрудник,
    COALESCE(m.first_name || ' ' || m.last_name, '—') AS руководитель
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY m.last_name, e.last_name;```
```

### 10.3 Средняя зарплата по отделам


```sql
SELECT
    d.title AS отдел,
    COUNT(e.id) AS сотрудников,
    AVG(e.salary) AS средняя_зарплата
FROM departments d
JOIN employees e ON e.department_id = d.id
GROUP BY d.title
ORDER BY средняя_зарплата DESC;```
```

### 10.4 Кто в каком проекте участвует


```sql
SELECT
    pr.title AS проект,
    e.first_name || ' ' || e.last_name AS участник,
    pm.role AS роль
FROM project_members pm
JOIN employees e  ON pm.employee_id = e.id
JOIN projects  pr ON pm.project_id  = pr.id
ORDER BY pr.title, pm.role;```
```

### 10.5 Сотрудники без проектов


```sql
SELECT e.first_name, e.last_name
FROM employees e
LEFT JOIN project_members pm ON pm.employee_id = e.id
WHERE pm.employee_id IS NULL;```
```

### 10.6 Топ-3 зарплаты


```sql
SELECT first_name, last_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;```
```

### 10.7 Стаж работы в годах

```sql
SELECT
    first_name || ' ' || last_name AS ФИО,
    hire_date,
    CAST((julianday('now') - julianday(hire_date)) / 365.25 AS INTEGER) AS лет
FROM employees
ORDER BY лет DESC;```
```

---

## 11. Рекомендации по эксплуатации

### 11.1 Удаление данных

Порядок удаления (из-за FK):

1. `project_members`
    
2. `employees`
    
3. `projects`
    
4. `positions`
    
5. `departments`
    

Пример:
```sql
DELETE FROM project_members;
DELETE FROM employees;
DELETE FROM projects;
DELETE FROM positions;
DELETE FROM departments;```
```

### 11.2 Увольнение сотрудника

Физически не удалять — деактивировать:
```sql
UPDATE employees SET is_active = 0 WHERE id = ?;
```

### 11.3 Индексы для ускорения

```sql
CREATE INDEX idx_employees_department ON employees(department_id);
CREATE INDEX idx_employees_position   ON employees(position_id);
CREATE INDEX idx_employees_manager    ON employees(manager_id);
CREATE INDEX idx_pm_project           ON project_members(project_id);
CREATE INDEX idx_pm_employee          ON project_members(employee_id);```
```

### 11.4 Ограничения текущей модели

|Ограничение|Что можно улучшить|
|---|---|
|Даты хранятся как текст|Использовать `DATE`/`DATETIME` в других СУБД|
|Нет истории зарплат|Добавить таблицу `salary_history`|
|Нет учёта рабочего времени|Таблица `timesheet`|
|Нет отпусков/больничных|Таблица `leaves`|
|Нет KPI|Таблица `performance_reviews`|
|Нет аудита изменений|Таблица `audit_log` + триггеры|

### 11.5 Расширения модели (идеи)

- **`salary_history`** — все изменения зарплаты с датой и причиной.
    
- **`leaves`** — отпуска и больничные.
    
- **`timesheet`** — приход/уход, часы.
    
- **`performance_reviews`** — оценки эффективности.
    
- **`skills` / `employee_skills`** — навыки сотрудников.
    
- **`audit_log`** — кто и когда менял данные.
    

---

## 9 Приложение. Сводная таблица полей

|Таблица|Поле|Тип|PK|FK|NOT NULL|UNIQUE|DEFAULT|
|---|---|---|---|---|---|---|---|
|departments|id|INTEGER|✔||✔|||
|departments|title|VARCHAR(100)|||✔|✔||
|departments|ceiling|INTEGER||||||
|departments|phone|VARCHAR(20)||||||
|positions|id|INTEGER|✔||✔|||
|positions|title|VARCHAR(100)|||✔|✔||
|positions|min_salary|INTEGER||||||
|positions|max_salary|INTEGER||||||
|employees|id|INTEGER|✔||✔|||
|employees|first_name|VARCHAR(50)|||✔|||
|employees|last_name|VARCHAR(50)|||✔|||
|employees|email|VARCHAR(100)||||✔||
|employees|phone|VARCHAR(30)||||||
|employees|birth_date|VARCHAR(10)||||||
|employees|hire_date|VARCHAR(10)|||✔|||
|employees|department_id|INTEGER||✔||||
|employees|position_id|INTEGER||✔||||
|employees|manager_id|INTEGER||✔||||
|employees|salary|INTEGER||||||
|employees|is_active|INTEGER|||||1|
|projects|id|INTEGER|✔||✔|||
|projects|title|VARCHAR(150)|||✔|||
|projects|start_date|VARCHAR(10)||||||
|projects|end_date|VARCHAR(10)||||||
|projects|budget|INTEGER||||||
|project_members|employee_id|INTEGER|✔|✔|✔|||
|project_members|project_id|INTEGER|✔|✔|✔|||
|project_members|role|VARCHAR(50)||||||

---

## 📌 Приложение. Итоги по данным

|Таблица|Количество записей|
|---|---|
|departments|5|
|positions|7|
|employees|10|
|projects|4|
|project_members|9|
|**Итого**|**35**|

### Дополнительно

- **Общий бюджет проектов:** 10 500 000.
    
- **Средняя зарплата:** ~145 000.
    
- **Максимальная зарплата:** 250 000 (Иван Петров).
    
- **Минимальная зарплата:** 45 000 (Сергей Новиков, стажёр).
    
- **Сотрудники с руководителем:** 5.
    
- **Сотрудники без руководителя (топы):** 5.
