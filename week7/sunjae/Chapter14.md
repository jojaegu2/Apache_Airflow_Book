# Chapter 14 프로젝트: 뉴욕에서 가장 빠른 길 찾기
- 뉴욕 안에서 A에서 B로 가려면 어떤 교통수단이 빠른지 데이터를 추출 및 로드, 변환
- 이동하는 지역과 하루 중 시간에 따라 어떤 교통수단이 더 빠른지 질의하는 프로젝트 수행

- 도커 컴포즈 구성
    - Citi Bike 데이터를 제공하는 REST API
    - Yellow Cab 택시 데이터를 제공하는 파일
    - S3 프로토콜을 지원하는 객체 저장소인 MinIO
    - 데이터 쿼리 및 저장을 위한 PostreSQL 데이터베이스
    - 결과를 표시하는 Flask애플리케이션  
![alt text](./img2/image.png)


>http://localhost:5432: Airflow Postgres DB (airflow/airflow)  
>http://localhost:5433: NYC Taxi Postgres DB (taxi/ridetlc)  
>http://localhost:5434: Citi Bike Postgres DB (citi/cycling)  
>http://localhost:5435: NYC Transportation results Postgres DB (nyc/tr4N5p0RT4TI0N)  
>http://localhost:8080: Airflow webserver (airflow/airflow)  
>http://localhost:8081: NYC Taxi static file server  
>http://localhost:8082: Citi Bike API (citibike/cycling)  
>http://localhost:8083: NYC Transportation API  
>http://localhost:9000: MinIO (AKIAIOSFODNN7EXAMPLE/wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY)  

- NYC Yellow Taxi : https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page
- NYC Citi Bike: https://www.citibikenyc.com/system-data
- Yellow Cab 및 Citi Bikes 탑승에 대한 데이터는 매월 제공

## 14.1 데이터에 대한 이해
- Yello Cab 데이터 위치 http://localhost:8081
- Citi Bike 데이터 위치 http://localhost:8082

### 14.1.1 Yellow Cab 파일 공유
- 데이터는 정적 CSV 파일로 제공되며 각 CSV 파일에는 지난 15분 동안 완료된 택시 탑승이 포함된다.
- 한 시간 분량의 데이터만 보관하며 1시간 지난 데이터는 자동으로 제거된다.
- curl http://localhost:8081 로 샘플 요청
- 인덱스는 사용 가능한 파일 리스트를 반환한다.
- curl http://localhost:8081/[csv파일] 로 데이터를 조회할 수 있다.
- 데이터 구성
    - 시작 및 종료 시간과 시작 및 종료 구역 ID와 함께 택시 탑승에 대한 위치 정보

### 14.1.2 Citi Bike REST API
- 이 API는 기본 인증을 시행하므로 사용자 이름(citibike)과 사용자 패스워드(cycling)을 제공해야한다.
- API는 사전에 정의된 시간(1시간) 동안 발생한 자전거 이용에 대한 정보를 전달해준다.
- curl --user citibkie:cycling http://localhost:8082/recent/< period:일,시,분 >/< amount:기간의 수 >
- 데이터 구성
    - Citi Bike를 사용한 개별 주행기록
    - 시작 및 종료시간, 시작 및 종료 위치의 위도/경도 좌표

### 14.1.3 접근 계획 결정
![alt text](./img2/image-1.png)  
- 두 지점 사이의 거리를 비교하려면 두 데이터 세트의 위치를 같은 기준으로 매핑해야 한다.
- Citi Bike 정거장의 위도/경도를 택시구역에 매핑하여 적용할 수 있도록 설정한다.
- Yellow Cab 데이터는 요청 시간 기준 1시간 분량의 데이터만 제시되므로 데이터를 다운로드해야한다.(MinIO에 데이터 저장)

## 14.2 데이터 추출
- Yellow Cab은 15분 간격으로 제공되며 Citi Bike 간격 조건을 걸 수 있으므로 15분 간격으로 요청
- 동일한 간격으로 두 개의 요청을 만들고 모든 데이터를 병렬로 처리할 수 있다.
```python
import airflow.utils.dates
from airflow.models import DAG
dag = DAG(
    dag_id="nyc_dag",
    schedule_interval="*/15 * * * *", # 15분마다 수행
    start_date=airflow.utils.dates.days_ago(1),
    catchup=False,
)
```
### 14.2.1 Citi Bike 데이터 다운로드하기
- SimpleHttpOperator는 요청에 대한 응답을 받을 수는 있지만, 응답 내용을 저장할 수는 없다.
- PythonOperator를 이용해 Citi Bike API를 쿼리하고 MinIO스토리지에 저장하도록한다.
```python
import json
import requests
from airflow.hooks.base import BaseHook
from airflow.models import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from requests.auth import HTTPBasicAuth

# 결과 파일 이름에 Airflow 태스크의 타임스탬프를 사용한다.
def _download_citi_bike_data(ts_nodash, **_):
    # Airflow 연결에서 Citi Bike 자격 증명을 로드한다.
    citibike_conn = BaseHook.get_connection(conn_id="citibike")
    url = f"http://{citibike_conn.host}:{citibike_conn.port}/recent/minute/15"
    response = requests.get(
        url, auth=HTTPBasicAuth(citibike_conn.login,citibike_conn.password)
    )
    data = response.json() 
    
    # S3Hook을 사용하여 MinIO와 통신한다.
    s3_hook = S3Hook(aws_conn_id="s3")
    s3_hook.load_string(
        string_data=json.dumps(data),
        key=f"raw/citibike/{ts_nodash}.json", 
        bucket_name="datalake"
    )

download_citi_bike_data = PythonOperator(
    task_id="download_citi_bike_data",
    python_callable=_download_citi_bike_data,
    dag=dag,
)
```
- HTTP-to_S3 태스크에 사용할 Airflow 오퍼레이터는 없지만 Hook 및 커넥션을 적용할 수 있다.
- Citi Bike API, MinIO 스토리지(S3Hook)에 연결해야한다.

```sh
➥ export AIRFLOW_CONN_CITIBIKE=http://citibike:cycling@citibike_api:5000
➥ export AIRFLOW_CONN_S3="s3://@?host=http://minio:9000&aws_access_key_id
        =AKIAIOSFODNN7EXAMPLE&aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfi
        CYEXAMPLEKEY"
```
- S3Hook은 기본적으로 http://s3.amazonaws.com 에서 AWS S3와 통신하지만, MinIO와 연결을 해준다.  
![alt text](./img2/image-2.png)

```python
s3_hook = S3Hook(aws_conn_id="s3")
s3_hook.load_string(
    string_data=json.dumps(data),
    key=f"raw/citibike/{ts_nodash}.json", # 키 이름에 템플릿이 지정된 태스크 타임 스탬프를 사용하여 파일을 저장한다.
    bucket_name="datalake"
)
```
![alt text](./img2/image-3.png)  

### 14.2.2 Yellow Cab 데이터 다운로드
- HTTP-S3 태스크이지만 앞서 작업한 것 과 차이가 있다.
    - 파일을 별도로 제공한다.
    - Yellow Cab데이터는 CSV파일로 데이터를 반환한다.
    - 파일 이름을 미리 알지 못한다. 파일 리스트를 받으려면 인덱스를 먼저 나열해야 한다.

```python
def _download_taxi_data():
    taxi_conn = BaseHook.get_connection(conn_id="taxi")
    s3_hook = S3Hook(aws_conn_id="s3")

    url = f"http://{taxi_conn.host}"
    response = requests.get(url) # 파일 리스트를 가져온다.
    files = response.json()

    for filename in [f["name"] for f in files]:
        response = requests.get(f"{url}/{filename}") # 하나의 파일을 가져온다.
        s3_key = f"raw/taxi/{filename}" 

        try: # MinIO에 파일 업로드
            s3_hook.load_string(
                string_data=response.text,
                key=s3_key,
                bucket_name="datalake",
            )
            print(f"Uploaded {s3_key} to MinIO.")
        except ValueError: # 파일 존재 시 ValueError를 예외처리
            print(f"File {s3_key} already exists.")
    
download_taxi_data = PythonOperator(
    task_id="download_taxi_data",
    python_callable=_download_taxi_data,
    dag=dag,
)
```
![alt text](./img2/image-4.png)  
![alt text](./img2/image-5.png)  

## 14.3 데이터에 유사한 변환 적용
- Citi Bike 스테이션 좌표를 Yellow Cab 구역에 매핑하게되면 두 데이터를 비교할 수 있기에 변환 작업이 필요하다.
- 빅데이터 환경에서는 SparkSubmitOperator나 SSHOperator와 같은 오퍼레이터로 트리거 할 수 있다.이후 Spark태스크는 S3에서 읽고 데이터에 변환을 적용한 다음 S3에 다시 저장하는 절차를 수행할 수 있다.
- 작은 규모에서는 Pandas를 적용할 수 있지만 PandasOperator가 없으므로 PythonOperator를 사용하여 실행한다.
- Pandas 태스크가 시스템 리소스 한계에 도달하면 시스템 중단하고, Airflow역시 함께 중단될 수 있다.

- 현재 /raw엔 두 개의 데이터 세트가 있다.
    - /raw/citibike/{ts_nodash}.json
    - /raw/taxi/*.csv
- 두 데이터를 Pandas로 읽고 변환하여 다음과 같이 저장한다.
    - /processed/citibike/{ts_nodash}.parquet
    - /processed/taxi/{ts_nodash}.parquet
- Pandas DataFrame을 이용해 단일 오퍼레이터를 구현할 수 있다.

```python
import logging

from airflow.models import BaseOperator
from airflow.utils.decorators import apply_defaults


class PandasOperator(BaseOperator):
    # 모든 kwards인수는 템플릿 값을 보유할 수 있다.
    template_fields = (
        "_input_callable_kwargs",
        "_transform_callable_kwargs",
        "_output_callable_kwargs",
    )

    @apply_defaults
    def __init__(
        self,
        input_callable,
        output_callable,
        transform_callable=None,
        input_callable_kwargs=None,
        transform_callable_kwargs=None,
        output_callable_kwargs=None,
        **kwargs,
    ):
        super().__init__(**kwargs)

        # Attributes for reading data
        self._input_callable = input_callable
        self._input_callable_kwargs = input_callable_kwargs or {}

        # Attributes for transformations
        self._transform_callable = transform_callable
        self._transform_callable_kwargs = transform_callable_kwargs or {}

        # Attributes for writing data
        self._output_callable = output_callable
        self._output_callable_kwargs = output_callable_kwargs or {}

    def execute(self, context):
        # 입력 콜러블을 호출하여 Pandas DataFrame을 반환한다.
        df = self._input_callable(**self._input_callable_kwargs) 
        logging.info("Read DataFrame with shape: %s.", df.shape)

        # DataFrame에 대한 변환을 적용한다.
        if self._transform_callable:
            df = self._transform_callable(df, **self._transform_callable_kwargs) 
            logging.info("DataFrame shape after transform: %s.", df.shape)

        self._output_callable(df, **self._output_callable_kwargs) #Data Frame을 저장한다.
```

- input_callable : 데이터를 Pandas DataFrame으로 읽는다.
- transform_callable : DataFrame에 변환 적용
- output_callable : DataFrame 구성


```python

def get_minio_object(
    pandas_read_callable, bucket, paths, pandas_read_callable_kwargs=None
):
    s3_conn = BaseHook.get_connection(conn_id="s3")
    minio_client = Minio( # MinIO클라이언트 초기화
        s3_conn.extra_dejson["host"].split("://")[1],
        access_key=s3_conn.extra_dejson["aws_access_key_id"],
        secret_key=s3_conn.extra_dejson["aws_secret_access_key"],
        secure=False,
    )

    if isinstance(paths, str):
        if paths.startswith("[") and paths.endswith("]"):
            paths = eval(paths)
        else:
            paths = [paths]

    if pandas_read_callable_kwargs is None:
        pandas_read_callable_kwargs = {}

    dfs = []
    for path in paths:
        minio_object = minio_client.get_object(bucket_name=bucket, object_name=path)
        # MinIO에서 파일을 읽는다.
        df = pandas_read_callable(minio_object, **pandas_read_callable_kwargs) 
        dfs.append(df)
    return pd.concat(dfs) # Pandas DataFrame을 반환한다.

def transform_taxi_data(df): # 데이터 프레임 입력
    df[["pickup_datetime", "dropoff_datetime"]] = df[
        ["pickup_datetime", "dropoff_datetime"]
    ].apply(pd.to_datetime)
    df["tripduration"] = (
        (df["dropoff_datetime"] - df["pickup_datetime"]).dt.total_seconds().astype(int)
    )
    df = df.rename(
        columns={
            "pickup_datetime": "starttime",
            "pickup_locationid": "start_location_id",
            "dropoff_datetime": "stoptime",
            "dropoff_locationid": "end_location_id",
        }
    ).drop(columns=["trip_distance"])
    return df # 데이터프레임 출력

def write_minio_object(
    df, pandas_write_callable, bucket, path, pandas_write_callable_kwargs=None
):
    s3_conn = BaseHook.get_connection(conn_id="s3")
    minio_client = Minio(
        s3_conn.extra_dejson["host"].split("://")[1],
        access_key=s3_conn.extra_dejson["aws_access_key_id"],
        secret_key=s3_conn.extra_dejson["aws_secret_access_key"],
        secure=False,
    )
    bytes_buffer = io.BytesIO()
    # DataFrame 쓰기 방법에 대한 참조를 가져온다.
    pandas_write_method = getattr(df, pandas_write_callable.__name__) 
    # DataFrame 쓰기 메소드를 호출하여 MinIO에 저장할 수 있는 바이트 버퍼에 쓴다.
    pandas_write_method(bytes_buffer, **pandas_write_callable_kwargs)
    nbytes = bytes_buffer.tell()
    bytes_buffer.seek(0)
    # MinIO에 바이트 버퍼를 저장한다.
    minio_client.put_object(
        bucket_name=bucket, object_name=path, length=nbytes, data=bytes_buffer
    )

process_taxi_data = PandasOperator(
    task_id="process_taxi_data",
    input_callable=get_minio_object, # MinIO 저장소에서 CSV를 읽는다.
    input_callable_kwargs={
        "pandas_read_callable": pd.read_csv,
        "bucket": "datalake",
        "paths": "{{ ti.xcom_pull(task_ids='download_taxi_data') }}",
    },

    transform_callable=transform_taxi_data, # DataFrame에 변환을 적용한다.

    output_callable=write_minio_object, # MinIO스토리지에 Parquet을 쓴다.
    output_callable_kwargs={
        "bucket": "datalake",
        "path": "processed/taxi/{{ ts_nodash }}.parquet",
        "pandas_write_callable": pd.DataFrame.to_parquet,
        "pandas_write_callable_kwargs": {"engine": "auto"},
    },
    dag=dag,
)
```

## 14.4 데이터 파이프라인 구조화
- 'datalake'라는 버킷에 'raw', 'processed'폴더를 만들었다.
- 데이터가 종종 여러 사람, 데이터파이프라인 에서 사용되기에 배포하고 재사용하기 위해 사람과 프로세스가 읽을 수 있는 위치에 데이터를 저장해야 한다.

- 파이프라인을 재현 가능하게 만들어야 한다.
- 데이터의 사본을 저장하는 단계를 가춰야 한다.(raws폴더에 저장)
![alt text](./img2/image-6.png)  
- 이 원시데이터에서 사용자는 원하는 만큼 변경, 수정, 전처리 등과 같은 작업을 할 수 있으며, 처리된 폴더에 다시 저장하게 된다.
- 데이터 변환은 시간 집약적이므로 재실행을 피하고 결과만 다시 읽을 수 있도록한다.



## 14.5 재현 가능한 데이터 파이프 라인 개발
- **데이터 파이프라인 처리**: 원시 데이터를 가공하고 Postgres 데이터베이스에 결과를 삽입함. 세부적인 변환 작업에 대한 논의는 생략.

- **파이프라인의 중요한 개념: **재실행 가능성(멱등성)** 보장**:
  1. **가공 데이터 저장**: `/processed` 폴더에 저장할 때, 기존 파일을 덮어쓰는 플래그 설정하여 재실행 시 파일 충돌 방지.
  2. **데이터베이스에 결과 쓰기**: 재실행 시 중복된 행이 발생할 수 있으므로, 이를 방지하기 위한 방법 필요.
  
- **멱등성 보장 방법**: 
  - 테이블에 고유한 열 추가(예: Airflow 작업 실행 날짜)로 각 작업을 식별하여 중복 삽입 방지.

```python
"""
CREATE TABLE citi_bike_rides(
 tripduration INTEGER,
 starttime TIMESTAMP,
 start_location_id INTEGER,
 stoptime TIMESTAMP,
 end_location_id INTEGER
);
"""
df = pd.read_csv(... citi bike data ...)
engine = sqlalchemy.create_engine(
 BaseHook.get_connection(self._postgres_conn_id).get_uri()
)
# Pandas DataFrame과 테이블 구조가 일치해야 한다.
df.to_sql("citi_bike_rides", con=engine, index=False, if_exists="append")
```
- df.to_sql() 실행 시, 이미 존재하는 행을 상비할지 여부는 알 수 없다.

```python
--CREATE TABLE citi_bike_rides(
-- tripduration INTEGER,
-- starttime TIMESTAMP,
-- start_location_id INTEGER,
-- stoptime TIMESTAMP,
-- end_location_id INTEGER,
-- airflow_execution_date TIMESTAMP
--);
df = pd.read_csv(... citi bike data ...)
# Pandas 데이터 프레임에 execution_date를 열로 추가한다.
df["airflow_execution_date"] = pd.Timestamp(
    context["execution_date"].timestamp(),
    unit='s',
)
engine = sqlalchemy.create_engine(
    BaseHook.get_connection(self._postgres_conn_id).get_uri()
)

# 트랜잭션 시작
with engine.begin() as conn:
    # 현재 실행날짜 기준으로 기존 레코드 삭제
    conn.execute(
        "DELETE FROM citi_bike_rides"
        f"WHERE airflow_execution_date='{context['execution_date']}';"
    )
    # 이후 쿼리 수행
    df.to_sql("citi_bike_rides", con=conn, index=False, if_exists="append")
```
- **데이터베이스 트랜잭션 시작**: 
  - 먼저 주어진 실행 날짜에 해당하는 기존 행을 삭제한 후, 새로운 행을 삽입함.
  - 실행 날짜에 해당하는 행이 없으면 삭제되지 않음.
  - 두 SQL 명령(df.to_sql()이 내부적으로 SQL 실행)을 트랜잭션으로 묶어, 두 쿼리가 모두 성공하거나 둘 다 실패하도록 보장함.
  - 실패 시 데이터베이스에 불완전한 결과가 남지 않음.

- **웹 애플리케이션 시작**: 데이터가 성공적으로 처리되고 저장된 후, http://localhost:8083에서 결과를 조회하는 웹 애플리케이션을 시작함.

![alt text](./img2/image-7.png)
  
- **Airflow 작업**: 15분 간격으로 데이터를 다운로드, 변환, 저장하는 작업을 자동으로 트리거함. 

- **백엔드 측면**: 자동화된 데이터 파이프라인이 15분마다 실행되어, 주어진 시간과 지역 사이의 택시와 Citi Bike 속도를 비교한 결과를 Postgres 데이터베이스에 저장.