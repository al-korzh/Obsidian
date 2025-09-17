Конечно, вот план реализации.

Он разбит на 5 логических этапов: от подготовки до полной автоматизации.

---

### **Этап 1: Анализ и подготовка**

**Цель:** Определить точную структуру таблицы и параметры для запроса.

1. **Определить финальную схему.** На основе нашего предыдущего анализа составьте окончательный список столбцов для предагрегированной таблицы (например, `user_id, host, device_ip, date, daily_events, daily_sum_ctr, ...`) и их типы данных в ClickHouse.
    
2. **Рассчитать оптимальный размер "соли".**
    
    - Выполните аналитический запрос к сырым данным, чтобы найти `max(count)` для одного ключа (`user_id, host, device_ip`).
        
    - Определите комфортный "размер чанка" (например, 1 млн событий).
        
    - Рассчитайте `размер_соли = max_count / chunk_size`. Запомните это число.
        
3. **Создать целевую таблицу.** Напишите и выполните `CREATE TABLE` для вашей новой предагрегированной таблицы.
    
    - **Движок:** `ENGINE = MergeTree()` — хороший выбор по умолчанию.
        
    - **Партиционирование:** `PARTITION BY toYYYYMM(date)` — очень рекомендуется для управления данными.
        
    - **Ключ сортировки:** `ORDER BY (user_id, host, device_ip, date)` — для быстрой фильтрации.
        

---

### **Этап 2: Разработка и тестирование SQL-запроса**

**Цель:** Создать один работающий и проверенный SQL-запрос, который делает всю работу.

1. **Написать SQL-запрос.** Используя шаблон из нашего предыдущего ответа, напишите полный запрос с двумя уровнями `SELECT` (внутренний с `salt` и внешний для финальной группировки).
    
    - Подставьте в него рассчитанный на **Этапе 1** `размер_соли`.
        
    - Включите все необходимые агрегаты (`sum`, `count`, `sum_sq`, `groupUniqArray` и т.д.).
        
2. **Протестировать на малом объеме.** Запустите этот запрос с `INSERT INTO ...` для **одного дня** (`WHERE date = 'YYYY-MM-DD'`). Убедитесь, что он выполняется без ошибок и данные появляются в целевой таблице. Проверьте несколько строк на адекватность.
    

---

### **Этап 3: Первичное наполнение (Backfill)**

**Цель:** Заполнить таблицу историческими данными.

1. **Подготовить скрипт для запуска.** Создайте простой скрипт (Python, Bash), который будет выполнять ваш SQL-запрос.
    
2. **Запускать итеративно, по частям!** **Ни в коем случае не запускайте запрос сразу за всю историю.** Это создаст огромную нагрузку на кластер.
    
    - В цикле запускайте ваш скрипт для каждого месяца (или недели) исторического периода.
        
    - Пример логики: `for month in [2024-01, 2024-02, ...]: run_query(month)`.
        
3. **Мониторить кластер.** Во время выполнения следите за нагрузкой на ClickHouse (`system.processes`, `system.merges`).
    

---

### **Этап 4: Автоматизация (Инкрементальные обновления)**

**Цель:** Настроить ежедневное автоматическое пополнение таблицы.

1. **Параметризовать скрипт.** Убедитесь, что ваш скрипт из **Этапа 3** может принимать дату как параметр (например, `--date YYYY-MM-DD`).
    
2. **Настроить планировщик.** Используйте любой планировщик (Airflow, Cron, Jenkins и т.д.), чтобы настроить ежедневный запуск вашего скрипта.
    
    - Команда будет выглядеть примерно так: `python my_etl_script.py --date=$(date -d "yesterday" +%Y-%m-%d)`.
        
    - Запуск должен происходить после того, как все сырые данные за вчерашний день гарантированно загружены.
        

---

### **Этап 5: Валидация**

**Цель:** Убедиться, что данные в новой таблице корректны.

1. **Сравнить с результатами Spark.** Возьмите данные за один и тот же день, сгенерированные старым Spark-процессом и новым на ClickHouse.
    
2. **Проверить показатели:**
    
    - **Количество строк:** `SELECT count() FROM ... WHERE date = X` должно совпадать.
        
    - **Суммы по метрикам:** `SELECT sum(daily_events), avg(daily_sum_ctr) ...` должны быть идентичны (с учетом погрешности `float`).
        
    - **Выборочная проверка:** Сравните значения для 2-3 конкретных ключей (`user_id, host, device_ip`), чтобы убедиться в полной идентичности расчетов.
        

Выполнив этот план, вы получите надежный и значительно более производительный процесс формирования вашей витрины данных. ✅

Почему нет следующих колонок

1. daily_sum_avg_visit_duration

2. daily_count_avg_visit_duration

3. daily_sum_sq_avg_visit_duration

4. daily_sum_avg_duration_per_page

5. daily_count_avg_duration_per_page

6. daily_sum_time_show_to_view

7. daily_sum_sq_time_show_to_view

8. user_id_period_id

9. user_id_period_length

10. user_id_is_first_in_period

11. user_id_unique_period_length

12. host_period_id


import json  
import time  
import uuid  
  
from typing import *  
from math import ceil  
from datetime import datetime  
from py4j.protocol import Py4JJavaError  
from pyspark.storagelevel import StorageLevel  
from pyspark.sql import functions as F, SparkSession, DataFrame, Window  
  
  
class ETL:  
    SALT_SAFETY_MARGIN = 1.5  
    SALT_CHUNK_SIZE = 1_000_000  
  
    def __init__(self,  
                 spark: SparkSession,  
                 clickhouse_config: Dict[str, str]):  
        """Initializes the DataCollector instance.  
  
        Args:            spark (SparkSession): The active Spark session.            clickhouse_config (Dict[str, str]): Connection parameters for                ClickHouse, including driver, url, user, and password.        """        self.spark: SparkSession = spark  
        self.clickhouse_config: Dict[str, str] = clickhouse_config  
  
        self.spark.conf.set("spark.sql.session.timeZone", "Europe/Moscow")  
        self.spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")  
  
        self.fs: Any = spark._jvm.org.apache.hadoop.fs.FileSystem.get(  
            spark._jsc.hadoopConfiguration()  
        )  
  
        log4j = spark._jvm.org.apache.log4j  
        self.logger = log4j.LogManager.getLogger(__name__)  
        self.logger.setLevel(log4j.Level.INFO)  
        self.logger.info("ETL process initialized.")  
  
    def __call__(self,  
                 dates: List[str],  
                 output_path: str,  
                 manifest_path: str,  
                 dry_run: bool = False) -> None:  
        """Runs the ETL process for a given list of dates.  
  
        This is the main entry point for the pipeline. It orchestrates the data        loading, processing, and writing steps, wrapping the execution in a        try/except block to ensure that a manifest entry is always created.  
        Args:            dates (List[str]): A list of dates in 'YYYY-MM-DD' format for which                data needs to be processed.            output_path (str): The root path for saving the final Parquet files.            manifest_path (str): The full path to the log file (manifest).            dry_run (bool): If True, the pipeline runs without writing any data.        """        if not dates:  
            self.logger.warning("Date list for processing is empty. Nothing to do.")  
            return  
  
        batch_id = f"{dates[0]} to {dates[-1]}"  
        self.logger.info(f"START: {batch_id} for {len(dates)} dates.")  
  
        start_time = time.time()  
        try:  
            enriched_raw_df = self._load_and_enrich_raw_data(dates)  
            enriched_raw_df.persist(StorageLevel.MEMORY_AND_DISK)  
  
            user_agg_df = self._process_users(enriched_raw_df)  
            self._write_output(user_agg_df, f"{output_path}/user", dry_run)  
  
            host_agg_df = self._process_hosts(enriched_raw_df)  
            self._write_output(host_agg_df, f"{output_path}/host", dry_run)  
  
            ip_agg_df = self._process_ips(enriched_raw_df)  
            self._write_output(ip_agg_df, f"{output_path}/ip", dry_run)  
  
            enriched_raw_df.unpersist()  
  
            self._write_manifest(  
                status='success',  
                duration_sec=time.time() - start_time,  
                dates=dates,  
                output_path=output_path,  
                manifest_path=manifest_path  
            )  
            self.logger.info(f"FINISH: {batch_id}")  
        except Exception as e:  
            self.logger.error(f"Execution failed for batch {batch_id}: \n{e}")  
            self._write_manifest(  
                status='failed',  
                duration_sec=time.time() - start_time,  
                dates=dates,  
                output_path=output_path,  
                manifest_path=manifest_path,  
                error_msg=str(e)  
            )  
  
    def _process_users(self, df: DataFrame) -> DataFrame:  
        self.logger.info("Aggregating for users...")  
        # ... здесь будет логика агрегации для user с солью ...  
        return df  # Заглушка  
  
    def _process_hosts(self, df: DataFrame) -> DataFrame:  
        self.logger.info("Aggregating for hosts...")  
        # ... здесь будет логика агрегации для host с солью ...  
        return df  # Заглушка  
  
    def _process_ips(self, df: DataFrame) -> DataFrame:  
        self.logger.info("Aggregating for IPs...")  
        # ... здесь будет простая агрегация для ip без соли ...  
        return df  # Заглушка  
  
    def _get_optimal_salt(self, dates: List[str], entity: str) -> int:  
        self.logger.info(f"Calculating optimal salt for '{entity}'...")  
  
        query = self._get_salt_query(dates, entity)  
        try:  
            max_size_df = self._read_clickhouse(query)  
            result_row = max_size_df.first()  
            max_size = result_row.max_group_size if result_row else 0  
        except Exception as e:  
            self.logger.warning(f"WARNING: Failed to calculate salt for {entity}. Using default value 1. Error: {e}")  
            return 1  
  
        if not max_size or max_size < self.SALT_CHUNK_SIZE:  
            self.logger.warning(f"Max group size ({max_size:,}) is small, salt is not required.")  
            return 1  
  
        salt = max(2, ceil((max_size / self.SALT_CHUNK_SIZE) * self.SALT_SAFETY_MARGIN))  
  
        self.logger.info(f"Max group size: {max_size:,}. Calculated salt: {salt}")  
        return salt  
  
    def _write_output(self, df: DataFrame, output_path: str, dry_run: bool) -> None:  
        """Writes the final DataFrame to Parquet with optimized partitioning.  
  
        Calculates the optimal number of output partitions based on the data        size to target ~128MB files. It first writes to a temporary location        to measure the size, then repartitions and writes to the final        destination. The temporary data is cleaned up afterwards.  
        Args:            df (DataFrame): The final DataFrame to be saved.            output_path (str): The root path for saving the final Parquet files.            dry_run (bool): If True, the write operation is skipped.        """        if dry_run:  
            self.logger.warning("Dry run mode: skipping actual write.")  
            count = df.count()  
            self.logger.info(f"Dry run: DataFrame contains {count} rows.")  
            return  
  
        temp_output = f"{output_path}/_temp_{str(uuid.uuid4())}"  
        df.write.mode("overwrite").option("compression", "snappy").parquet(temp_output)  
  
        path_obj = self.spark._jvm.org.apache.hadoop.fs.Path(temp_output)  
        size_bytes = self.fs.getContentSummary(path_obj).getLength()  
        size_gb = size_bytes / (1024 ** 3)  
        # Target file size is ~128MB  
        num_partitions = max(1, ceil(size_gb * 1024 / 128))  
  
        self.logger.info(f"Temp output size: {size_gb:.2f} GB. "                      f"Repartitioning to {num_partitions} partitions.")  
  
        (df.repartition(num_partitions)  
         .write  
         .mode("overwrite")  
         .option("compression", "snappy")  
         .partitionBy("date")  
         .parquet(output_path))  
  
        self.fs.delete(path_obj, True)  
        self.logger.info(f"Finished writing data to {output_path}.")  
  
    def _write_manifest(self, status: str, duration_sec: float, dates: List[str],  
                        output_path: str, manifest_path: str,  
                        error_msg: Optional[str] = None) -> None:  
        """Writes execution results to a JSONL manifest file.  
  
        Appends a JSON object for each processed date to the manifest file.        Implements a file-based lock to prevent race conditions from        concurrent runs.  
        Args:            status (str): The final execution status ('success' or 'failed').            duration_sec (float): The total execution time in seconds.            dates (List[str]): The list of dates that were processed.            output_path (str): The root path where data was written.            manifest_path (str): The full path to the manifest file.            error_msg (Optional[str]): The error message if the execution                failed.  
        Raises:            TimeoutError: If the manifest file lock could not be acquired                within the specified time.            RuntimeError: In case of a Java error while writing to the file.        """        rows = []  
        for _date in dates:  
            row: Dict[str, Any] = {  
                "date": _date,  
                "status": status,  
                "duration_sec": round(duration_sec, 2),  
                "error": error_msg or "",  
                "processed_at": datetime.now().isoformat(),  
                "output_path": f"{output_path}/date={_date}"  
            }  
            rows.append(json.dumps(row))  
  
        if not rows:  
            return  
  
        path_str: str = manifest_path  
        path_obj: Any = self.spark._jvm.org.apache.hadoop.fs.Path(path_str)  
        lock_path_obj: Any = self.spark._jvm.org.apache.hadoop.fs.Path(path_str + ".lock")  
        lock_timeout: int = 120  
        sleep_interval: int = 2  
  
        def acquire_lock():  
            """Creates a .lock file, waiting if it already exists."""  
            start = time.time()  
            while self.fs.exists(lock_path_obj):  
                if time.time() - start > lock_timeout:  
                    raise TimeoutError(f"Timeout acquiring manifest lock: {lock_path_obj}")  
                time.sleep(sleep_interval)  
            self.fs.create(lock_path_obj).close()  
  
        def release_lock():  
            """Deletes the .lock file."""  
            if self.fs.exists(lock_path_obj):  
                self.fs.delete(lock_path_obj, False)  
  
        try:  
            acquire_lock()  
            stream = self.fs.append(path_obj) if self.fs.exists(path_obj) else self.fs.create(path_obj)  
            writer = self.spark._jvm.java.io.OutputStreamWriter(stream, "UTF-8")  
            for row_str in rows:  
                writer.write(row_str + "\n")  
            writer.close()  
            self.logger.info(f"Wrote {len(rows)} entries to manifest '{manifest_path}' with status '{status}'")  
        except Py4JJavaError as e:  
            self.logger.error("Failed to write manifest due to a Java error.")  
            raise RuntimeError("Error while writing the manifest") from e  
        finally:  
            release_lock()  
  
    def _load_and_enrich_raw_data(self, dates: List[str]) -> DataFrame:  
        self.logger.info("Loading and enriching raw data...")  
  
        meta_df = self._load_meta(dates).persist(StorageLevel.MEMORY_AND_DISK)  
        preclick_df = self._load_preclick(dates).persist(StorageLevel.MEMORY_AND_DISK)  
  
        # Join with 'preclick' data  
        joined_df = meta_df.alias("meta").join(  
            preclick_df.alias("pre"),  
            on=[  
                F.col("meta.event_date") == F.col("pre.event_date"),  
                F.col("meta.event_time") == F.col("pre.event_time"),  
                F.col("meta.resp_bid_id") == F.col("pre.bid_id"),  
                F.col("meta.ssp_id") == F.col("pre.ssp_id"),  
                F.col("meta.dsp_id") == F.col("pre.dsp_id"),  
                F.col("meta.user_id") == F.col("pre.user_id"),  
                F.col("meta.imp_id") == F.col("pre.imp_id"),  
                F.col("meta.imp_id_source") == F.col("pre.imp_id_source"),  
            ],  
            how='left'  
        ).select(  
            "meta.*",  
            "pre.page_depth",  
            "pre.avg_visit_duration"  
        )  
        meta_count = meta_df.count()  
        final_count = joined_df.count()  
  
        meta_df.unpersist()  
        preclick_df.unpersist()  
  
        self.logger.info(f"Initial count: {meta_count}")  
        self.logger.info(f"Count after first join: {final_count}")  
  
        # Integrity check: the number of rows should not change after the second JOIN  
        if meta_count != final_count:  
            msg = f"Assertion failed: row count changed after preclick join. Before={meta_count}, After={final_count}"  
            self.logger.error(msg)  
            raise ValueError(msg)  
  
        return joined_df  
  
    def _read_clickhouse(self, query: str) -> DataFrame:  
        """Reads data from a ClickHouse table using a JDBC connection.  
  
        Args:            query (str): The SQL query to execute. It must be a subquery                with an alias (e.g., `(SELECT ... FROM ...) AS subq`).  
        Returns:            DataFrame: A Spark DataFrame containing the query results.        """        return self.spark.read.format("jdbc") \  
            .option("driver", self.clickhouse_config["driver"]) \  
            .option("url", self.clickhouse_config["url"]) \  
            .option("user", self.clickhouse_config["user"]) \  
            .option("password", self.clickhouse_config["password"]) \  
            .option("dbtable", query) \  
            .option("fetchsize", "10000") \  
            .load()  
  
    def _load_meta(self, dates: List[str]) -> DataFrame:  
        """Loads data from the `rtb.meta_v8` table.  
  
        Args:            dates (List[str]): A list of dates in 'YYYY-MM-DD' format.  
        Returns:            DataFrame: A Spark DataFrame.        """        query = self._get_meta_query(dates)  
        return self._read_clickhouse(query)  
  
    def _load_preclick(self, dates: List[str]) -> DataFrame:  
        """Loads data from the yandex_preclick table in ClickHouse and removes duplicates.  
  
        Args:            dates (List[str]): A list of dates in 'YYYY-MM-DD' format.  
        Returns:            DataFrame: A Spark DataFrame.        """        query = self._get_preclick_query(dates)  
        df = self._read_clickhouse(query)  
        df = df.withColumn("unique_id", F.monotonically_increasing_id())  
  
        window = Window.partitionBy(  
            "event_date", "event_time", "bid_id", "ssp_id", "dsp_id", "user_id", "imp_id", "imp_id_source"  
        ).orderBy(F.col("unique_id").desc())  
  
        return (  
            df.withColumn("rn", F.row_number().over(window))  
            .filter(F.col("rn") == 1)  
            .drop("rn", "unique_id")  
        )  
  
    @staticmethod  
    def _get_salt_query(dates: List[str], entity: str) -> str:  
        return f"""  
            (SELECT max(size) AS max_group_size FROM (                SELECT count() AS size                FROM rtb.meta_v8                WHERE event_date IN ({",".join([f"'{d}'" for d in dates])})  
                GROUP BY {entity}  
            )) as t            """  
    @staticmethod  
    def _get_meta_query(dates: List[str]) -> str:  
        """Returns the SQL query to get 'meta' data.  
  
        Args:            dates (List[str]): A list of dates in 'YYYY-MM-DD' format.  
        Returns:            str: The complete SQL query as a string.        """        return f"""  
            (SELECT *            FROM rtb.meta_v8            WHERE event_date IN ({",".join([f"'{d}'" for d in dates])})  
            ) AS subq            """  
    @staticmethod  
    def _get_preclick_query(dates: List[str]) -> str:  
        """Returns the SQL query to get 'preclick' data.  
  
        Args:            dates (List[str]): A list of dates in 'YYYY-MM-DD' format.  
        Returns:            str: The complete SQL query as a string.        """        return f"""  
            (SELECT                            page_depth,  
                avg_visit_duration             FROM rtb.yandex_preclick             WHERE event_date IN ({",".join([f"'{d}'" for d in dates])})  
            ) AS subq            """