Нужно реализовать подготовку статистических признаков для ML проекта Antifraud, чтобы передавать расширенный список вместе с реквестом при запросе к сервису инференса.

Рассматриваются следующие сущности:
* `user_id`
* `device_ip` (ipv4 ИЛИ ipv6)
* `host` (site_domain ИЛИ app_bundle)


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
* Для метрик (`ctr`, `viewability`, `vtr`, `imp_mldata_user_ctr`, `imp_mldata_user_vr`, `imp_mldata_user_vtr`) создаются:
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
	  Количество уникальных значений, использует промежуточный признак `daily_distincts_`
* `{entity}_num_periods_{window-length}d`:
	Количество **отдельных, непрерывных периодов** активности (сессий) у сущности за N дней. Период — это один или несколько дней активности подряд. Разрыв в 2+ дня начинает новый период.
	**Логика расчета:**
	1. Получить **список дат**, за которые у сущности есть предагрегация в рамках окна N дней.
	2. Отсортировать даты (от новой к старой).
	3. Инициализировать `num_periods = 1` (самый первый день в окне — это уже начало одного периода).
	4. Идти по списку дат и сравнивать `текущую_дату` с `предыдущей_датой`.
	5. Если `предыдущая_дата - текущая_дата > 1` (т.е. разрыв больше 1 дня), то `num_periods++`.
	6. Вернуть `num_periods`.


### 

##### Итоговый список признаков

|   |   |   |
|---|---|---|
|Имя итогового признака|Промежуточный признак|Финальный расчет|
|avg_visit_duration_avg_7d|avg_visit_duration (для user_id)|_avg_7d от промежуточного признака|
|avg_visit_duration_stddev_7d|avg_visit_duration (для user_id)|_stddev_7d от промежуточного признака|
|page_depth_avg_7d|page_depth (для user_id)|_avg_7d от промежуточного признака|
|page_depth_stddev_7d|page_depth (для user_id)|_stddev_7d от промежуточного признака|
|avg_duration_per_page_avg_7d|avg_duration_per_page (для user_id)|_avg_7d от промежуточного признака|
|avg_duration_per_page_stddev_7d|avg_duration_per_page (для user_id)|_stddev_7d от промежуточного признака|
|time_to_show_avg_7d|time_to_show (для user_id)|_avg_7d от промежуточного признака|
|time_to_show_stddev_7d|time_to_show (для user_id)|_stddev_7d от промежуточного признака|
|time_show_to_view_avg_7d|time_show_to_view (для user_id)|_avg_7d от промежуточного признака|
|time_show_to_view_stddev_7d|time_show_to_view (для user_id)|_stddev_7d от промежуточного признака|
|imp_mldata_user_ctr_avg_7d|imp_mldata_user_ctr (для user_id)|_avg_7d от промежуточного признака|
|imp_mldata_user_ctr_stddev_7d|imp_mldata_user_ctr (для user_id)|_stddev_7d от промежуточного признака|
|imp_mldata_user_vr_avg_7d|imp_mldata_user_vr (для user_id)|_avg_7d от промежуточного признака|
|imp_mldata_user_vr_stddev_7d|imp_mldata_user_vr (для user_id)|_stddev_7d от промежуточного признака|
|imp_mldata_user_vtr_avg_7d|imp_mldata_user_vtr (для user_id)|_avg_7d от промежуточного признака|
|imp_mldata_user_vtr_stddev_7d|imp_mldata_user_vtr (для user_id)|_stddev_7d от промежуточного признака|
|avg_visit_duration_avg_28d|avg_visit_duration (для user_id)|_avg_28d от промежуточного признака|
|avg_visit_duration_stddev_28d|avg_visit_duration (для user_id)|_stddev_28d от промежуточного признака|
|page_depth_avg_28d|page_depth (для user_id)|_avg_28d от промежуточного признака|
|page_depth_stddev_28d|page_depth (для user_id)|_stddev_28d от промежуточного признака|
|avg_duration_per_page_avg_28d|avg_duration_per_page (для user_id)|_avg_28d от промежуточного признака|
|avg_duration_per_page_stddev_28d|avg_duration_per_page (для user_id)|_stddev_28d от промежуточного признака|
|time_to_show_avg_28d|time_to_show (для user_id)|_avg_28d от промежуточного признака|
|time_to_show_stddev_28d|time_to_show (для user_id)|_stddev_28d от промежуточного признака|
|time_show_to_view_avg_28d|time_show_to_view (для user_id)|_avg_28d от промежуточного признака|
|time_show_to_view_stddev_28d|time_show_to_view (для user_id)|_stddev_28d от промежуточного признака|
|imp_mldata_user_ctr_avg_28d|imp_mldata_user_ctr (для user_id)|_avg_28d от промежуточного признака|
|imp_mldata_user_ctr_stddev_28d|imp_mldata_user_ctr (для user_id)|_stddev_28d от промежуточного признака|
|imp_mldata_user_vr_avg_28d|imp_mldata_user_vr (для user_id)|_avg_28d от промежуточного признака|
|imp_mldata_user_vr_stddev_28d|imp_mldata_user_vr (для user_id)|_stddev_28d от промежуточного признака|
|imp_mldata_user_vtr_avg_28d|imp_mldata_user_vtr (для user_id)|_avg_28d от промежуточного признака|
|imp_mldata_user_vtr_stddev_28d|imp_mldata_user_vtr (для user_id)|_stddev_28d от промежуточного признака|
|avg_visit_duration_avg_91d|avg_visit_duration (для user_id)|_avg_91d от промежуточного признака|
|avg_visit_duration_stddev_91d|avg_visit_duration (для user_id)|_stddev_91d от промежуточного признака|
|page_depth_avg_91d|page_depth (для user_id)|_avg_91d от промежуточного признака|
|page_depth_stddev_91d|page_depth (для user_id)|_stddev_91d от промежуточного признака|
|avg_duration_per_page_avg_91d|avg_duration_per_page (для user_id)|_avg_91d от промежуточного признака|
|avg_duration_per_page_stddev_91d|avg_duration_per_page (для user_id)|_stddev_91d от промежуточного признака|
|time_to_show_avg_91d|time_to_show (для user_id)|_avg_91d от промежуточного признака|
|time_to_show_stddev_91d|time_to_show (для user_id)|_stddev_91d от промежуточного признака|
|time_show_to_view_avg_91d|time_show_to_view (для user_id)|_avg_91d от промежуточного признака|
|time_show_to_view_stddev_91d|time_show_to_view (для user_id)|_stddev_91d от промежуточного признака|
|imp_mldata_user_ctr_avg_91d|imp_mldata_user_ctr (для user_id)|_avg_91d от промежуточного признака|
|imp_mldata_user_ctr_stddev_91d|imp_mldata_user_ctr (для user_id)|_stddev_91d от промежуточного признака|
|imp_mldata_user_vr_avg_91d|imp_mldata_user_vr (для user_id)|_avg_91d от промежуточного признака|
|imp_mldata_user_vr_stddev_91d|imp_mldata_user_vr (для user_id)|_stddev_91d от промежуточного признака|
|imp_mldata_user_vtr_avg_91d|imp_mldata_user_vtr (для user_id)|_avg_91d от промежуточного признака|
|imp_mldata_user_vtr_stddev_91d|imp_mldata_user_vtr (для user_id)|_stddev_91d от промежуточного признака|
|imp_mldata_user_ctr_distinct_7d|imp_mldata_user_ctr (для user_id)|_distinct_7d от промежуточного признака|
|imp_mldata_user_vr_distinct_7d|imp_mldata_user_vr (для user_id)|_distinct_7d от промежуточного признака|
|imp_mldata_user_vtr_distinct_7d|imp_mldata_user_vtr (для user_id)|_distinct_7d от промежуточного признака|
|imp_mldata_user_ctr_distinct_28d|imp_mldata_user_ctr (для user_id)|_distinct_28d от промежуточного признака|
|imp_mldata_user_vr_distinct_28d|imp_mldata_user_vr (для user_id)|_distinct_28d от промежуточного признака|
|imp_mldata_user_vtr_distinct_28d|imp_mldata_user_vtr (для user_id)|_distinct_28d от промежуточного признака|
|imp_mldata_user_ctr_distinct_91d|imp_mldata_user_ctr (для user_id)|_distinct_91d от промежуточного признака|
|imp_mldata_user_vr_distinct_91d|imp_mldata_user_vr (для user_id)|_distinct_91d от промежуточного признака|
|imp_mldata_user_vtr_distinct_91d|imp_mldata_user_vtr (для user_id)|_distinct_91d от промежуточного признака|
|imp_mldata_user_ctr_is_zero_rate_7d|imp_mldata_user_ctr_is_zero (для user_id)|_rate_7d от промежуточного признака|
|imp_mldata_user_vr_is_zero_rate_7d|imp_mldata_user_vr_is_zero (для user_id)|_rate_7d от промежуточного признака|
|imp_mldata_user_vtr_is_zero_rate_7d|imp_mldata_user_vtr_is_zero (для user_id)|_rate_7d от промежуточного признака|
|imp_mldata_user_ctr_is_high_rate_7d|imp_mldata_user_ctr_is_high (для user_id)|_rate_7d от промежуточного признака|
|imp_mldata_user_vr_is_high_rate_7d|imp_mldata_user_vr_is_high (для user_id)|_rate_7d от промежуточного признака|
|imp_mldata_user_vtr_is_high_rate_7d|imp_mldata_user_vtr_is_high (для user_id)|_rate_7d от промежуточного признака|
|imp_mldata_user_ctr_is_zero_rate_28d|imp_mldata_user_ctr_is_zero (для user_id)|_rate_28d от промежуточного признака|
|imp_mldata_user_vr_is_zero_rate_28d|imp_mldata_user_vr_is_zero (для user_id)|_rate_28d от промежуточного признака|
|imp_mldata_user_vtr_is_zero_rate_28d|imp_mldata_user_vtr_is_zero (для user_id)|_rate_28d от промежуточного признака|
|imp_mldata_user_ctr_is_high_rate_28d|imp_mldata_user_ctr_is_high (для user_id)|_rate_28d от промежуточного признака|
|imp_mldata_user_vr_is_high_rate_28d|imp_mldata_user_vr_is_high (для user_id)|_rate_28d от промежуточного признака|
|imp_mldata_user_vtr_is_high_rate_28d|imp_mldata_user_vtr_is_high (для user_id)|_rate_28d от промежуточного признака|
|imp_mldata_user_ctr_is_zero_rate_91d|imp_mldata_user_ctr_is_zero (для user_id)|_rate_91d от промежуточного признака|
|imp_mldata_user_vr_is_zero_rate_91d|imp_mldata_user_vr_is_zero (для user_id)|_rate_91d от промежуточного признака|
|imp_mldata_user_vtr_is_zero_rate_91d|imp_mldata_user_vtr_is_zero (для user_id)|_rate_91d от промежуточного признака|
|imp_mldata_user_ctr_is_high_rate_91d|imp_mldata_user_ctr_is_high (для user_id)|_rate_91d от промежуточного признака|
|imp_mldata_user_vr_is_high_rate_91d|imp_mldata_user_vr_is_high (для user_id)|_rate_91d от промежуточного признака|
|imp_mldata_user_vtr_is_high_rate_91d|imp_mldata_user_vtr_is_high (для user_id)|_rate_91d от промежуточного признака|
|user_id_events_sum_7d|daily_events (для user_id)|_sum_7d от промежуточного признака|
|user_id_events_avg_per_day_7d|daily_events (для user_id)|_avg_per_day_7d от промежуточного признака|
|user_id_events_sum_28d|daily_events (для user_id)|_sum_28d от промежуточного признака|
|user_id_events_avg_per_day_28d|daily_events (для user_id)|_avg_per_day_28d от промежуточного признака|
|user_id_events_sum_91d|daily_events (для user_id)|_sum_91d от промежуточного признака|
|user_id_events_avg_per_day_91d|daily_events (для user_id)|_avg_per_day_91d от промежуточного признака|
|user_id_num_periods_7d|(периоды user_id)|_num_periods_7d|
|user_id_avg_period_length_7d|(периоды user_id)|_avg_period_length_7d|
|user_id_num_periods_28d|(периоды user_id)|_num_periods_28d|
|user_id_avg_period_length_28d|(периоды user_id)|_avg_period_length_28d|
|user_id_num_periods_91d|(периоды user_id)|_num_periods_91d|
|user_id_avg_period_length_91d|(периоды user_id)|_avg_period_length_91d|