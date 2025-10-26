#4
```SQL
WITH FlightDistances AS (
    SELECT
        f.flight_id,
        f.aircraft_code,
        (
            6371 * acos(
                cos(radians(dep.coordinates[1])) 
                * cos(radians(arr.coordinates[1])) 
                * cos(radians(arr.coordinates[0]) - radians(dep.coordinates[0]))
                + sin(radians(dep.coordinates[1]))
                * sin(radians(arr.coordinates[1])) 
            )
        ) AS distance_km
    FROM flights f
    JOIN airports_data dep ON f.departure_airport = dep.airport_code
    JOIN airports_data arr ON f.arrival_airport = arr.airport_code
    WHERE f.status IN ('Arrived', 'Departed')
),
PricePerKm AS (
    SELECT 
        ad.model->>'ru' AS "Модель самолета",
        tf.fare_conditions AS "Класс",
        fd.distance_km,
        tf.amount,
        CASE 
            WHEN fd.distance_km > 0 THEN tf.amount / fd.distance_km 
            ELSE 0 
        END AS price_per_km
    FROM ticket_flights tf
    JOIN FlightDistances fd ON tf.flight_id = fd.flight_id
    JOIN aircrafts_data ad ON fd.aircraft_code = ad.aircraft_code
    WHERE tf.amount > 0 AND fd.distance_km > 100
)
SELECT
    "Модель самолета",
    "Класс",
    COUNT(*) AS "Кол-во билетов (выборка)",
    ROUND(AVG(price_per_km), 2) AS "Средняя цена за 1 км (для пасс.), руб.",
    ROUND(SUM(amount) / SUM(distance_km), 2) AS "Выручка за 1 км (для а/к), руб."
FROM PricePerKm
GROUP BY "Модель самолета", "Класс"
ORDER BY 
    "Класс", 
    "Средняя цена за 1 км (для пасс.), руб." ASC;
```
## Комментарии:
Сначала рассчитываю расстояние для кажд рейса (по ф.Гаверсинуса). Формула использует координаты (широту и долготу) аэропортов вылета и прилета.
WITH PricePerKm AS () соединяю ticket_flights (где есть цена билета amount) с FlightDistances (где есть distance_km). Рассчитываю стоимость одного километра полета (price_per_km) для каждого отдельного билета.
AVG(price_per_km) — ф-ция показыает выгоду для пассажира (средняя цена за 1 км).
SUM(amount) / SUM(distance_km) — показывает выгоду для авиакомпании (общая выручка с 1 км полета этого борта).