[← Назад к списку всех кейсов](../README.md)

# Анализ ключевых метрик мобильной игры

### Контекст кейса
* **Объект:** База данных игрового приложения (логи сессий, транзакции, данные пользователей).
* **Цель:** Расчет продуктовых метрик (Retention, DAU/WAU/MAU) и анализ монетизации для принятия решений о развитии продукта.
* **Инструменты:** PostgreSQL (сложные JOIN, обобщенные табличные выражения - CTE, оконные функции).

---

### Основные этапы работы

1. **Расчет аудиторных метрик (DAU, WAU, MAU)**
    * **Решение:** Группировка уникальных пользователей по временным интервалам с использованием `COUNT(DISTINCT)`.
    * **Результат:** Определен размер активной аудитории и коэффициент "липкости" (Sticky Factor), отражающий регулярность возврата игроков.
    <details>
    <summary>Посмотреть SQL-запрос</summary>

    ```sql
    SELECT 
        date_trunc('day', login_time) AS day,
        count(distinct user_id) as dau
    FROM user_logins
    GROUP BY 1;
    ```
    </details>

2. **Анализ удержания (Retention Rate)**
    * **Решение:** Использование Self-Join для сопоставления даты регистрации и дат последующей активности.
    * **Результат:** Построены кривые удержания. Выявлены "дни оттока", после которых пользователи чаще всего покидают игру.
    <details>
    <summary>Посмотреть SQL-запрос</summary>

    ```sql
    -- Пример запроса для Retention 1-го дня
    WITH first_login AS (
        SELECT user_id, MIN(login_time)::date as reg_date
        FROM logins GROUP BY 1
    )
    SELECT 
        f.reg_date,
        COUNT(DISTINCT f.user_id) as starters,
        COUNT(DISTINCT l.user_id) as day_1_retention
    FROM first_login f
    LEFT JOIN logins l ON f.user_id = l.user_id 
        AND l.login_time::date = f.reg_date + interval '1 day'
    GROUP BY 1;
    ```
    </details>

3. **Анализ монетизации и среднего чека (ARPU, ARPPU)**
    * **Решение:** Расчет дохода на одного активного и одного платящего пользователя.
    * **Результат:** Найдены наиболее прибыльные сегменты игроков и оценена эффективность внутриигровых покупок.

---

### Файлы и материалы
* **Презентация результатов:** [Скачать в PPTX](https://docs.google.com/presentation/d/1iY146dJpi-KyNVKGmeHSyxf07_yASqGu/export) - ключевые выводы для руководства.
* **SQL-запросы:** [Открыть в Google Docs](https://docs.google.com/document/d/1lFgMy2D6xWvwF6IyISWk7LuWzQ_UWVtP5q7Y1mWJFis/edit?usp=drive_link) - полный скрипт со всеми запросами.
