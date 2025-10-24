#1
```SQL
WITH PassengerFlights AS (
    SELECT 
        tf.ticket_no,
        f.flight_id,
        f.departure_airport,
        f.arrival_airport,
        f.scheduled_departure,
        f.scheduled_arrival,
        LEAD(f.departure_airport) OVER(PARTITION BY tf.ticket_no ORDER BY f.scheduled_departure) AS next_departure_airport,
        LEAD(f.scheduled_departure) OVER(PARTITION BY tf.ticket_no ORDER BY f.scheduled_departure) AS next_departure_time
    FROM ticket_flights tf
    JOIN flights f ON tf.flight_id = f.flight_id
    WHERE f.status IN ('Arrived', 'Departed', 'On Time') 
),
TransitFlow AS (
    SELECT 
        ticket_no,
        arrival_airport AS transit_airport,
        EXTRACT(EPOCH FROM (next_departure_time - scheduled_arrival)) / 3600 AS connection_hours
    FROM PassengerFlights
    WHERE 
        arrival_airport = next_departure_airport
        AND next_departure_time IS NOT NULL
        AND (next_departure_time - scheduled_arrival) BETWEEN INTERVAL '1 hour' AND INTERVAL '24 hours'
),
AirportTransitStats AS (
    SELECT
        transit_airport,
        COUNT(DISTINCT ticket_no) AS transit_passengers_count,
        AVG(connection_hours) AS avg_connection_hours
    FROM TransitFlow
    GROUP BY transit_airport
),
AirportTotalFlow AS (
    SELECT 
        airport_code,
        COUNT(DISTINCT ticket_no) AS total_unique_passengers
    FROM (
        SELECT departure_airport AS airport_code, ticket_no FROM flights f JOIN ticket_flights tf ON f.flight_id = tf.flight_id WHERE f.status <> 'Cancelled'
        UNION ALL
        SELECT arrival_airport AS airport_code, ticket_no FROM flights f JOIN ticket_flights tf ON f.flight_id = tf.flight_id WHERE f.status <> 'Cancelled'
    ) AS all_movements
    GROUP BY airport_code
)
SELECT 
    a.airport_name->>'ru' AS "Аэропорт (Хаб)",
    a.city->>'ru' AS "Город",
    COALESCE(ats.transit_passengers_count, 0) AS "Кол-во транзитных пассажиров",
    atf.total_unique_passengers AS "Общий поток (уник. пасс.)",
    CASE 
        WHEN atf.total_unique_passengers > 0 
        THEN ROUND((COALESCE(ats.transit_passengers_count, 0)::decimal / atf.total_unique_passengers) * 100, 2)
        ELSE 0 
    END AS "Доля транзита, %",
    ROUND(COALESCE(ats.avg_connection_hours, 0), 1) AS "Среднее время стыковки, ч"
FROM airports_data a
JOIN AirportTotalFlow atf ON a.airport_code = atf.airport_code 
LEFT JOIN AirportTransitStats ats ON a.airport_code = ats.transit_airport
ORDER BY "Кол-во транзитных пассажиров" DESC;
```
##Комментарии:
Сначала собираю все перелеты, принадлежащие одному билету. 
С помощью LEAD() подтягиваю информацию о след рейсе в рамках этого же билета, чтобы найти потенциальные стыковки.
После Фильтруем данные из PassengerFlights, оставляя только те пары рейсов, где аэропорт прилета первого совпадает с аэропортом вылета второго. Там же рассчитываю время стыковки.
Далее группирую всех транзитных пассажиров по аэропорту стыковки и считаем их уникальное кол-во, 
а также среднее время ожидания
И считаю общий пассажиропоток аэропорта — всех уник. пассажиров, кто хотя бы раз прилетал в него или вылетал из него
Объединяю статистику по транзиту и общ потоку , добавляю  названия аэропортов и рассчитываю долю транзита
#2
```SQL
WITH AircraftStats AS (
    SELECT 
        s.aircraft_code,
        ad.model->>'ru' as model_name,
        COUNT(s.seat_no) AS total_capacity,
        SUM(CASE WHEN s.fare_conditions = 'Business' THEN 1 ELSE 0 END) AS business_capacity
    FROM seats s
    JOIN aircrafts_data ad ON s.aircraft_code = ad.aircraft_code
    GROUP BY s.aircraft_code, ad.model
),
RouteDemand AS (
    SELECT
        f.departure_airport,
        f.arrival_airport,
        f.aircraft_code AS actual_aircraft_code,
        AVG(tf.tickets_sold) AS avg_total_pax,
        AVG(tf.business_tickets_sold) AS avg_business_pax
    FROM flights f
    JOIN (
        SELECT 
            flight_id, 
            COUNT(ticket_no) AS tickets_sold,
            SUM(CASE WHEN fare_conditions = 'Business' THEN 1 ELSE 0 END) AS business_tickets_sold
        FROM ticket_flights
        GROUP BY flight_id
    ) tf ON f.flight_id = tf.flight_id
    WHERE f.status IN ('Arrived', 'Departed')
    GROUP BY f.departure_airport, f.arrival_airport, f.aircraft_code
),
OptimizationAnalysis AS (
    SELECT
        ad_dep.city->>'ru' AS "Город вылета",
        ad_arr.city->>'ru' AS "Город прилета",
        rd.avg_total_pax,
        rd.avg_business_pax,
        act_stats.model_name AS "Факт. модель",
        act_stats.total_capacity AS "Факт. емкость",
        act_stats.business_capacity AS "Факт. бизнес-емкость",
        ROUND((rd.avg_total_pax / act_stats.total_capacity) * 100, 1) AS "Факт. загрузка, %",
        (SELECT rec_stats.model_name
         FROM AircraftStats rec_stats
         WHERE rec_stats.total_capacity >= (rd.avg_total_pax * 1.15)
           AND rec_stats.business_capacity >= rd.avg_business_pax
         ORDER BY rec_stats.total_capacity ASC 
         LIMIT 1
        ) AS "Реком. модель",
        (SELECT rec_stats.total_capacity
         FROM AircraftStats rec_stats
         WHERE rec_stats.total_capacity >= (rd.avg_total_pax * 1.15)
           AND rec_stats.business_capacity >= rd.avg_business_pax
         ORDER BY rec_stats.total_capacity ASC
         LIMIT 1
        ) AS "Реком. емкость"
        
    FROM RouteDemand rd
    JOIN AircraftStats act_stats ON rd.actual_aircraft_code = act_stats.aircraft_code
    JOIN airports_data ad_dep ON rd.departure_airport = ad_dep.airport_code
    JOIN airports_data ad_arr ON rd.arrival_airport = ad_arr.airport_code
)
SELECT 
    oa."Город вылета",
    oa."Город прилета",
    oa."Факт. модель",
    oa."Факт. емкость",
    ROUND(oa.avg_total_pax) AS "Средний спрос",
    oa."Факт. загрузка, %",
    oa."Реком. модель",
    oa."Реком. емкость",
    (oa."Факт. емкость" - oa."Реком. емкость") AS "Избыточная емкость (потенциал)"
FROM OptimizationAnalysis oa
WHERE 
    oa."Реком. модель" IS NOT NULL
    AND oa."Факт. модель" <> oa."Реком. модель"
    AND (oa."Факт. загрузка, %" < 60 OR oa."Факт. загрузка, %" > 95) 
ORDER BY 
    "Избыточная емкость (потенциал)" DESC;
```
##Комментарии:
сначала рассчитываю тех. хар-ки (вместимость общая и бизнес-класса) для каждой модели самолета
Далее анализирую данные: считаем средний фактический спрос (сколько пассажиров, в т.ч. бизнесом, летало) на каждом конкретном маршруте.
После соединяю факт. спрос (RouteDemand) с факт. вместимостью (AircraftStats) и считаю % загрузки. Здесь же подбираю "Рекомендуемую модель" — самый маленький самолет, который вместил бы средний спрос (с запасиком) и всех бизнес-пассажиров.
И показываю только те рейсы, где факт. самолет не равен рекомендуемому, и где загрузка аномально низкая (самолет слишком большой) или высокая (самолет слишком маленький)
#3.1
```SQL
SELECT 
    CASE EXTRACT(ISODOW FROM book_date)
        WHEN 1 THEN '1. Понедельник'
        WHEN 2 THEN '2. Вторник'
        WHEN 3 THEN '3. Среда'
        WHEN 4 THEN '4. Четверг'
        WHEN 5 THEN '5. Пятница'
        WHEN 6 THEN '6. Суббота'
        WHEN 7 THEN '7. Воскресенье'
    END AS "День недели",
    EXTRACT(HOUR FROM book_date) AS "Час суток (UTC)",
    COUNT(*) AS "Кол-во бронирований"
FROM bookings
GROUP BY "День недели", "Час суток (UTC)"
ORDER BY "Кол-во бронирований" DESC
LIMIT 20; 
```
##Комментарии:
Извлекаю день недели (ISODOW) и час (HOUR) из даты бронирования. 
CASE использую для, чтобы превратить цифры в дни недели
Групптоую по  двум параметрам и считаем кол-во бронирований.
#3.2
```SQL
WITH PriceAnalysis AS (
    SELECT 
        tf.fare_conditions AS "Класс",
        EXTRACT(DAY FROM (f.scheduled_departure::date - b.book_date::date)) AS days_in_advance,
        tf.amount,
        AVG(tf.amount) OVER(
            PARTITION BY f.departure_airport, f.arrival_airport, tf.fare_conditions
        ) AS avg_route_price
    FROM bookings b
    JOIN tickets t ON b.book_ref = t.book_ref
    JOIN ticket_flights tf ON t.ticket_no = tf.ticket_no
    JOIN flights f ON tf.flight_id = f.flight_id
    WHERE f.status <> 'Cancelled'
      AND f.scheduled_departure::date > b.book_date::date
      AND tf.amount > 0
      AND tf.amount < 500000 
)
SELECT 
    "Класс",
    days_in_advance AS "Дней до вылета",
    COUNT(*) AS "Кол-во билетов (выборка)",
    ROUND(AVG(amount / avg_route_price), 3) AS "Индекс цены (отношение к средней)"
FROM PriceAnalysis
WHERE days_in_advance BETWEEN 1 AND 90 
GROUP BY "Класс", days_in_advance
HAVING COUNT(*) > 100 
ORDER BY 
    "Класс" ASC, 
    "Индекс цены (отношение к средней)" ASC;
```
##Комментарии:
Вычисляю , за сколько дней до вылета был куплен билет (days_in_advance).С помощью ф-ции AVG(...) OVER(PARTITION BY ...) нормализую цену. Для каждого билета мы находим ср. цену на его маршруте и в его классе, чтобы корректно сравнивать дешевые и дорогие направления.
SELECT ... FROM PriceAnalysis ... Групптрую билеты по классу и кол-во дней до вылета.
Рассчитываю "Индекс цены" (AVG(amount / avg_route_price)). 
Если инд < 1.0, то в этот день билеты покупались выгоднее среднего по рынку.
Сорт по этому индексу показывает самые выгодные "окна" для покупки.
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
##Комментарии:
Сначала рассчитываю расстояние для кажд рейса (по ф.Гаверсинуса). Формула использует координаты (широту и долготу) аэропортов вылета и прилета.
WITH PricePerKm AS () соединяю ticket_flights (где есть цена билета amount) с FlightDistances (где есть distance_km). Рассчитываю стоимость одного километра полета (price_per_km) для каждого отдельного билета.
AVG(price_per_km) — ф-ция показыает выгоду для пассажира (средняя цена за 1 км).
SUM(amount) / SUM(distance_km) — показывает выгоду для авиакомпании (общая выручка с 1 км полета этого борта).