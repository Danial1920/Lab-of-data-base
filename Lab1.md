#1.2
```SQL
SELECT *
FROM flights
WHERE arrival_airport = 'SVO'
  AND actual_arrival::date = '2017-07-22'
  AND actual_arrival::time BETWEEN '16:00' AND '19:00';
```
## Пояснение
Использую оператор `WHERE` для фильтрации по нескольким критериям:

1.  Аэропорт прибытия — 'SVO'.
2.  Дата: Использую `::date` для извлечения только даты из actual_arrival и сравниваю ее с '2017-07-22'.
3.  Время: Применяю `::time` для извлечения времени, которое затем сравнивается с помощью оператора `BETWEEN`.


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

Использую оператор `UPDATE` для таблицы flights.
* В блоке `SET` указываю, что обновляю. Прибавляю 30 минут к плановому вылету и плановому прилету, чтобы сохранить длительность полета.
* В блоке `WHERE` отбираю рейсы, вылетающие в ночь между 11 и 12 сентября, то есть с 23:00 11-го до 06:00 12-го.



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
Выполняю анализ задержанных рейсов:
1.  Фильтрация: Отбираю только нужные строки: status = 'Delayed' и месяц равен 6 (июнь).
2.  Группировка: Группирую по дате (без времени) вылета.
3.  Подсчет: Считаю количество задержанных рейсов (`COUNT(*)`) за каждую дату.
4.  Сортировка: Сортирую результат для лучшего понимания.