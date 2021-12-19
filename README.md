# Задание №12: Секционирование таблицы

### Секционировать большую таблицу из демо базы flights
**Выполнение:** Для выполнения задания был загружен дамп демо-базы *demo-small-en.zip*  
После развертывания дампа таблица flights была секционирована следующим образом:
```
-- Создаем таблицу с возможностью секционирования по дате отправления рейсов
create table flights_p (
	flight_id serial,
	flight_no bpchar(6),
	scheduled_departure timestamptz,
	scheduled_arrival timestamptz,
	departure_airport bpchar(3),
	arrival_airport bpchar(3),
	status varchar(20),
	aircraft_code bpchar(3),
	actual_departure timestamptz,
	actual_arrival timestamptz
)
partition by range (scheduled_departure);

-- Выполняем код, создающий секции по месяцам и годам
do
$$
declare
	r record;
    y_m_date_start varchar;
    y_m_date_end varchar;
begin
	for r in
		select extract(year from scheduled_departure) as r_year,
				extract(month from scheduled_departure) as r_month
			from flights
		group by 1, 2
		order by 1, 2
	loop
		y_m_date_start = r.r_year || '-' || r.r_month;
		y_m_date_end = r.r_year || '-' || r.r_month + 1;
		execute format('create table flights_part_%s_%s partition of flights_p for values from (''%s-01'') to (''%s-01'')', r.r_year, r.r_month, y_m_date_start, y_m_date_end);
	end loop;
end
$$

-- Переносим в новую таблицу значения из оригинальной таблицы flights
insert into flights_p (select * from flights);

-- Смотрим explain простого селекта по колонке, по которой осуществлялось секционирование,
-- чтобы убедиться, что все отработало правильно
explain
select * from flights_p where scheduled_departure between '2017-07-01' and '2017-07-31';
                                                                               QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on flights_part_2017_7 flights_p  (cost=0.00..238.35 rows=8156 width=63)
   Filter: ((scheduled_departure >= '2017-07-01 00:00:00+00'::timestamp with time zone) AND (scheduled_departure <= '2017-07-31 00:00:00+00'::timestamp with time zone))
(2 rows)

-- Как можно увидеть в результате, explain, для данного запроса использовалась только секция flights_part_2017_7
```
