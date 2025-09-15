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
	Отношение `avg_visit_duration` к `page_depth`, но для "прошлой" строки
	```python
	w = Window.partitionBy("user_id").orderBy(F.to_timestamp("event_time", "yyyy-MM-dd HH:mm:ss"))
	F.lag(F.col("avg_visit_duration") / F.col("page_depth"), 1).over(w)
	```
* `time_to_show`: 
	
	```python
	F.col("time_show") = F.expr(f"""  
	    aggregate(        zip_with(event_type, event_time, (t, ts) -> IF(t IN ({','.join(["3", "5"])}), ts, null)),  
        cast(null as timestamp),        (acc, x) -> IF(x IS NOT NULL AND (acc IS NULL OR x < acc), x, acc)    )""").alias("time_show")
	
	F.col("time_to_show") = F.when(  
	    F.col("time_show").isNotNull(),  
	    F.unix_timestamp("time_show") - F.unix_timestamp(F.to_timestamp("event_time", "yyyy-MM-dd HH:mm:ss")) 
	)
	```
	* `time_show_to_view`:
	```python
	F.col("time_view") = F.expr(f"""  
	    aggregate(        zip_with(event_type, event_time, (t, ts) -> IF(t IN ({','.join(["172", "173", "10"])}), ts, null)),  
        cast(null as timestamp),        (acc, x) -> IF(x IS NOT NULL AND (acc IS NULL OR x < acc), x, acc)    )""").alias("time_view")
        
	"time_show_to_view": F.when(  
	    F.col("time_view").isNotNull() & F.col("time_show").isNotNull(),  
	    F.unix_timestamp("time_view") - F.unix_timestamp("time_show")  
	),
	```
	* Для метрик (`ctr`, `viewability`, `vtr`, `imp_mldata_user_ctr`, `imp_mldata_user_vr`, `imp_mldata_user_vtr`) создаем:
		* `{metric}_is_zero`: 1 если значение равно 0, иначе 0. Пропуски остаются пропусками.
		* `{metric}_is_high`: 1 если значение > 5, иначе 0. Пропуски остаются пропусками.

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
		1. `daily_distincts_{metric}`:  массив уникальных значений
	6. Для метрик (`_is_zero`, `_is_high`):
		1. `daily_sum_{metric}`: сумма значений метрики.
	7. Для метрик (`is_night`, `is_morning`, `is_day`, `is_evening`):
		1. `daily_max_{metric}`: максимальное значение

##### Статистика

* `{entity}_avg_{window-length}d`:
	Отношение суммы сумм `daily_sum_` к сумме `daily_count_` за период.
```python
sum(f"daily_sum_{entity}").over(w) / sum(f"daily_count_{entity}").over(w)
```
* `{entity}_stddev_{window-length}d`:
	Корень из разницы среднего квадрата и квадрата среднего
```python
sqrt(  
	(sum(f"daily_sum_sq_{entity}").over(w) / sum(f"daily_count_{entity}").over(w)) - \  
	(pow(sum(f"daily_sum_{entity}").over(w) /sum(f"daily_count_{entity}").over(w), 2))  
)
  ```
  * `{entity}_rate_{window-length}d`:
		Доля значений (используется только для `_is_` метрик)
  ```python
sum(f"daily_sum_{entity}").over(w) / sum("daily_events").over(w)
  ```
  * `{entity}_active_periods_{window-length}d`:
  ```python
sum("daily_max_is_night").over(w) +  
sum("daily_max_is_morning").over(w) +  
sum("daily_max_is_day").over(w) +  
sum("daily_max_is_evening").over(w)  
  ```
  * `{entity}_distinct_{window-length}d`:
	  Количество уникальных значений
  ```python
  size(array_distinct(flatten(collect_set(f"daily_distincts_{entity}").over(w))))
  ```

##### 