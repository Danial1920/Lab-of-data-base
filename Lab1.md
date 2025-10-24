#1.1 
SELECT *
FROM flights
WHERE departure_airport = 'UFA'
  AND (actual_departure - scheduled_departure) > INTERVAL '2 hours';
##Пояснение
Выбираю все столбцы с помощью select из flights. В where задаю два условия:
Аэропорт вылета должен быть 'UFA'
Разница между фактическим (actual_departure) и плановым( scheduled_departure) временем вылета должна быть больше 2 часов

#1.2
SELECT *
FROM flights
WHERE arrival_airport = 'SVO'
  AND actual_arrival::date = '2017-07-22'
  AND actual_arrival::time BETWEEN '16:00' AND '19:00';
##Пояснение
Снова использую where для фильтрации:
Аэропорт прибытия  — 'SVO'
Испольщую ::date для получения только даты из actual_arrival и сравниваю ее с '2017-07-22'
Также ::time извлекает время, которое мы сравниваем с помощью between


#1.3
SELECT *
FROM flights
WHERE aircraft_code = 'SU9'
  AND scheduled_departure::time BETWEEN '04:00' AND '12:00';
##Пояснение
Фильтрую по коду самолета 'SU9'. Для определения "утренних" рейсов, беру плановое время вылета (scheduled_departure), извлекаю из него время (::time) и проверяю, попадает ли оно в диапазон 04:00 и 12:00

#2.1
UPDATE flights
SET
    scheduled_departure = scheduled_departure + INTERVAL '30 minutes',
    scheduled_arrival = scheduled_arrival + INTERVAL '30 minutes'
WHERE
    scheduled_departure >= '2017-09-11 23:00:00'
    AND scheduled_departure < '2017-09-12 06:00:00';
##Пояснение
Использую update для flights.
В блоке SET указываю , что обновляю.Далее прибавляю 30 минут к плановому вылету и плановому прилету, чтобы сохранить длительность полета.
В where отбираю рейсы, вылетающие в ночь между 11 и 12 сентября, то есть с 23:00 11-го до 06:00 12-го

#2.2
UPDATE ticket_flights
SET amount = amount * 1.10
WHERE fare_conditions = 'Business'
  AND amount < 10000;
##Пояснение
Обновляю таблицу ticket_flights, где хранится стоимость
Далее увеличиваю число на 10%
А where фильтрует строки: класс обслуживания  'Business' и стоимость меньше 10000

#2.3
UPDATE ticket_flights AS tf
SET amount = tf.amount + 1000
FROM flights AS f
WHERE tf.flight_id = f.flight_id
  AND (
    (f.scheduled_departure::time BETWEEN '07:00' AND '10:00')
    OR (f.scheduled_departure::time BETWEEN '17:00' AND '20:00')
  );
##Пояснение
Берем данные из двух таблиц с помощью update: 
ticket_flights (чтобы обновить цену) и flights (чтобы проверить время)
where tf.flight_id = f.flight_id — это условие объединения двух таблиц
Далее идут условия фильтрации по времени из таблицы flights (f.scheduled_departure), юзаю or для объединения двух пиковых интервалов.

#2.4
UPDATE flights
SET status = 'Cancelled'
WHERE status = 'Delayed'
  AND departure_airport = 'SVO';
##Пояснение
меняю status на 'Cancelled' для всех строк, где текущий status равен 'Delayed' и аэропорт вылета 'SVO'.

#3.1
SELECT
    departure_airport,
    COUNT(flight_id) AS flight_count,
    AVG(actual_arrival - actual_departure) AS avg_duration
FROM flights
WHERE status = 'Arrived' -- Считаем длительность только для завершенных рейсов
GROUP BY departure_airport;
##Пояснение
GROUP BY departure_airport — схлопывает все строки с одинаковым departure_airport в одну
COUNT(flight_id) — ф-ия ,которая считает кол-во рейсов в каждой группе
AVG(actual_arrival - actual_departure) — ф-ия, которая считает среднюю продолжительность полета в группе.
Добаил WHERE status = 'Arrived'

#3.2
SELECT
    EXTRACT(HOUR FROM scheduled_departure) AS departure_hour,
    COUNT(*) AS flight_count
FROM flights
WHERE departure_airport = 'DME'
GROUP BY departure_hour
ORDER BY departure_hour;
##Пояснение
Сначала фильтрую все рейсы из 'DME'.
EXTRACT(HOUR FROM scheduled_departure) — ф-ия , извлекающая час из времени вылета.
GROUP BY departure_hour — группирую по этому часу.
COUNT(*) — подсчитваю, сколько рейсов попало в каждый час.
#3.3
SELECT
    timezone,
    MAX(latitude) AS max_latitude
FROM airports
GROUP BY timezone;
##Пояснение
Группирую  airports по timezone.
Самый "северный" — тот, у которого максимальная широта
MAX(latitude) — ф-ия , которая находит макс. значение широты в каждой группе
#3.4
SELECT
    scheduled_departure::date AS flight_date,
    COUNT(*) AS delayed_count
FROM flights
WHERE status = 'Delayed'
  AND EXTRACT(MONTH FROM scheduled_departure) = 6
GROUP BY flight_date
ORDER BY flight_date;
##Пояснение
Отбираю только нужные строки: status = 'Delayed' и месяц равен 6 (июнь)
Далее группирую по дате (без времени)
Считаю кол-во задержанных рейсов за эту дату
И сортирую для лучшего понимания
#4.1
SELECT *
FROM tickets
WHERE passenger_name LIKE 'IVANOV %'
  AND (contact_data ->> 'email') IS NOT NULL; 
##Пояснение
Ищу в таблице tickets фамилию "Иванов"  с помощью LIKE 'IVANOV %'
 Знак % означает "любые символы после".
Поле contact_data в этой базе имеет тип jsonb. Оператор ->> извлекает значение по ключу 'email' в виде текста. В конйе проверяю, что это значение не равно NULL.