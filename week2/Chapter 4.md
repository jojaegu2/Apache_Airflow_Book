---
Chapter: Airflow 콘텍스트를 사용하여 태스크 템플릿 작업하기
---
# 증분 데이터 처리
### 사용할 데이터
위키피디아에서 제공하는 페이지뷰 데이터를 이용
파일 형태: gzip, text(압축해제 후)

| title    | description                                                                   |
| -------- | ----------------------------------------------------------------------------- |
| 메타데이터 정보 | https://wikitech.wikimedia.org/wiki/Data_Platform/Data_Lake/Traffic/Pageviews |
| 데이터   경로 | https://dumps.wikimedia.org/other/pageviews/                                  |

![[Airflow - Wikipedia 데이터 정보.png]]

![[Airflow - Wikipedia 데이터 분석.png]]


## 프로세스
![[Airflow - 증분 처리 과정.png]]

### 데이터 다운로드
`https://dumps.wikimedia.org/other/pageviews/{year}/{year}-{month}/pageviews-{year}{month}{day}{hour}0000.gz`
날짜와 시간을 위의 URL에 입력하여 데이터를 추출

### BashOperator 를 이용하는 방법
```python
import airflow
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

with DAG(
    dag_id='wikipedia_pageview',
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval='@hourly',
    catchup=False,
) as dag:
    get_data = BashOperator(
        task_id='get_data',
        bash_command=(
            'curl -o /opt/airflow/tmp/wikipageviews.gz '
            'https://dumps.wikimedia.org/other/pageviews/'
            '{{ logical_date.year }}/'
            '{{ logical_date.year }}-'
            '{{ "{:02}".format(logical_date.month) }}/'
            'pageviews-{{ logical_date.year }}'
            '{{ "{:02}".format(logical_date.month) }}'
            '{{ "{:02}".format(logical_date.day) }}'
            '{{ "{:02}".format(logical_date.hour) }}0000.gz'
        )
    )

    get_data

```

- {{ `template_name` }}
	- `template_name`에 지정된 airflow 템플릿을 지정
	- template은 런타임 시 값을 입려하기 때문에 프로그래밍시에는 값을 확인할 수 없다.
	- `template_name`이 airflow 템플릿에 속한 값이 아니라면 문자열 그대로 해석된다.
- `execution_date` 대신 `logical_date` 템플릿을 사용
	- `logical_date` 도 `execution_date` 처럼 `Pendulum.datetime` 으로 표현되었으므로 동일하게 처리가 가능하다
	- `logical_date.year, logical_date.month, logical_date.day` 등 접근 가능
- {:02} 표현은 2자리를 나타내면서 빈 자리는 0으로 채운다.
	- ex) 7 > 07
- 위의 date는 UTC 기반으로 나타내기 때문에 대한민국 시간보다 9시간 늦다.

### PythonOperator를 이용하는 방법
```python
from urllib import request

import airflow
from airflow import DAG
from airflow.operators.python import PythonOperator


def _get_data(**context):
    year, month, day, hour, *_ = context['logical_date'].timetuple()
    url = (
        'https://dumps.wikimedia.org/other/pageviews/'
        f'{year}/{year}-{month:0>2}/'
        f'pageviews-{year}{month:0>2}{day:0>2}-{hour:0>2}0000.gz'
    )
    output_path = '/opt/airflow/tmp/wikipageviews.gz'
    request.urlretrieve(url, output_path)


# def _get_data(logical_date, **context): # context에는 logical_date 키를 제외한 나머지 값들을 포함한다.
#     year, month, day, hour, *_ = logical_date.timetuple()
#     url = (
#         'https://dumps.wikimedia.org/other/pageviews/'
#         f'{year}/{year}-{month:0>2}/'
#         f'pageviews-{year}{month:0>2}{day:0>2}-{hour:0>2}0000.gz'
#     )
#     output_path = '/opt/airflow/tmp/wikipageviews.gz'
#     request.urlretrieve(url, output_path)


with DAG(
    dag_id='wikipedia_pageview_python',
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval='@hourly',
    catchup=False,
) as dag:
    get_data = PythonOperator(
        task_id='get_data',
        python_callable=_get_data
    )

    get_data

```
- `BashOperator`와 다르게 `PythonOperator`는 템플릿을 인수로 사용하지 않고 별도로 런타임 콘텍스트를 적용할 수 있는 **python_callable** 인수를 사용한다.
- `PythonOperator`는 `python_callable` 인수에 함수를 콜러블을 제공한다.
	- 콜러블: 함수를 콜러블 객체로 만들어 주는 것
- Airflow 1 버전에서는 `PythonOperator` 상에 `provide_context=True`로 설정해야 모든 태스크 컨텍스트 변수를 호출할 수 있었다.
	- Airflow 2 버전에서부터는 `PythonOperator` 가 콜러블 인수 이름으로부터 컨텍스트 변수가 호출 가능한지 판단하기 때문에 `provide_context=True`로 설정할 필요가 없다.
- 시멘틱 표현을 위해서 `**context` 라는 값으로 컨텍스트를 키워드 인자로 전달
	- 키워드 인자를 함수 내부에서 파싱해서 사용
- `**context` 키워드 인자 대신에 명시적으로 컨텍스트 내 키를 변수로 전달하여 사용할 수 있다.
	- 주석처리된 `_get_data` 함수가 예시이다.
	- 가독성이 높아지고, 명시적인 파라미터 전달을 통해 linter와 타입 힌팅 같은 도구를 사용하는 이점을 얻을 수 있다.
![[Airflow - context 전달 과정.png]]
1. python_callable에 지정된 함수를 호출
2. 태스크 컨텍스트 변수들을 키워드 인수로 전달
3. python에서는 키워드 인수를 파싱해서 해당 값을 사용

### PythonOperator를 통해 태스크 콘텍스트 확인하기
```python
import json

import airflow
from airflow import DAG
from airflow.operators.python import PythonOperator


def _print_context(**kwargs):
    for k, v in kwargs.items():
        with open('/opt/airflow/tmp/context', 'a') as f:
            f.write(f'{k}, {v}\n')


with DAG(
    dag_id='template_print',
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval='@hourly',
    catchup=False,
) as dag:
    print_context = PythonOperator(
        task_id='print_context',
        python_callable=_print_context,
    )

    print_context
```

Context
```text
conf, <airflow.configuration.AirflowConfigParser object at 0xffffb491d070>
dag, <DAG: template_print>
dag_run, <DagRun template_print @ 2024-08-13 02:12:17.586236+00:00: manual__2024-08-13T02:12:17.586236+00:00, state:running, queued_at: 2024-08-13 02:12:17.590847+00:00. externally triggered: True>
data_interval_end, 2024-08-13 02:00:00+00:00 
data_interval_start, 2024-08-13 01:00:00+00:00
ds, 2024-08-13
ds_nodash, 20240813
execution_date, 2024-08-13 02:12:17.586236+00:00
expanded_ti_count, None
inlets, []
logical_date, 2024-08-13 02:12:17.586236+00:00
macros, <module 'airflow.macros' from '/home/airflow/.local/lib/python3.9/site-packages/airflow/macros/__init__.py'>
map_index_template, None
next_ds, 2024-08-13
next_ds_nodash, 20240813
next_execution_date, 2024-08-13 02:12:17.586236+00:00
outlets, []
params, {}
prev_data_interval_start_success, 2024-08-13 01:00:00+00:00
prev_data_interval_end_success, 2024-08-13 02:00:00+00:00
prev_ds, 2024-08-13
prev_ds_nodash, 20240813
prev_execution_date, 2024-08-13 02:12:17.586236+00:00
prev_execution_date_success, 2024-08-13 02:11:26.486176+00:00
prev_start_date_success, 2024-08-13 02:11:26.894598+00:00
prev_end_date_success, 2024-08-13 02:11:27.855192+00:00
run_id, manual__2024-08-13T02:12:17.586236+00:00
task, <Task(PythonOperator): print_context>
task_instance, <TaskInstance: template_print.print_context manual__2024-08-13T02:12:17.586236+00:00 [running]>
task_instance_key_str, template_print__print_context__20240813
test_mode, False
ti, <TaskInstance: template_print.print_context manual__2024-08-13T02:12:17.586236+00:00 [running]>
tomorrow_ds, 2024-08-14
tomorrow_ds_nodash, 20240814
triggering_dataset_events, defaultdict(<class 'list'>, {})
ts, 2024-08-13T02:12:17.586236+00:00
ts_nodash, 20240813T021217
ts_nodash_with_tz, 20240813T021217.586236+0000
var, {'json': None, 'value': None}
conn, None
yesterday_ds, 2024-08-12
yesterday_ds_nodash, 20240812
templates_dict, None
```


### PythonOperator에 변수 제공
- `PythonOperator`에서는 콜러블 함수에서 추가 인수를 전달하는 방법이 있다.
#### op_args
```python
get_data=PythonOperator(
	task_id='get_data',
	python_callable=_get_data,
	op_args=['/tmp/wikipageviews.gz'],
	dag=dag
)
```
- `op_args` 에 주어진 리스트의 각 값이 콜러블 함수에 전달된다.
	- `_get_data`호출 시 `/tmp/wikipageviews.gz` 라는 값을 인자로 전달한다.
#### op_kwargs
```python
get_data=PythonOperator(
	task_id='get_data',
	python_callable=_get_data,
	op_kwargs={'output_path': '/tmp/wikipageviews.gz'},
	dag=dag
)
```
- `op_kwargs`에 주어진 딕셔너리의 각 키와 값이 콜러블 함수에 전달됩니다.
	- `_get_data` 호출시 `output_path` 변수에 `/tmp/wikipageviews.gz`라는 값이 전달된다.


#### 콜러블 함수에 변수로 템플릿을 전달하는 방법
```python
fetch_events = PythonOperator(
    task_id="fetch_events",
    python_callable=get_events,
    op_kwargs={
        'start_date': "{{ data_interval_start.strftime('%Y-%m-%d') }}", # 데이터 윈도우 시작 지점
        'end_date': "{{ data_interval_end.strftime('%Y-%m-%d') }}", # 데이터 윈도우 종료 지점
    },
    dag=dag,
)
```

## 템플릿의 인수 검사하기
### 웹서버에서 확인해보기
DAG - Task - Details 내 Rendered Templates
![[Airflow - task details.png]]
- Task - Details 에서 Rendered Templates 를 확인할 수 있다.
#### airflow CLI
`airflow tasks render <dag_id> <task_id> <execution_date>`
![[Airflow - tasks render.png]]

## Postgres에 데이터 적재하기
Xcom 예시는 5장에서 다루기 때문에 여기서는 외부 파일로 데이터를 유지하면서 데이터를 적재하는 방식을 사용
```python
from urllib import request

import airflow
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator


def _get_data(output_path, logical_date, **context):
    year, month, day, hour, *_ = (logical_date.subtract(hours=5)).timetuple()
    url = (
        'https://dumps.wikimedia.org/other/pageviews/'
        f'{year}/{year}-{month:0>2}/'
        f'pageviews-{year}{month:0>2}{day:0>2}-{hour:0>2}0000.gz'
    )
    request.urlretrieve(url, output_path)


def _fetch_pageviews(pagenames, input_path, sql_path, logical_date, **context):
    result = dict.fromkeys(pagenames, 0)
    with open(input_path, 'r') as f:
        for line in f:
            domain_code, page_title, view_counts, _ = line.split(' ')
            if domain_code == 'en' and page_title in pagenames:
                result[page_title] = view_counts

    execution_date = logical_date.subtract(hours=5)
    with open(sql_path, 'w') as f:
        for pagename, pageviewcount in result.items():
            f.write(
                'INSERT INTO airflow.pageview_counts'
                f" VALUES ('{pagename}', {pageviewcount}, '{execution_date}');\n"
            )


with DAG(
    dag_id='wikipedia_pageview_python',
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval='@hourly',
    catchup=False,
    template_searchpath='/opt/airflow/tmp'
) as dag:
    get_data = PythonOperator(
        task_id='get_data',
        python_callable=_get_data,
        op_kwargs={'output_path': '/opt/airflow/tmp/wikipageviews.gz'}
    )

    extract_gz = BashOperator(
        task_id='extract_gz',
        bash_command='gunzip --force /opt/airflow/tmp/wikipageviews.gz'
    )

    fetch_pageview = PythonOperator(
        task_id='fetch_pageview',
        python_callable=_fetch_pageviews,
        op_kwargs={
            'pagenames': {
                'Google',
                'Amazon',
                'Apple',
                'Microsoft',
                'Facebook',
            },
            'input_path': '/opt/airflow/tmp/wikipageviews',
            'sql_path': '/opt/airflow/tmp/wikipageviews.sql'
        }
    )

    write_to_postgres = PostgresOperator(
        task_id='write_to_postgres',
        postgres_conn_id='postgres_conn',
        sql='wikipageviews.sql',
    )

    get_data >> extract_gz >> fetch_pageview >> write_to_postgres

```
- Postgres 적재를 위해 PostgresOperator 설치
	- `pip install apache-airflow-providers-postgres`
- sql 실행할 파일을 .sql 파일 형태로 생성
- webserver나 airflow CLI을 통해 connection 생성
- PostgresOperator에 sql 명시
	- DAG의 template_searchpath에 sql 파일 경로를 지정

### connection 생성
Webserver > Admin > Connection
![[Airflow - Connection.png]]
CLI
```bash
airflow connections add \
	--conn-type postgres \
	--conn-host host \
	--conn-login username \
	--conn-password password \
	--port 5432 \
	--database database \
	postgres_conn
```