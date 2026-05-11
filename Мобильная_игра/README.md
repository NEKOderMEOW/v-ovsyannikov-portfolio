[← Назад к списку всех кейсов](../README.md)

# Анализ ключевых метрик мобильной игры

### Контекст кейса
* **Объект:** Схема базы данных мобильной игры (логи сессий, регистрации пользователей, транзакции.. [подробнее](https://docs.google.com/document/d/1lFgMy2D6xWvwF6IyISWk7LuWzQ_UWVtP5q7Y1mWJFis/edit?tab=t.0#heading=h.8ypi7f1o27w8)).
* **Специфика:** Платформы iOS и Android, наличие реферальной программы, внутриигровые покупки, акционные события.
* **Инструменты:** PostgreSQL (CTE, сложные JOIN-ы), Metabase, Google Slides.
---

### Основные этапы работы

#### 1. Анализ регистраций и игровой активности
* **Задача:** Оценить динамику притока пользователей и их игровое поведение (сессии, длительность, качество).
* **Решение:** 
  - Подсчёт уникальных пользователей и дат регистраций;
  - Расчёт общего числа сессий, доли сессий длительностью >5 минут, среднего времени в игре и доли сессий >1 часа;
  - Анализ реферальной программы: количество приглашений, доля регистраций по ним, среднее число инвайтов на пользователя, выделение топ-50 приглашающих.
* **Результат:**
  - За весь период зарегистрировано 3 078 человек;
  - С января 2023 года число регистраций выросло в 2,5 раза, март – рекорд (+48% к февралю);
  - Всего проведено >27 тыс. сессий, из них 71% качественных (>5 мин);
  - Среднее время в игре (без учёта коротких сессий) превышает 2,5 часа, а в марте взлетело до 5,5 часов;
  - Доля сессий дольше часа стабильна на уровне 70% после апрельской коррекции;
  - Почти 3 000 человек отправили около 8 000 приглашений, из которых 36% привели к регистрациям;
  - В среднем на одного приглашающего приходится 2,7 инвайта;
  - Топ-50 активных пользователей пригласили от 6 до 19 новых участников.
<details>
<summary>Посмотреть SQL-запросы</summary>

```sql
--кол-во пользователей и регистраций
SELECT COUNT(DISTINCT id_user) AS cnt_user
    ,  COUNT(*) AS cnt_reg
FROM skygame.users;

--пользователи с повторными регистрациями 
SELECT id_user
FROM skygame.users
GROUP BY id_user
HAVING COUNT(reg_date) > 1;

--анализ размаха дат регистраций и их отсутствия
SELECT  MIN(reg_date) AS min_reg_date
      , MAX(reg_date) AS max_reg_date
      , SUM(CASE 
               WHEN reg_date IS NULL 
               THEN 1 ELSE 0 END) 
                  AS without_reg
FROM skygame.users;

--распределение регистраций по месяцам
SELECT  date_trunc('month', reg_date) AS mm
      , COUNT(*) as cnt_reg
FROM skygame.users
GROUP BY mm
ORDER BY mm;

--количество всех игровых сессий и сессий дольше 5 минут 
SELECT COUNT(*) as cnt_all
     , SUM(CASE 
            WHEN end_session - start_session > INTERVAL '5 minute' 
            THEN 1 ELSE 0 END) AS cnt_signif
FROM skygame.game_sessions;

--динамика количества сессий, сессий дольше 5 минут и доли сессий дольше 5 минут среди всех сессий
SELECT date_trunc('month', start_session) AS mm
     , COUNT(*) as cnt_all
     , SUM(CASE 
            WHEN end_session - start_session > INTERVAL '5 minute' 
            THEN 1 ELSE 0 END) AS cnt_signif
     , SUM(CASE 
            WHEN end_session - start_session > INTERVAL '5 minute' 
            THEN 1.0 ELSE 0.0 END) / count(*) AS share_signif
FROM skygame.game_sessions
GROUP BY mm
ORDER BY mm;

--динамика среднего времени игровой сессии
SELECT date_trunc('month', start_session) AS mm
     , AVG(extract(epoch from end_session - start_session))/3600 AS avg_len
FROM skygame.game_sessions
WHERE end_session - start_session > INTERVAL '5 minute'
GROUP BY mm
ORDER BY mm;

--динамика доли игровых сессий дольше 1 часа
SELECT date_trunc('month', start_session) AS mm
     , SUM(CASE 
            WHEN end_session - start_session > INTERVAL '1 hour' 
            THEN 1.0 ELSE 0.0 END) / COUNT(*) AS share_long_session
FROM skygame.game_sessions
WHERE end_session - start_session > INTERVAL '5 minute'
GROUP BY mm
ORDER BY mm;

--количество приглашений, пригласивших пользователей и доля регистраций среди приглашений
SELECT COUNT(*) AS cnt_ref 
     , COUNT(DISTINCT id_user) AS cnt_user
     , SUM(ref_reg) / COUNT(*) AS share_reg
     , COUNT(*)::float / COUNT(DISTINCT id_user) AS avg_reg
FROM skygame.referral;

--топ-50 пользователей по количеству приглашений
SELECT id_user
     , COUNT(*) AS cnt_ref 
FROM skygame.referral
GROUP BY id_user
ORDER BY cnt_ref DESC
LIMIT 50;

```
</details>

#### 2. Выявление проблемы записи окончаний сессий
* **Задача:** Оценить масштаб аномалий (NULL-значения) и определить их привязку к платформе.
* **Решение:** Сравнение долей сессий с NULL-значениями в разрезе типа устройства.
* **Результат:**
  - 3 262 сессии (≈12%) без времени окончания;
  - На iOS доля ошибок — 19% (98% всех ошибок), на Android — всего 1%;
  - Проблема платформенная, требует приоритетного исправления на iOS.
<details>
<summary>Посмотреть SQL-запрос</summary>

```sql
--расчет количества ошибок и долей с ошибкой по устройствам среди всех сессий и каждого типа устройства по отдельности 
SELECT SUM(CASE 
            WHEN gs.end_session IS NULL 
            THEN 1 ELSE 0 END) AS cnt_end_null
     , SUM(CASE 
            WHEN gs.end_session IS NULL 
            THEN 1.0 ELSE 0.0 END)
		   / COUNT(*) AS share_end_null
     , SUM(CASE 
            WHEN gs.end_session IS NULL
               AND u.dev_type = 'ios'
            THEN 1.0 ELSE 0.0 END)
         / SUM(CASE 
                  WHEN u.dev_type = 'ios' 
                  THEN 1.0 ELSE 0.0 END) AS share_ios
     , SUM(CASE 
            WHEN gs.end_session IS NULL
               AND u.dev_type = 'android' 
            THEN 1.0 ELSE 0.0 END)
         / SUM(CASE 
                  WHEN u.dev_type = 'android' 
                  THEN 1.0 ELSE 0.0 END) AS share_android
     , SUM(CASE 
            WHEN gs.end_session IS NULL
				   AND u.dev_type = 'ios' 
            THEN 1.0 ELSE 0.0 END)
         / SUM(CASE 
                  WHEN gs.end_session IS NULL 
                  THEN 1.0 ELSE 0.0 END) AS share_ios_of_all_null
     , SUM(CASE 
            WHEN gs.end_session IS NULL
				   AND u.dev_type = 'android' 
            THEN 1.0 ELSE 0.0 END)
		   / SUM(CASE 
                  WHEN gs.end_session IS NULL 
                  THEN 1.0 ELSE 0.0 END) AS share_android_of_all_null
FROM skygame.game_sessions gs
	JOIN skygame.users u
		ON gs.id_user = u.id_user;

```
</details>

#### 3. Анализ мартовской акции (первые три недели марта)
* **Задача:** Оценить влияние акции на активность, удержание и "липкость" приложения.
* **Решение:** 
  - Расчёт DAU, WAU, MAU, недельного и месячного Sticky Factor;
  - Сравнение показателей до, во время и после акции.
* **Результат:**
  - MAU достиг 1 544 человек (в 3 раза выше среднего 2022 года);
  - Sticky Factor (месячный) упал до 10% из-за притока новых пользователей, которые не сформировали привычку;
  - Еженедельная "липкость" колебалась 21–30% в марте, к апрелю снизилась до уровней начала года;
  - Акция дала краткосрочный всплеск, но долгосрочное удержание остаётся низким.
<details>
<summary>Посмотреть SQL-запрос</summary>

```sql
--расчет DAU, WAU, MAU, Sticky Factor Weekly, Sticky Factor Monthly
WITH daily_stats AS (
   SELECT start_session::date AS dd
        , date_trunc('week', start_session) AS ww
        , date_trunc('month', start_session) AS mm
        , COUNT(DISTINCT id_user) AS dau
   FROM skygame.game_sessions
   GROUP BY 1, 2, 3
),
weekly_stats AS (
   SELECT date_trunc('week', start_session) AS ww
        , COUNT(DISTINCT id_user) AS wau
   FROM skygame.game_sessions
   GROUP BY 1
),
monthly_stats AS (
   SELECT date_trunc('month', start_session) AS mm
        , COUNT(DISTINCT id_user) AS mau
   FROM skygame.game_sessions
   GROUP BY 1
)

SELECT 
      d.dd AS dd
    , d.dau AS "DAU"
    , w.wau AS "WAU"
    , m.mau AS "MAU"
    , d.dau::float / w.wau AS "Sticky Factor Weekly" 
    , d.dau::float / m.mau AS "Sticky Factor Monthly"
FROM daily_stats d
JOIN weekly_stats w ON d.ww = w.ww
JOIN monthly_stats m ON d.mm = m.mm
WHERE w.ww <> '2022-06-27'
ORDER BY dd;

```
</details>

#### 4. Анализ монетизации (выручка по типам предметов и когортная щедрость)
* **Задача:** Изучить структуру дохода, влияние акции на продажи и ценность игроков разных когорт.
* **Решение:** 
  - Группировка выручки по типам предметов;
  - Когортный расчёт средней выручки на пользователя и средней выручки в месяц по когортам.
* **Результат:**
  - В марте выручка по категории «Currency» превысила 100 000 руб. (+100% к февралю);
  - Средняя выручка на пользователя за всё время стабильна (≈1000–1100 руб.);
  - Средняя выручка в месяц у мартовской когорты взлетела до 562 руб. — в 5 раз выше летних когорт 2022 года;
  - Категории «Транспорт» и «Оружие» — стабильно низкая выручка;
  - Монетизация сильно завязана на ивенты.
<details>
<summary>Посмотреть SQL-запросы</summary>

```sql
--динамика выручки в разрезе по типам игровых предметов 
SELECT date_trunc('month', dtime_pay) mm
     , type
     , SUM(cnt_buy * price) AS revenue
FROM skygame.monetary m
	JOIN skygame.item_list i
		ON m.id_item_buy = i.id_item
	JOIN skygame.log_prices p 
		ON p.id_item = i.id_item
		AND dtime_pay::date >= valid_from
		AND dtime_pay::date < COALESCE(
                              valid_to,
                              to_date('3000-01-01', 
                                      'YYYY-MM-DD')
                              )
GROUP BY mm, type 
ORDER BY type, mm  

--динамика общей средней выручки и средней выручки в месяц по когортам 
SELECT *
     , extract('day' from (SELECT max(dtime_pay) from skygame.monetary) 
         - mm) / 30 as interv
     , avg_rev / (extract('day' from
         (SELECT max(dtime_pay) from skygame.monetary) - mm) / 30) 
            AS avg_rev_per_month
FROM 
(
SELECT date_trunc('month', reg_date) mm
     , SUM(cnt_buy * price) AS revenue
     , COUNT(DISTINCT u.id_user) AS cnt
     , SUM(cnt_buy * price) / COUNT(DISTINCT u.id_user) AS avg_rev
FROM skygame.monetary m
	JOIN skygame.users u
		ON m.id_user = u.id_user
	JOIN skygame.log_prices p 
		ON p.id_item = m.id_item_buy
		AND dtime_pay::date >= valid_from
		AND dtime_pay::date < COALESCE(
                              valid_to, 
                              to_date('3000-01-01', 
                                      'YYYY-MM-DD')
                              )
WHERE reg_date < (SELECT max(dtime_pay) from skygame.monetary) - interval '1 month'
GROUP BY mm 
ORDER BY mm
) t

```
</details>

#### 5. Оценка изменения цены кристаллов (с января 2023 года)
* **Задача:** Проанализировать влияние повышения цены на частоту покупок и выручку от кристаллов.
* **Решение:** Построение динамики среднего количества покупок и общей выручки по месяцам для кристаллов.
* **Результат:**
  - В марте выручка от кристаллов достигла пика 37 840 руб. (в 3 раза выше среднего);
  - Количество покупок на пользователя снизилось с 30 до 22, но средний чек вырос;
  - В апреле выручка стабилизировалась на 13 200 руб. — адаптация прошла успешно, решение экономически обоснованно.
<details>
<summary>Посмотреть SQL-запрос</summary>

```sql
--динамика среднего числа покупок кристаллов и выручки с их продажи
SELECT date_trunc('month', dtime_pay) mm
     , AVG(cnt_buy) AS avg_cnt
     , SUM(cnt_buy * price) AS revenue
FROM skygame.monetary m
	JOIN skygame.item_list i
		ON m.id_item_buy = i.id_item
	JOIN skygame.log_prices p 
		ON p.id_item = i.id_item
		AND dtime_pay::date >= valid_from
		AND dtime_pay::date < COALESCE(
                              valid_to, 
                              to_date('3000-01-01', 
                                      'YYYY-MM-DD')
                              )
WHERE name_item = 'Crystal'
GROUP BY mm, type 
ORDER BY mm, type 

```
</details>

#### 6. Анализ вирусности (K-фактор и ожидаемый объем когорты)
* **Задача:** Измерить органический рост за счёт рефералов и спрогнозировать объём новой когорты.
* **Решение:** Вычисление K-фактора, исторического среднего объема когорты и прогноз объёма будущей когорты перемножением этих двух показателей.
* **Результат:**
  - K-фактор = 92% (почти каждый пользователь приводит ещё одного);
  - Более 90% аудитории (2 834 из 3 078) пришли по реферальным ссылкам;
  - Исторический средний размер когорты — 342 пользователя;
  - Прогнозируемый объём новой когорты — около 315;
  - Реферальная механика — главный драйвер роста аудитории.
<details>
<summary>Посмотреть SQL-запрос</summary>

```sql
--расчет ожидаемого объема будущей когорты на основе к-фактора и исторического среднего объема когорт
WITH k_factor AS
(
SELECT SUM(ref_reg) / COUNT(DISTINCT u.id_user)
FROM skygame.users u
	LEFT JOIN skygame.referral r
		ON u.id_user = r.id_user
),
avg_hist_cohort AS
(
SELECT COUNT(DISTINCT id_user)::float
		/ COUNT(DISTINCT date_trunc('month', reg_date))
FROM skygame.users
)
SELECT
   (SELECT * FROM k_factor) *
   (SELECT * FROM avg_hist_cohort) AS predict_cohort;

```
</details>

#### 7. Оценка канала привлечения (опробован в ноябре–декабре 2022 года)
* **Задача:** Сравнить эффективность тестового канала с остальными источниками по глубине вовлечения.
* **Решение:** Сравнение среднего времени игры для пользователей, зарегистрировавшихся в ноябре–декабре 2022, и всех остальных.
* **Результат:**
  - Пользователи из тестового канала проводят в игре в среднем 3 часа 37 минут — на 22% дольше, чем остальная аудитория;
  - Канал признан приоритетным для дальнейшего развития.
<details>
<summary>Посмотреть SQL-запрос</summary>

```sql
--сравнение среднего времени в игре для пользователей с регистрацией в ноябре-декабре 2022 и всех остальных 
SELECT CASE
         WHEN reg_date BETWEEN '2022-11-01' AND '2022-12-31'
         THEN 1 ELSE 0 END AS flag_cohort
    ,  AVG(end_session - start_session) AS avg_game_time
FROM skygame.users u
	LEFT JOIN skygame.game_sessions g
		ON u.id_user = g.id_user
		AND (end_session - start_session) > INTERVAL '5 minute'
GROUP BY flag_cohort;

```
</details>

#### 8. Анализ лояльности игроков (по трём критериям)
* **Задача:** Определить размер и динамику лояльного ядра: активные рефералы, плательщики с выручкой >1000 руб., топ‑100 по среднемесячной выручке (LTR/LT).
* **Решение:** 
  - Построение динамики месячной активной аудитории (LMAU) для пользователей, удовлетворяющих:
    - **Реферальная лояльность:** минимум 3 приглашения, из них хотя бы одна регистрация;
    - **Финансовая лояльность:** суммарная выручка >1000 руб.;
    - **И/ИЛИ** комбинации первых двух критериев;
    - **Топ-100** по среднемесячной выручке.
* **Результат:**
  - В марте число лояльных по рефералам достигло 532 чел. (+112% к февралю), финансово лояльных — 184 чел. (+100%);
  - "Мульти-лояльных" (оба критерия) — 61 чел. (+100%);
  - После акции показатели скорректировались, но остались выше: реферальная лояльность на 36% выше, финансовая — на 44% выше, чем до акции;
  - Топ‑100 по среднемесячной выручке вырос с 7 до 38 человек и не снизился в апреле — сформировалось устойчивое платёжеспособное ядро.
<details>
<summary>Посмотреть SQL-запросы</summary>

```sql
--LMAU по критерию приглашений
WITH crit_invite AS (
SELECT id_user
     , COUNT(*) AS cnt_invite 
     , SUM(ref_reg) AS cnt_reg
FROM skygame.referral 
GROUP BY id_user 
HAVING COUNT(*) >= 3
	AND SUM(ref_reg) >= 1
)
SELECT date_trunc('month', start_session) AS mm 
     , COUNT(DISTINCT id_user) AS "LMAU"
FROM skygame.game_sessions
WHERE id_user IN (SELECT id_user FROM crit_invite)
GROUP BY mm
ORDER BY mm;

--LMAU по критерию выручки
WITH crit_1000 AS (
SELECT id_user 
     , SUM(cnt_buy * price) AS revenue
FROM skygame.monetary m
	JOIN skygame.log_prices l 
		ON m.id_item_buy = l.id_item 
		AND dtime_pay >= valid_from 
		AND dtime_pay <= COALESCE(valid_to, '3000-01-01')
GROUP BY m.id_user 
HAVING SUM(cnt_buy * price) > 1000 
)
SELECT date_trunc('month', start_session) AS mm 
     , COUNT(DISTINCT id_user) AS "LMAU"
FROM skygame.game_sessions
WHERE id_user IN (SELECT id_user FROM crit_1000)
GROUP BY mm
ORDER BY mm;

--LMAU по критерию приглашений И критерию выручки
WITH crit_invite AS (
SELECT id_user
     , COUNT(*) AS cnt_invite 
     , SUM(ref_reg) AS cnt_reg
FROM skygame.referral 
GROUP BY id_user 
HAVING COUNT(*) >= 3
	AND SUM(ref_reg) >= 1
), crit_1000 AS (
SELECT id_user 
     , SUM(cnt_buy * price) AS revenue
FROM skygame.monetary m
	JOIN skygame.log_prices l 
		ON m.id_item_buy = l.id_item 
		AND dtime_pay >= valid_from 
		AND dtime_pay <= COALESCE(valid_to, '3000-01-01')
GROUP BY m.id_user 
HAVING SUM(cnt_buy * price) > 1000 
)
SELECT date_trunc('month', start_session) AS mm 
     , COUNT(DISTINCT id_user) AS "LMAU"
FROM skygame.game_sessions
WHERE id_user IN (SELECT id_user FROM crit_invite)
	AND id_user IN (SELECT id_user FROM crit_1000)
GROUP BY mm
ORDER BY mm;

--LMAU по критерию приглашений ИЛИ критерию выручки
WITH crit_invite AS (
SELECT id_user
     , COUNT(*) AS cnt_invite 
     , SUM(ref_reg) AS cnt_reg
FROM skygame.referral 
GROUP BY id_user 
HAVING COUNT(*) >= 3
	AND SUM(ref_reg) >= 1
), crit_1000 AS (
SELECT id_user 
     , SUM(cnt_buy * price) AS revenue
FROM skygame.monetary m
	JOIN skygame.log_prices l 
		ON m.id_item_buy = l.id_item 
		AND dtime_pay >= valid_from 
		AND dtime_pay <= COALESCE(valid_to, '3000-01-01')
GROUP BY m.id_user 
HAVING SUM(cnt_buy * price) > 1000 
)
SELECT date_trunc('month', start_session) AS mm 
     , COUNT(DISTINCT id_user) AS "LMAU"
FROM skygame.game_sessions
WHERE id_user IN (SELECT id_user FROM crit_invite)
	OR id_user IN (SELECT id_user FROM crit_1000)
GROUP BY mm
ORDER BY mm;

--LMAU по топ-100 пользователей по средней выплате за месяц
WITH LTR AS (
SELECT id_user 
     , SUM(cnt_buy * price) AS revenue
FROM skygame.monetary m
	JOIN skygame.log_prices l 
		ON m.id_item_buy = l.id_item 
		AND dtime_pay >= valid_from 
		AND dtime_pay <= COALESCE(valid_to, '3000-01-01')
GROUP BY m.id_user  
), LT_mm AS (
SELECT u.id_user
     , CEIL(EXTRACT('day' FROM MAX(start_session) - MIN(reg_date)) / 30) AS LT_mm
FROM skygame.users u
	JOIN skygame.game_sessions g 
		ON u.id_user = g.id_user 
GROUP BY u.id_user 
), crit_ltr_mm AS (
SELECT LTR.id_user
     , revenue / LT_mm AS ltr_mm
     , LT_mm
FROM LTR JOIN LT_mm 
	ON LTR.id_user = LT_mm.id_user
ORDER BY ltr_mm DESC
LIMIT 100
)
SELECT date_trunc('month', start_session) AS mm 
     , COUNT(DISTINCT id_user) AS "LMAU"
FROM skygame.game_sessions
WHERE id_user IN (SELECT id_user FROM crit_ltr_mm)
GROUP BY mm
ORDER BY mm;

```
</details>

---

### Файлы и материалы
* **Презентация результатов:** [Открыть в Google Slides](https://docs.google.com/presentation/d/1NhBwX8ZQn8mMzsKhMjGvBvpv_E9R-7ocEZsegIdwVSk/edit?slide=id.g3df79e86344_0_7#slide=id.g3df79e86344_0_7) - ключевые выводы для руководства.
* **SQL-запросы:** [Открыть в Google Docs](https://docs.google.com/document/d/1lFgMy2D6xWvwF6IyISWk7LuWzQ_UWVtP5q7Y1mWJFis/edit?usp=drive_link) - полный скрипт со всеми запросами.
