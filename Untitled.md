Нужно реализовать подготовку статистических признаков для ML проекта Antifraud, чтобы передавать расширенный список вместе с реквестом при запросе к сервису инференса.

Рассматриваются следующие сущности:
* `user_id`
* `device_ip` (ipv4 ИЛИ ipv6)
* `host` (site_domain ИЛИ app_bundle)



### Логика
##### Формирование дополнительных признаков
Для некоторых статистик нужно будет сперва ля ля
* `avg_visit_duration`: поле из `rtb.yandex_preclick`
* `page_depth`: поле из `rtb.yandex_preclick`
* `avg_duration_per_page`: 
	```python
	w = Window.partitionBy("user_id").orderBy(F.to_timestamp("event_time", "yyyy-MM-dd HH:mm:ss"))
	F.lag(F.col("avg_visit_duration") / F.col("page_depth"), 1).over(w)
	```

##### Предагрегация
В процессе этого шага формируются предварительные срезы данных для каждой сущности, что используются для вычисления конечных статистик.

1. Данные группируются по каждой сущности (`user_id`, `host`, `device_ip`) и дате
2. Для каждой группы считаются **дневные агрегаты**:
	1. `daily_events`: общее количество событий. Расчитывается как количество записей, функцией count
	2. `daily_min_ts`: минимальный timestamp.
	3. `daily_max_ts`: максимальный timestamp.
	4. Для метрик (`time_to_show`, `time_show_to_view`, `ctr`, `viewability`, `vtr`, `imp_mldata_user_ctr`, `imp_mldata_user_vr`, `imp_mldata_user_vtr`):
		1. `daily_sum_{metric}`: сумма значений метрики.
		2. `daily_count_{metric}`: количество НЕ-NULL значений метрики.
		3. `daily_sum_sq_{metric}`: сумма квадратов значений метрики.
	5. Для метрик (`ctr`, `viewability`, `vtr`, `imp_mldata_user_ctr`, `imp_mldata_user_vr`, `imp_mldata_user_vtr`):
		1. `daily_distincts_{metric}`:  количество уникальных значений
	6. Для метрик:
		1. `daily_sum_{metric}`: сумма значений метрики.
	7. Для метрик (`is_night`, `is_morning`, `is_day`, `is_evening`):
		1. `daily_max_{metric}`: максимальное значение