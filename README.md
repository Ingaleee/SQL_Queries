## Задание 1: 100 заданий с самым долгим временем выполнения
Время, затраченное на выполнение задания - это период времени, прошедший с момента перехода задания в статус "В работе" и до перехода в статус "Выполнено".
Нужно вывести 100 заданий с самым долгим временем выполнения. 
Полученный список заданий должен быть отсортирован от заданий с наибольшим временем выполнения к заданиям с наименьшим временем выполнения.

Замечания:
- Невыполненные задания (не дошедшие до статуса "Выполнено") не учитываются.
- Когда исполнитель берет задание в работу, оно переходит в статус "В работе" (InProgress) и находится там до завершения работы. После чего переходит в статус "Выполнено" (Done).
  В любой момент времени задание может быть безвозвратно отменено - в этом случае оно перейдет в статус "Отменено" (Canceled).
- Нет разницы выполняется задание или подзадание.
- Выборка должна включать задания за все время.

Выборка должна содержать следующий набор полей:
- номер задания (task_number)
- заголовок задания (task_title)
- название статуса задания (status_name)
- email автора задания (author_email)
- email текущего исполнителя (assignee_email)
- дата и время создания задания (created_at)
- дата и время первого перехода в статус В работе (in_progress_at)
- дата и время выполнения задания (completed_at)
- количество дней, часов, минут и секнуд, которые задание находилось в работе - в формате "dd HH:mm:ss" (work_duration)

### Решение
```sql
WITH
    RankedTasks
    AS
    (
        SELECT
            t.number AS task_number,
            t.title AS task_title,
            ts.name AS status_name,
            u.email AS author_email,
            ua.email AS assignee_email,
            t.created_at,
            tl_in.at AS in_progress_at,
            tl_done.at AS completed_at,
            EXTRACT(DAY FROM (tl_done.at - tl_in.at)) || ' ' || 
        TO_CHAR(tl_done.at - tl_in.at, 'HH24:MI:SS') AS work_duration
        FROM
            tasks t
            JOIN task_statuses ts ON t.status = ts.id
            JOIN users u ON t.created_by_user_id = u.id
            LEFT JOIN users ua ON t.assigned_to_user_id = ua.id
            JOIN task_logs tl_in ON t.id = tl_in.task_id AND tl_in.status = (SELECT
                    id
                FROM
                    task_statuses
                WHERE alias = 'InProgress')
            JOIN task_logs tl_done ON t.id = tl_done.task_id AND tl_done.status = (SELECT
                    id
                FROM
                    task_statuses
                WHERE alias = 'Done')
        WHERE t.status = (SELECT
            id
        FROM
            task_statuses
        WHERE alias = 'Done')
        ORDER BY (tl_done.at - tl_in.at) DESC
    LIMIT 100
)
SELECT
    task_number
    
    ,
    task_title,
    status_name,
    author_email,
    assignee_email,
    TO_CHAR
(created_at, 'DD.MM.YYYY HH24:MI:SS') AS created_at,
    TO_CHAR
(in_progress_at, 'DD.MM.YYYY HH24:MI:SS') AS in_progress_at,
    TO_CHAR
(completed_at, 'DD.MM.YYYY HH24:MI:SS') AS completed_at,
    work_duration
FROM RankedTasks;

```

## Задание 2: Выборка для проверки вложенности
Задания могу быть простыми и составными. Составное задание содержит в себе дочерние - так получается иерархия заданий.
Глубина иерархии ограничено Н-уровнями, поэтому перед добавлением подзадачи к текущей задачи нужно понять, может ли пользователь добавить задачу уровнем ниже текущего или нет. Для этого нужно написать выборку для метода проверки перед добавлением подзадания, которая бы вернула уровень вложенности указанного задания и полный путь до него от родительского задания.

Замечания:
- ИД проверяемого задания передаем в sql как параметр _:parent_task_id_
- если задание _Е_ находится на 5м уровне, то путь должен быть "_//A/B/C/D/E_".

Выборка должна содержать:
- только 1 строку
- поле "Уровень задания" (level) - уровень указанного в параметре задания
- поле "Путь" (path)

### Решение
```sql
WITH RECURSIVE TaskHierarchy AS
    (
        SELECT
        id,
        parent_task_id,
        title,
        1 AS level,
        title AS path
    FROM
        tasks
    WHERE id = :parent_task_id
UNION ALL
    SELECT
        t.id,
        t.parent_task_id,
        t.title,
        th.level + 1,
        th.path || '//' || t.title
    FROM
        tasks t
        JOIN TaskHierarchy th ON t.parent_task_id = th.id
)
SELECT
    MAX(level) AS level,
    MAX(path) AS path
FROM
    TaskHierarchy;
```

## Задание 3: Денормализация
Наш трекер задач пользуется популярностью и количество только активных задач перевалило уже за несколько миллионов. Продакт пришел с очередной юзер-стори:
```
Я как Диспетчер в списке активных задач всегда должен видеть задачу самого высокого уровня из цепочки отдельным полем

Требования:
1. Список активных задач включает в себя задачи со статусом "В работе"
2. Список должен быть отсортирован от самой новой задачи к самой старой
3. В списке должны быть поля:
  - Номер задачи (task_number)
  - Заголовок (task_title)
  - Номер родительской задачи (parent_task_number)
  - Заголовок родительской задачи (parent_task_title)
  - Номер самой первой задачи из цепочки (root_task_number)
  - Заголовок самой первой задачи из цепочки (root_task_title)
  - Email, на кого назначена задача (assigned_to_email)
  - Когда задача была создана (created_at)
 4. Должна быть возможность получить данные с пагинацией по N-строк (@limit, @offset)
```

Обсудив требования с лидом тебе прилетели 2 задачи:
1. Данных очень много и нужно денормализовать таблицу tasks
   Добавить в таблицу tasks поле `root_task_id bigint not null`
   Написать скрипт заполнения нового поля root_task_id для всей таблицы (если задача является рутом, то ее id должен совпадать с root_task_id)
2. Написать запрос получения данных для отображения в списке активных задач
   (!) Выяснилось, что дополнительно еще нужно возвращать идентификаторы задач, чтобы фронтенд мог сделать ссылки на них (т.е. добавить поля task_id, parent_task_id, root_task_id)

<details>
  <summary>FAQ</summary>

**Q: Что такое root_task_id?**

A: Например, есть задача с id=10 и parent_task_id=9, задача с id=9 имеет parent_task_id=8 и т.д. до задача id=1 имеет parent_task_id=null. Для всех этих задач root_task_id=1.

**Q: Не понял в каком формате нужен результат?**

Ожидаемый результат выполнения SQL-запроса:

| task_id | task_number | task_title | parent_task_id | parent_task_number | parent_task_title | root_task_id | root_task_number | root_task_title | assigned_to_email | created_at          |
|---------|-------------|------------|----------------|--------------------|-------------------|--------------|------------------|-----------------|-------------------|---------------------|
| 1       | A123        | Тест 123   | null           | null               | null              | 1            | A123             | Тест 123        | test@test.tt      | 01.01.2023 08:00:00 |
| 2       | B123        | B-тест     | 1              | A123               | Тест 123          | 1            | A123             | Тест 123        | user@test.tt      | 01.01.2023 11:00:00 |
| 3       | C123        | 123-тест   | 2              | B123               | B-тест            | 1            | A123             | Тест 123        | dev@test.tt       | 01.01.2023 11:10:00 |
| 10      | 1-2345      | New task   | null           | null               | null              | 10           | 1-2345           | New task        | test@test.tt      | 12.02.2024 11:00:00 |

**Q: Все это можно делать в одной миграции?**

А: Нет, каждая DDL операция - отдельная миграция, DML-операция тоже долзна быть в отдельной миграции.

</details>

### Скрипты миграций
```sql
ALTER TABLE tasks ADD COLUMN root_task_id BIGINT NOT NULL DEFAULT 0;

WITH RECURSIVE RootTasks AS
    (
        SELECT
        id,
        parent_task_id,
        id AS root_task_id
    FROM
        tasks
    WHERE parent_task_id IS NULL
UNION ALL
    SELECT
        t.id,
        t.parent_task_id,
        rt.root_task_id
    FROM
        tasks t
        JOIN RootTasks rt ON t.parent_task_id = rt.id
)
UPDATE tasks t
SET root_task_id
= rt.root_task_id
FROM RootTasks rt
WHERE t.id = rt.id;
```

### Запрос выборки
```sql
SELECT
    t.id AS task_id,
    t.number AS task_number,
    t.title AS task_title,
    pt.id AS parent_task_id,
    pt.number AS parent_task_number,
    pt.title AS parent_task_title,
    rt.id AS root_task_id,
    rt.number AS root_task_number,
    rt.title AS root_task_title,
    (SELECT
        email
    FROM
        users
    WHERE id = t.assigned_to_user_id) AS assigned_to_email,
    TO_CHAR(t.created_at, 'DD.MM.YYYY HH24:MI:SS') AS created_at
FROM
    tasks t
    LEFT JOIN tasks pt ON t.parent_task_id = pt.id
    LEFT JOIN tasks rt ON t.root_task_id = rt.id
WHERE t.status = (SELECT
    id
FROM
    task_statuses
WHERE alias = 'InProgress')
ORDER BY t.created_at DESC
LIMIT :limit
OFFSET :offset;
```
