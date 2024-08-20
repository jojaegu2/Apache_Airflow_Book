---
Chapter: Airflow 스케쥴링
---
## Configuration
> Docker-compose의 Worker 컨테이너와 event API 앱을 띄울 컨테이너간 통신 연결을 해야하는데 이 작업을 위해 굳이 연결할 필요성이 없어보여 기존 event API 앱을 파이썬 함수로 수정하고 DAG에서는 BashOperator 대신 PythonOperator를 호출해서 결과를 얻는 방식으로 수정
1. `docker-compose.yaml`에 결과물들을 체크하기 위해 `tmp` 디렉토리 볼륨 생성
	`- ${AIRFLOW_PROJ_DIR:-.}/tmp:/opt/airflow/tmp`
2. 파이썬 함수로 기존 앱을 변경
```python
from datetime import date, timedelta
import time

from numpy import random
import pandas as pd
from faker import Faker


def _generate_events(end_date):
    """Generates a fake dataset with events for 30 days before end date."""

    events = pd.concat(
        [
            _generate_events_for_day(date=end_date - timedelta(days=(30 - i)))
            for i in range(30)
        ],
        axis=0,
    )

    return events


def _generate_events_for_day(date):
    """Generates events for a given day."""

    # Use date as seed.
    seed = int(time.mktime(date.timetuple()))

    Faker.seed(seed)
    random_state = random.RandomState(seed)

    # Determine how many users and how many events we will have.
    n_users = random_state.randint(low=50, high=100)
    n_events = random_state.randint(low=200, high=2000)

    # Generate a bunch of users.
    fake = Faker()
    users = [fake.ipv4() for _ in range(n_users)]

    return pd.DataFrame(
        {
            "user": random_state.choice(users, size=n_events, replace=True),
            "date": pd.to_datetime(date),
        }
    )


def get_events(start_date=None, end_date=None, output_file='events.json'):
    """Get events within a specific date range."""
    print(f'start_date: {start_date} / end_date: {end_date}')

    if end_date:
        year, month, day = list(map(int, end_date.split('-')))
    else:
        year, month, day = 2024, 12, 31
    events = _generate_events(end_date=date(year=year, month=month, day=day))

    if start_date is not None:
        events = events.loc[events["date"] >= start_date]

    if end_date is not None:
        events = events.loc[events["date"] < end_date]

    events.to_json(f'/opt/airflow/tmp/{output_file}',
                   orient="records",
                   date_format="iso")

```
3. DAG의 API 호출하는 부분을 Python Operator로 변경
```python
from datetime import datetime
from pathlib import Path

import pandas as pd
from airflow import DAG
from airflow.operators.python import PythonOperator

from event_generator import get_events

dag = DAG(
    dag_id="01_unscheduled",
    start_date=datetime(2019, 1, 1),
    schedule_interval=None  # 현재는 스케쥴링 되지 않음F
)

fetch_events = PythonOperator(
    task_id="fetch_events",
    python_callable=get_events,
    dag=dag,
)


def _calculate_stats(input_path, output_path):
    """Calculates event statistics."""

    Path(output_path).parent.mkdir(exist_ok=True)

    events = pd.read_json(input_path)
    stats = events.groupby(["date", "user"]).size().reset_index()

    stats.to_csv(output_path, index=False)


calculate_stats = PythonOperator(
    task_id="calculate_stats",
    python_callable=_calculate_stats,
    op_kwargs={"input_path": "/opt/airflow/tmp/events.json",
               "output_path": "/opt/airflow/tmp/stats.csv"},
    dag=dag,
)

fetch_events >> calculate_stats
```

## DAG에 스케쥴 지정하기
```python
dag = DAG(
	dag_id='<Your Dag ID>',
	scheduled_interval='<schedule interval>',
	start_date=<start_date>,
	end_date=<end_date>,
	...
)
```
- `scheduled_interval` 외에도 `start_date`를 지정하여 DAG를 언제부터 시작할지 설정해야 한다.
- DAG는 `start_date`기준으로 `scheduled_interval`을 적용하여 DAG를 스케쥴링한다.
	- 예) 2024.08.13 1시 이후에 스케쥴을 다음과 같이 `start_date`를 2024.08.13 로 지정하고, `@daily` 간격으로 실행한다고 하면 DAG의 첫 실행 날짜는 2024.08.13 + 1 day 이므로 2024.08.14이다.
	- **따라서, `start_date`는 과거의 시간으로 설정되어야 원하는 날짜에 DAG가 실행되는 것을 보장할 수 있다. 미래의 시간으로 설정하면 DAG는 실행되지 않는다.**
- `end_date`가 지정되지 않으면 기본값이 `None`이기 때문에 스케쥴 간격대로 영원이 실행된다.

### Cron 기반 스케쥴링
![[Airflow - Scheduled by cron.png]]
- 스케쥴이 cron 형태로 작성된 날짜와 시간에 맞춰 DAG가 실행된다.
- `*`은 해당 위치의 값에 관계없다는 뜻
- 예시
	- 0 * * * * : 매시간
	- 0 0 * * * : 매일 자정
	- 45 23 * * SAT : 매주 토요일 23시 45분
	- 0 0 * * MON-FRI : 매주 월요일부터 금요일 자정에 실행
- crontab 관련 사이트: https://crontab.guru/

### 매크로 기반 스케쥴링
- Airflow 에서 cron preset이라는 사전에 지정된 문자열로 스케쥴링이 가능하다.
![[Airflow - Cron Preset.png]]
- 공식 문서: https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/cron.html#cron-presets

### 빈도 기반 스케쥴링
- Cron이나 매크로에서는 정해진 시간에만 실행하게 되므로 2, 3, 4일과 같은 간격은 표현할 수 없다.
- `datetime.timedelta` 으로 빈도 기반으로 스케쥴링을 할 수 있다.
```python
import datetime

dag=DAG(
	dag_id='<Your DAG ID>',
	schedule_interval=datetime.timedelta(days=3),
	start_date=...,
	end_date=...
)
```
https://docs.python.org/ko/3.8/library/datetime.html#timedelta-objects


## 데이터 증분 처리하기
### 데이터 증분 처리에 대한 니즈
1. 전체 데이터를 대상으로 처리하는 경우, 데이터의 크기가 큰 경우에는 효율적이지 못하다.
	- 필요한 만큼 데이터를 가져와 처리하게 되면 데이터의 양을 줄일 수 있어 더 효율적이다.
	- Ex) 전체 30일치 데이터를 가져오는 것보다 필요한 데이터(1일치)만 가져와서 처리하는 것

## 증분 구현 방법
### 임의로 구간 지정하기
```python
fetch_events = PythonOperator(
    task_id="fetch_events",
    python_callable=get_events,
    op_kwargs={'start_date': '2024-08-01', 'end_date': '2024-08-02'},
    dag=dag,
)
```
- `op_kwargs`와 같은 방법으로 가져올 데이터만 지정해줄 수 있다.

### Execution Date 활용하기
- `execution_date`: DAG가 스케쥴 간격으로 실행되는 시작 시간을 나타내는 타임 스탬프
- `next_execution_date`: 다음 DAG가 실행되는 시작 시간을 나타내는 타임 스탬프
- `previous_execution_date`: 이전 DAG가 실행된 시작 시간을 나타내는 타임 스탬프
- 2.2 버전 이후로는 `execution_date`가 deprecated되고 `logical_date`가 추가되었다.
	- `execution_date`은 실행된 시간으로, 만일 재시작 하면 날짜가 바뀌는 이슈가 있다.
	- `logical_date`: 스케쥴 간격에 따른 논리적인 DAG 실행 시간. 
	- `data_interval_start`: 처리할 데이터 간격의 시작 지점
	- `data_intererval_end`: 처리할 데이터 간격의 끝 지점
```python
from datetime import datetime
from pathlib import Path

import pandas as pd
from airflow import DAG
from airflow.operators.python import PythonOperator

from event_generator import get_events

dag = DAG(
    dag_id="02_incremental",
    start_date=datetime(2019, 1, 1),
    schedule_interval='@daily',
    catchup=False
)

fetch_events = PythonOperator(
    task_id="fetch_events",
    python_callable=get_events,
    op_kwargs={
        'start_date': "{{ data_interval_start.strftime('%Y-%m-%d') }}", # 데이터 윈도우 시작 지점
        'end_date': "{{ data_interval_end.strftime('%Y-%m-%d') }}", # 데이터 윈도우 종료 지점
    },
    dag=dag,
)


def _calculate_stats(input_path, output_path):
    """Calculates event statistics."""
    Path(output_path).parent.mkdir(exist_ok=True)

    events = pd.read_json(input_path)
    stats = events.groupby(["date", "user"]).size().reset_index()

    stats.to_csv(output_path, index=False)


calculate_stats = PythonOperator(
    task_id="calculate_stats",
    python_callable=_calculate_stats,
    op_kwargs={"input_path": "/opt/airflow/tmp/events.json",
               "output_path": "/opt/airflow/tmp/stats.csv"},
    dag=dag,
)

fetch_events >> calculate_stats
```
- Airflow의 Jinja 템플릿을 이용
https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html#variables


## 데이터 파티셔닝
- 위의 코드에서 `start_date, end_date`를 지정하여 특정 데이터만 증분 처리하도록 하였지만, 결과 값은 매번 같은 파일에 덮어쓰게 된다.
- 실행 날짜를 파일 이름에 기록하여 데이터 세트를 나눌 수 있다.
```python
from datetime import datetime
from pathlib import Path

import pandas as pd
from airflow import DAG
from airflow.operators.python import PythonOperator

from event_generator import get_events


dag = DAG(
    dag_id="02_incremental",
    start_date=datetime(2019, 1, 1),
    schedule_interval='@daily',
    catchup=False
)

fetch_events = PythonOperator(
    task_id="fetch_events",
    python_callable=get_events,
    op_kwargs={
        'start_date': "{{ data_interval_start.strftime('%Y-%m-%d') }}",
        'end_date': "{{ data_interval_end.strftime('%Y-%m-%d') }}",
        "output_file": "/opt/airflow/tmp/events_{{ data_interval_start.strftime('%Y-%m-%d') }}.json",
    },
    dag=dag,
)


def _calculate_stats(input_path, output_path):
    """Calculates event statistics."""
    Path(output_path).parent.mkdir(exist_ok=True)

    events = pd.read_json(input_path)
    stats = events.groupby(["date", "user"]).size().reset_index()

    stats.to_csv(output_path, index=False)


calculate_stats = PythonOperator(
    task_id="calculate_stats",
    python_callable=_calculate_stats,
    op_kwargs={"input_path": "/opt/airflow/tmp/events_{{ data_interval_start.strftime('%Y-%m-%d') }}.json",
               "output_path": "/opt/airflow/tmp/stats_{{ data_interval_start.strftime('%Y-%m-%d') }}.csv"},
    dag=dag,
)


# def _calculate_stats(**context):
#     """Calculates event statistics."""
#     input_path = context["templates_dict"]["input_path"]
#     output_path = context["templates_dict"]["output_path"]

#     events = pd.read_json(input_path)
#     stats = events.groupby(["date", "user"]).size().reset_index()

#     Path(output_path).parent.mkdir(exist_ok=True)
#     stats.to_csv(output_path, index=False)
    

# calculate_stats = PythonOperator(
#     task_id="calculate_stats",
#     python_callable=_calculate_stats,
#     templates_dict={
#         "input_path": "/opt/airflow/tmp/events_{{ data_interval_start.strftime('%Y-%m-%d') }}.json",
#         "output_path": "/opt/airflow/tmp/stats_{{ data_interval_start.strftime('%Y-%m-%d') }}.csv",
#     },
#     # Required in Airflow 1.10 to access templates_dict, deprecated in Airflow 2+.
#     # provide_context=True,
#     dag=dag,
# )


fetch_events >> calculate_stats

```
- 파티셔닝(partitioning): 데이터 세트를 더 작고 관리하기 쉬운 조각으로 나누는 작업
- 파티션(partition): 데이터 세트의 작은 부분
- templates_dict로 하는 방법도 있지만 2버전 이상부터 deprecated 되었다.

## Airflow 실행 날짜 이해
### 고정된 스케쥴 간격으로 태스크 실행
![[Airflow - Scheduled interval.png]]
- `start_date, end_date, scheduled_interval` 처럼 간격 기반의 시간 표현에서는 해당 간격의 시간 슬롯이 경과되자마자 해당 간격 동안 DAG가 실행된다.
	- 예를 들어, 2024.01.01이 `start_date`이고 주기가 daily라면 첫번째 실행은 2024.01.02 00시고, 이 때 처리하는 `data_interval_start`는 2024.01.01 이고 `data_interval_end`는 2024.01.02 이다. 참고로  이 때의 `logical_date`는 2024.01.01 이다.
- 기간 접근 방식은 스케쥴 간격을 알기 때문에 스케쥴 간격에 해당하는 데이터만 처리하는 증분 데이터 처리에 적합하다.

### 간격 기반 스케쥴과 시점 기반 스케쥴에서의 증분 처리
![[Airflow - 고정 기반 스케쥴링과 간격 기반 스케쥬릴ㅇ.png]]
- 간격 기반 스케쥴링은 각 간격에 대해 실행할 간격의 시작과 종료에 대한 정확한 정보를 제공
- 시점 기반 스케쥴링에서는 주어진 시간에만 작업을 실행하기 때문에 간격은 작업 자체에 달려있다.

## 백필
- 임의의 시작 날짜로부터 스케쥴 간격을 정의할 수 있기 때문에 과거의 시작 날짜부터 과거 간격을 정의할 수 있다. 이를 이용하여 과거 데이터 세트를 로드하거나 분석하기 위해 DAG의 과거 시점을 지정해 실행할 수 있다. 이를 백필(Backfilling) 이라고 한다.
![[Airflow - catchup.png]]
- 백필은 DAG 속성 중 `catchup`을 기반으로 제어하며, 기본값은 True이다.
	- configuration에서 `catchup_by_default` 값을 통해 기본값을 제어할 수 있다.
- `catchup`이 활성화 된 경우, 과거의 시작 날짜로부터 현재 시점까지 모든 스케쥴 간격에 대한 DAG를 실행한다.
	- 일반적으로는 시작 날짜에 따라 많은 데이터 세트가 로드될 수 있기 때문에 `catchup = False`로 지정한다.
- 백필은 코드를 수정한 후 데이터를 다시 처리하는데 사용될 수 있다.
	- 예를 들어, 특정 데이터의 칼럼을 추가로 가져오는 경우에는 과거 데이터부터 현재 데이터까지 칼럼 값을 채울 수 있다.
- 커맨드라인에서 Backfill 하기
	- airflow backfill <dag_id> -s <start_date> -e <end_date>
	- `start_date`로부터 시작하지만 `end_date`는 포함하지 않는다.
	- 백필의 실행순서는 랜덤으로 실행하기 때문에 날짜 순으로 하고 싶다면 DAG 안의 `depends_on_past=True`로 지정하면 된다.


## 원자성
- Airflowd에서 원자성은 태스크는 성공적으로 수행하여 적절한 결과를 생성하거나 시스템 영향을 미치지 않고 실패하도록 정의하는 것
	- 모든 것이 완료되거나 완료되지 않도록 보장해야 한다. 절반의 태스크가 진행되지 않아, 결과적으로 잘못된 결과를 발생시켜는 안 된다.
![[Airflow - 원자성.png]]

### 원자성 보장하지 못하는 예시
```python
def extract_function():
	...

def transform_function():
	...

def load_function(transformed_data):
	# autocommit = True
	for key, value in transform_data.items():
		load_data_to_database(value)

extract_task = PythonOperator(
	task_id='<task_id>',
	python_callable=extract_function,
)

transformed_task = PythonOperator(
	task_id='<task_id>',
	python_callable=transform_function,
)

load_task = PythonOperator(
	task_id='<task_id>',
	python_callable=load_function,
)

extract_task >> transformed_task >> load_task
```
- load_function에서 auto_commit이 True이라 가정하면 transformed_data가 순차적으로 돌면서 한줄씩 로드할 것이다. 만일 loop 중간에서 에러가 발생하면 일부만 데이터베이스에 저장되고 나머지 전체 데이터는 저장되지 않는다.
	- 전체적으로 실패했지만 일부는 성공해 원자성이 보장되지 않는다.

### 원자성 보장하는 예시
```python
def extract_function():
	...

def transform_function():
	...

def load_function(transformed_data):
	# autocommit = False
	try:
		for key, value in transform_data.items():
			load_data_to_database(value)
	except:
		rollback()

extract_task = PythonOperator(
	task_id='<task_id>',
	python_callable=extract_function,
)

transformed_task = PythonOperator(
	task_id='<task_id>',
	python_callable=transform_function,
)

load_task = PythonOperator(
	task_id='<task_id>',
	python_callable=load_function,
)

extract_task >> transformed_task >> load_task
```
- load_function에 예외 처리 구문을 추가하고 에러 발생시 rollback을 해줌으로써 원자성을 보장할 수 있다.


## 멱등성
- 동일한 입력으로 동일한 태스크를 몇 번이라도 호출하여도 결과가 변경되지 않아야 한다.
- 일반적으로 데이터를 쓰는 태스크는 기존 결과를 확인하거나 이전 태스크 결과를 덮어쓸지 여부를 확인하여 멱등성을 유지할 수 있다.
	- 예시) 파일로 저장하는 경우에는 logical_date로 파티션하여 멱등성을 보장할 수 있다. 데이터베이스에 upsert을 실행하여 멱등성을 보장할 수 있다.

### 멱등성이 보장 안 되는 구조
```python
def save_data(data):
	with open(file, 'a') as f:
		f.write(data)

save_task = PythonOperator(
	task_id='<task_id>',
	python_callable=save_data
)
```
- 같은 파일에 계속 append 로 데이터를 추가하므로 동일한 데이터로 실행시 같은 데이터가 파일에 계속해서 쌓일 것이다.

### 멱등성이 보장 되는 구조
```python
fetch_events = PythonOperator(
    task_id="fetch_events",
    python_callable=get_events,
    op_kwargs={
        'start_date': "{{ data_interval_start.strftime('%Y-%m-%d') }}",
        'end_date': "{{ data_interval_end.strftime('%Y-%m-%d') }}",
        "output_file": "/opt/airflow/tmp/events_{{ data_interval_start.strftime('%Y-%m-%d') }}.json",
    },
    dag=dag,
)


def _calculate_stats(input_path, output_path):
    """Calculates event statistics."""
    Path(output_path).parent.mkdir(exist_ok=True)

    events = pd.read_json(input_path)
    stats = events.groupby(["date", "user"]).size().reset_index()

    stats.to_csv(output_path, index=False)


calculate_stats = PythonOperator(
    task_id="calculate_stats",
    python_callable=_calculate_stats,
    op_kwargs={"input_path": "/opt/airflow/tmp/events_{{ data_interval_start.strftime('%Y-%m-%d') }}.json",
               "output_path": "/opt/airflow/tmp/stats_{{ data_interval_start.strftime('%Y-%m-%d') }}.csv"},
    dag=dag,
)
```
- 날짜별로 파일을 나눠서 데이터를 보관하므로 멱등성을 보장할 수 있다.