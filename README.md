# Airflow-Practice
## Airflow-Practice - проект для выгрузки данных из бд Postgres, объединения их и сохранения в объектное хранилище данных в виде файла csv. </br>
### Проект создан в целях отработки навыков работы с Airflow Apache.  </br>
Стек используемых технологий:
1. Airflow Apache v2.0.1,
2. Docker compose
3. PostgresSQL
4. Celery
5. Flower
6. MinIO
---
### Для развертывания приложения необходимо:
1. Установить Docker. В документации Airflow Apache сообщается, что в зависимости от вашей операционной системы вам может потребоваться настроить Docker на использование не менее 4,00 ГБ памяти для правильной работы контейнеров Airflow
2. Установить Docker Compose версии не ниже 2.14.0
3. Сохранить файл docker-compose.yaml в папку с проектом
Файл содержит несколько сервисов:
* postgres: metadata database
* db-postgres1 и db-postgres2: бд, из которых будут забираться данные для объединения и создания файла csv. В .env файл будет необходимо добавить данные для создания бд(пункт 5)
* redis: брокер, пересылающий сообщения от планировщика к воркеру
* airflow-webserver: веб-сервер, доступный по адресу http://localhost:8080
* airflow-scheduler: планировщик, отслеживающий все DAGи
* airflow-worker: воркер, выполняющий задачи, поставленные планировщиком. Будет использоваться Celery
* airflow-init: инициализация настройки, подготовка бд
* flower: веб приложение для мониторинга и администрирования задач Celery, доступный по адресу http://localhost:5555
* s3: объектное хранилище, в котором будут лежать созданные csv файлы, доступно по адресу http://localhost:9001
4. В корне проекта создать подмонтированные папки командой ```mkdir -p ./dags ./logs ./plugins ./config```
5. В корне проекта создайть файл .env. Если вы работаете на Linux перед запуском нужно указать AIRFLOW_UID ```echo -e "AIRFLOW_UID=$(id -u)" > .env``` </br>
Добавить переменные в .env:
* AIRFLOW_GID=0 идентификатор группы
* POSTGRES1_USER имя пользователя для первой бд Postgres
* POSTGRES1_PASSWORD пароль для первой бд Postgres
* POSTGRES1_DB название бд для первой бд Postgres
* POSTGRES2_USER имя пользователя для второй бд Postgres
* POSTGRES2_PASSWORD пароль для второй бд Postgres
* POSTGRES2_DB название бд для второй бд Postgres
* MINIO_ROOT_USER имя пользователя для MinIO
* MINIO_ROOT_PASSWORD пароль для MinIO
6. Выполнить миграцию базы данных и создать первую учетную запись пользователя командой ```docker compose up airflow-init```. Созданная учетная запись имеет логин airflow и пароль airflow
7. Запустить композ файл командой ```docker compose up```
8. Перейти по адресу http://localhost:8080 и настроить connections для db-postgres1 и db-postgres2:
* Conn Id - впишите свое название коннекта
* Conn Type - выберите Postgres
* Host - берется из названия сервиса композ файла, в данном случае db-postgres1 или db-postgres2
* Schema - название бд, которое передавали при создании бд, в данном случае значение POSTGRES1_DB или POSTGRES2_DB
* Login - имя пользователя бд, в данном случае значение POSTGRES1_USER или POSTGRES2_USER
* Password - пароль от бд, в данном случае значение POSTGRES1_PASSWORD или POSTGRES2_PASSWORD
* Port - 5432
8. В интерфейсе MinIO, по адресу http://localhost:9001, создать Access Key и Bucket
9. Создать connections в Airflow для MinIO:
* Conn Id - впишите свое название коннекта
* Conn Type - S3
* Extra - {
    "aws_access_key_id":"<значение Acces key, которое было создано в MinIO>",
    "aws_secret_access_key": "<значение Secret key, которое было создано в MinIO>>",
    "host": "http://s3:9000"
 }
### Примеры использования  </br>
Есть postgres_db1 и postgres_db2 с одинаковым названием таблиц('vinoteka'), но разными данными. </br>
Необходимо выгрузить, объединить данные из этих таблиц, и положить их в бакет 'postgres-data-csv' MinIo в файл формата csv. </br>
Pipeline состоит из 3-х задач: extract_data_task, combine_data_task, upload_to_minio_task. Настроен на ежедневный запуск. </br>
В dags создаем Python Package 'data_collection_processing_and_writing_to_csv', в котором добавляем functions.py:
```
from datetime import datetime

import pandas as pd
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.providers.postgres.hooks.postgres import PostgresHook

# Список баз данных PostgreSQL, из которых извлекаются данные.
DATABASES = ['postgres_db1', 'postgres_db2']

# Название таблицы, из которой извлекаются данные.
TABLE_NAME = 'vinoteka'

# Название S3-бакета, куда загружаются объединенные данные.
BUCKET = 'postgres-data-csv'


def extract_data(ds, **kwargs):
    """
    Извлекает данные из указанных баз данных PostgreSQL и объединяет их в один DataFrame.

    Args:
        ds: Шаблон даты (например, {{ ds }}), передаваемый Airflow.

    Возвращаемое значение:
        Объединенные данные сохраняются в виде pickle-файла '/tmp/combined_data.pkl'.
    """
    databases = DATABASES
    dataframes = []

    # Цикл по списку баз данных и извлечение данных из каждой.
    for db in databases:
        hook = PostgresHook(postgres_conn_id=db)
        sql = f"SELECT * FROM {TABLE_NAME}"
        df = hook.get_pandas_df(sql)
        dataframes.append(df)
    
    # Объединение данных из всех баз данных в один DataFrame.
    combined_data = pd.concat(dataframes)
    
    # Сохранение объединенных данных в локальный файл.
    combined_data.to_pickle('/tmp/combined_data.pkl')


def combine_data(ds, **kwargs):
    """
    Дополнительно обрабатывает объединенные данные (если необходимо) и сохраняет их.

    Args:
        ds: Шаблон даты (например, {{ ds }}), передаваемый Airflow.

    Возвращаемое значение:
        Обработанные данные сохраняются в виде pickle-файла '/tmp/merged_data.pkl'.
    """
    combined_data = pd.read_pickle('/tmp/combined_data.pkl')
    
    # Дополнительные операции по обработке данных можно добавить здесь.
    
    # Сохранение обработанных данных в локальный файл.
    combined_data.to_pickle('/tmp/merged_data.pkl')


def upload_to_minio(ds, **kwargs):
    """
    Загружает обработанные данные в S3-совместимый MinIO хранилище в виде CSV-файла.

    Args:
        ds: Шаблон даты (например, {{ ds }}), передаваемый Airflow.

    Возвращаемое значение:
        CSV-файл загружается в указанный S3-бакет с уникальным именем.
    """
    merged_data = pd.read_pickle('/tmp/merged_data.pkl')
    
    # Определение пути к локальному CSV-файлу.
    csv_path = '/tmp/merged_data.csv'
    
    # Сохранение данных в CSV-файл.
    merged_data.to_csv(csv_path, index=False)
    
    # Создание уникального имени для загружаемого файла, основанного на текущем времени.
    now = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
    object_name = f'merged_data_{now}.csv'
    
    # Подключение к MinIO и загрузка файла в S3-бакет.
    minio_hook = S3Hook('minio_conn_id')
    minio_hook.load_file(
        bucket_name=BUCKET,
        key=object_name,
        filename=csv_path,
        replace=True,
    )
```
Настраиваем __init.py__
```
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.utils.dates import days_ago

from .functions import extract_data, combine_data, upload_to_minio

# Определение аргументов по умолчанию для всех задач в DAG
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': days_ago(1),
    'email_on_failure': False,
    'email_on_retry': False,
}

# Создание DAG для извлечения данных, их объединения и загрузки в MinIO
with DAG(
    'data_collection_processing_and_writing_to_csv',
    default_args=default_args,
    description='Extract data, combine and upload to minio',
    schedule_interval='@daily',
) as dag:
    
    # Оператор для извлечения данных из баз данных PostgreSQL
    extract_data_task = PythonOperator(
        task_id='extract_data',
        python_callable=extract_data,
        provide_context=True,
    )
    
    # Оператор для объединения и обработки данных
    combine_data_task = PythonOperator(
        task_id='combine_data',
        python_callable=combine_data,
        provide_context=True,
    )

    # Оператор для загрузки обработанных данных в MinIO в формате CSV
    upload_to_minio_task = PythonOperator(
        task_id='upload_to_minio',
        python_callable=upload_to_minio,
        provide_context=True,
    )
    
    # Определение порядка выполнения задач
    extract_data_task >> combine_data_task >> upload_to_minio_task
```
