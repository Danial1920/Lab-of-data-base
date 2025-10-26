#1.2
```SQL
SELECT *
FROM flights
WHERE arrival_airport = 'SVO'
  AND actual_arrival::date = '2017-07-22'
  AND actual_arrival::time BETWEEN '16:00' AND '19:00';
```
## Пояснение
Снова использую where для фильтрации:
Аэропорт прибытия  — 'SVO'
Испольщую ::date для получения только даты из actual_arrival и сравниваю ее с '2017-07-22'
Также ::time извлекает время, которое мы сравниваем с помощью between

#2.1
```SQL
UPDATE flights
SET
    scheduled_departure = scheduled_departure + INTERVAL '30 minutes',
    scheduled_arrival = scheduled_arrival + INTERVAL '30 minutes'
WHERE
    scheduled_departure >= '2017-09-11 23:00:00'
    AND scheduled_departure < '2017-09-12 06:00:00';
```
## Пояснение
Использую update для flights.
В блоке SET указываю , что обновляю.Далее прибавляю 30 минут к плановому вылету и плановому прилету, чтобы сохранить длительность полета.
В where отбираю рейсы, вылетающие в ночь между 11 и 12 сентября, то есть с 23:00 11-го до 06:00 12-го

#3.4
```SQL
SELECT
    scheduled_departure::date AS flight_date,
    COUNT(*) AS delayed_count
FROM flights
WHERE status = 'Delayed'
  AND EXTRACT(MONTH FROM scheduled_departure) = 6
GROUP BY flight_date
ORDER BY flight_date;
```
## Пояснение
Отбираю только нужные строки: status = 'Delayed' и месяц равен 6 (июнь)
Далее группирую по дате (без времени)
Считаю кол-во задержанных рейсов за эту дату
И сортирую для лучшего понимания
