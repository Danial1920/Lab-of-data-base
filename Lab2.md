#4
```SQL
WITH FlightDistances AS (
    SELECT
        f.flight_id,
        f.aircraft_code,
        CASE 
            WHEN dep.coordinates IS NOT NULL AND arr.coordinates IS NOT NULL THEN
                6371 * acos(
                    GREATEST(-1, LEAST(1, 
                        cos(radians((dep.coordinates::point)[1])) 
                        * cos(radians((arr.coordinates::point)[1])) 
                        * cos(radians((arr.coordinates::point)[0]) - radians((dep.coordinates::point)[0]))
                        + sin(radians((dep.coordinates::point)[1]))
                        * sin(radians((arr.coordinates::point)[1]))
                    ))
                )
            ELSE NULL
        END AS distance_km
    FROM flights f
    JOIN airports_data dep ON f.departure_airport = dep.airport_code
    JOIN airports_data arr ON f.arrival_airport = arr.airport_code
    WHERE f.status IN ('Arrived', 'Departed')
),
PricePerKm AS (
    SELECT 
        COALESCE(ad.model->>'ru', ad.model->>'en', ad.model::text) AS "Модель самолета",
        tf.fare_conditions AS "Класс",
        fd.distance_km,
        tf.amount,
        CASE 
            WHEN fd.distance_km > 0 THEN tf.amount / fd.distance_km 
            ELSE NULL 
        END AS price_per_km
    FROM ticket_flights tf
    JOIN FlightDistances fd ON tf.flight_id = fd.flight_id
    JOIN aircrafts_data ad ON fd.aircraft_code = ad.aircraft_code
    WHERE tf.amount > 0 AND fd.distance_km > 100  
),
AggregatedData AS (
    SELECT
        "Модель самолета",
        "Класс",
        COUNT(*) AS "Кол-во билетов",
        ROUND(AVG(price_per_km)::numeric, 2) AS "Средняя цена за 1 км (для пасс.), руб.",
        ROUND((SUM(amount) / SUM(distance_km))::numeric, 2) AS "Выручка за 1 км (для а/к), руб.",
        COUNT(*) * AVG(price_per_km) AS estimated_total_revenue
    FROM PricePerKm
    WHERE price_per_km IS NOT NULL
    GROUP BY "Модель самолета", "Класс"
),
RankedData AS (
    SELECT *,
        RANK() OVER (
            PARTITION BY "Класс" 
            ORDER BY "Средняя цена за 1 км (для пасс.), руб." ASC
        ) AS passenger_rank,
        RANK() OVER (
            PARTITION BY "Класс" 
            ORDER BY "Выручка за 1 км (для а/к), руб." DESC
        ) AS airline_rank
    FROM AggregatedData
)
SELECT
    "Модель самолета",
    "Класс",
    "Кол-во билетов",
    "Средняя цена за 1 км (для пасс.), руб.",
    "Выручка за 1 км (для а/к), руб.",
    CASE 
        WHEN passenger_rank = 1 THEN 'ЛУЧШИЙ ДЛЯ ПАССАЖИРА'
        ELSE '#' || passenger_rank::text
    END AS "Рейтинг для пассажира",
    CASE 
        WHEN airline_rank = 1 THEN 'ЛУЧШИЙ ДЛЯ АВИАКОМПАНИИ'
        ELSE '#' || airline_rank::text
    END AS "Рейтинг для а/к"
FROM RankedData
WHERE "Кол-во билетов" >= 10  -- Мин. выборка для стат. знач
ORDER BY 
    "Класс",
    "Средняя цена за 1 км (для пасс.), руб." ASC,
    "Выручка за 1 км (для а/к), руб." DESC;
```

## Комментарии:
### 1. FlightDistances 
* Для точного расчета использую ф.Гаверсинуса
* Функция GREATEST/LEAST для защиты от ошибок округления при расчете acos

---

### 2. PricePerKm 
* Рассчитана индивид. стоимость 1 км (`tf.amount / fd.distance_km`) для каждого билета.
* Фильтруем с помощтю "WHERE tf.amount > 0 AND fd.distance_km > 100" (исключаю  короткие рейсы повышая точность)
* (`COALESCE(model->>'ru', ...)`) - вывод модели на русском яз

---

### 3. AggregatedData 
* Группирую По "модель самолета" и "класс"
* AVG(price_per_km) — Средняя цена за 1 км
* SUM(amount) / SUM(distance_km) — Выручка за 1 км
* Для корректного округления приведены к ::numeric

---

### 4. RankedData
* Использую оконную ф-ию RANK() OVER.
* Присваиваю ранги ТОЛЬКО внутри кажд класса (`PARTITION BY "Класс"`).
* Логика работы:
    * passenger_rank: Сортировка по цене ASC (наим цена = ранг №1).
    * airline_rank: Сортировка по выручке DESC (наиб выручка = ранг №1).

---

### 5. Финальный SELECT 
* Оставляю онли стат значимые комбинации (`WHERE "Кол-во билетов" >= 10`).
* Использование CASE для маркировки лучших вариантов (`ЛУЧШИЙ ДЛЯ...`) на основе рангов
* Упоряд. по классу и затем по экономичности для пассажира.