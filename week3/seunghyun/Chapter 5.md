---
Chapter: 태스크 간 의존성 정의하기
---
## 태스크 의존성 패턴
### 선형 의존성 패턴
> 선형 태스크 체인으로 구성된 DAG
- 예시) 로켓 사진 데이터 가져오기 파이프라인

#### 특징
1. 이전 태스크의 결과가 다음 태스크의 입력 값으로 사용되기 때문에, 다음 태스크로 넘어가기 전에 각 태스크를 완료해야 한다.
2. 오른쪽 비트 시프트 연산자(>>)를 사용하여 태스크 간에 의존성을 만들어 태스크 간의 관계를 나타낼 수 있다.
```python
donwload_files >> get_pictures
get_pictures >> notify

or

donwload_files >> get_pictures >> notify
``` 
3. 업스트림 의존성이 성공적으로 실행된 후에 지정된 다음 태스크 실행을 시작할 수 있다.
	- 예시) `download_files` 태스크 성공하기 전에는 `get_pictures` 태스크를 시작할 수 없다.
5. 태스크의 의존성을 명시적으로 지정함으로써 여러 태스크에서 순서가 명확해진다.
6. 모든 오류는 airflow에 의해 다운스트림 태스크로 전달되어 실행을 지연시킨다.
	- 예시) `download_files` 태스크가 실패한 경우, airflow는 `download_files` 태스크의 문제가 해결될 때까지 해당 날짜에 대한 `get_pictures` 태스크를 실행하지 않는다.

### 팬아웃(Fan-out)/팬인(Fan-in) 패턴
> 선형 체인 외의 복잡한 태스크 의존성 패턴
- 예시) 우산 판매 예측 모델

#### 특징
1. 팬아웃 종속성(한 태스크를 여러 다운스트림 태스크에 연결하는 것)으로 병렬로 독립된 태스크를 실행할 수 있다.
	- start 종속성이 종료된 후에 `fetch_weather, fetch_sales` 태스크가 실행된다.
```python
# 팬아웃(일 대 다) 의존성
start = DummyOperator(task_id='start') # 2.3 버전까지
start = EmptyOperator(task_id='start') # 2.4 버전 이후

start >> [fetch_weather, fetch_sales]
```
2. 팬인 종속성(여러 태스크를 하나의 다운스트림 태스크에 연결하는 것)으로 여러 업스트림의 태스크를 하나의 다운스트림 태스크로 의존성을 연결할 수 있다.
	- `clean_weather, clean_sales` 태스크들이 모두 종료된 후에야 `join_datasets` 태스크가 실행된다.
```python
# 팬인(다 대 일) 의존성
[clean_weather, clean_sales] >> join_datasets
```


## 브랜치
- 태스크의 변경으로 전체 파이프라인에 영향을 주어서는 안 되고, 태스크가 변경되기 전과 후의 모든 상황을 모두 커버할 수 있게 하기 위해서 사용하는 방법

### 태스크 내에서 브랜치
- 변경된 부분을 태스크의 조건문을 통해 반영하는 방법
	- `logicla_data`를 조건에 따라 나눠서 함수를 실행하게 하여서 전체 파이프라인에는 영향없이 해당 태스크만 수정하였고, 특정 날짜 이전의 데이터와 이후의 데이터 모두 다룰 수 있게 변경되었다.
```python
def _clean_data(**context):
	if context['logical_date'] < 'Some date':
		_clean_old_data(**context)
	else:
		_clean_new_data(**context)

...

clean_data = PythonOperator(
	task_id='clean_data',
	python_callable=_clean_data,
)
```
- DAG 자체 구조를 수정하지 않고도 DAG에서 약간의 유연성을 허용할 수 있다는 것
- 이 접근 방식은 코드로 분기가 가능한 유사한 태스크로 구성된 경우에만 작동한다.
	- 새로운 데이터에 대한 전처리가 필요하다고 하면, 별개의 수집과 전처리 태스크 세트로 분할해야할 것이다.
- DAG 실행 중에 airflow에서 어떤 코드 분기를 사용하고 있는지 확인하기 어렵다.
	- 태스크의 로깅을 더 세세하게 하는 방법이 있지만, DAG 자체에서 브랜치를 보다 명시적으로 만드는 방법이 있다.

### DAG 내부에서 브랜치하기
- 두 개의 개별 태스크 세트를 개발하고 DAG에서 선택적으로  태스크를 실행하는 방법
```python
from airflow.operators.python import BranchPythonOperator

def _clean_new_data():
	...

def _clean_old_data():
	...

def _pick_processing_function():
	...
	

pick_processing_function=BranchPythonOperator(
	task_id='pick_processing_function',
	python_callable=_pick_processing_function # 태스크 id
)
```
- `BranchPythonOperator`에 콜러블 인수로 태스크 ID 혹은 태스크 ID 리스트를 반환한다.
	- 태스크 ID 리스트인 경우에는 참조된 모든 태스크를 실행한다.

```python
from datetime import datetime

from airflow import DAG
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import BranchPythonOperator, PythonOperator


def _clean_old_data():
    print('Old!!')


def _clean_new_data():
    print('New!!')


def _pick_process(**context):
    if context['logical_date']:
        return 'clean_old_data'
    else:
        return 'clean_new_data'


def _next_old_data():
    print('next old')


def _next_new_data():
    print('next new')


def _join_data():
    print('join')


with DAG(
    dag_id='branching',
    schedule=None,
) as dag:

    clean_old_data = PythonOperator(
        task_id='clean_old_data',
        python_callable=_clean_old_data,
    )

    clean_new_data = PythonOperator(
        task_id='clean_new_data',
        python_callable=_clean_new_data,
    )

    next_old_data = PythonOperator(
        task_id='next_old_data',
        python_callable=_next_old_data
    )

    next_new_data = PythonOperator(
        task_id='next_new_data',
        python_callable=_next_new_data
    )

    join_data = PythonOperator(
        task_id='join_data',
        python_callable=_join_data
    )

    pick_process = BranchPythonOperator(
        task_id='pick_process',
        python_callable=_pick_process,
    )

    start_task = EmptyOperator(
        task_id='start_task'
    )

    start_task >> pick_process >> [clean_new_data, clean_old_data]
    clean_new_data >> next_new_data
    clean_old_data >> next_old_data
    [next_new_data, next_old_data] >> join_data
```
- 위의 케이스에서는 `clean_new_data` 혹은 `clean_old_data` 중 하나가 skip 되면 다음 의존성인 `join_data` 태스크를 건너뛰게 되어 원하는 흐름으로 동작하지 않는다.
	- `join_data`는 `next_new_data`와 `next_old_data` 두 업스트림 태스크에 대한 의존성이 있기 때문이다.
```python
join_data = PythonOperator(
	task_id='join_data',
	python_callable=_join_data,
	trigger_rule='none_failed',
)
```
- `trigger_rule`을 적용함으로써 규칙에 따라 태스크를 실행할 수 있다.
	- https://airflow.apache.org/docs/apache-airflow/2.9.3/core-concepts/dags.html#trigger-rules
- 브랜치를 독립적으로 유지하기 위해 태스크들을 결합하는 더미 태스크를 추가한다.
![[Airflow - Branching.png]]


## 조건부 태스크
> Airflow는 특정 조건에 따라 DAG에서 특정 태스크를 건너뛸 수 있는 다른 방법을 제공한다.
> 조건을 통해 특정 데이터 세트를 사용할 수 있을 때에만 실행하거나 최근에 실행된 DAG인 경우만 태스크를 실행할 수 있다.


### PythonOperator를 사용한 태스크 내 조건
```python
def _deploy(**context):
	if context['logical_date'] == ...:
		deploy_model()

deploy = PythonOperator(
	task_id='deploy',
	python_callable=_deploy,
)
```
- PythonOperator 이외의 다른 오퍼레이터를 사용할 수 없다.
- Airflow UI에서 태스크 결과를 추적할 때 혼란스러울 수 있다.

### 조건부 태스크 만들기
> 태스크 자체를 조건부화하여 미리 정의된 조건에 따라서만 실행하도록 할 수 있다. Airflow에서 해당 조건을 테스트하고 조건이 실패할 경우 모든 다운스트림 작업을 건너뛰는 태스크를 DAG에 추가하여 태스크를 조건부화할 수 있다.
```python
from airflow.exceptions import AirflowSkipException

def _deploy(**context):
	if <condition>:
		raise AirflowSkipException()
```
- `_deploy`에서 조건에 의해 `AirflowSkipException`이 발생하는 경우, 모든 다운스트림 태스크를 건너뛸 수 있다.
- 조건부 태스크가 조건에 의해 건너뛰게 되면, 그 다음스트림 태스크의 트리거 규칙을 살펴보고 트리거 수행 여부를 판단한다.
	- default 값인 all_success면 업스트림 태스크가 건너뛰어졌으므로, 다운스트림 태스크도 마찬가지로 건너뛰게 된다.

## 트리거 규칙에 대한 추가 정보
- 트리거 규칙을 이해하기 위해서는 먼저 airflow가 DAG 실행 내에서 태스크를 실행하는 방법을 알아야한다.
	1. DAG를 실행할 때 각 태스크를 지속적으로 확인하여 실행 여부를 확인한다.
	2. 태스크 실행이 가능하다고 판단되면, 그 즉시 스케쥴러에 의해 선택된 후 실행을 예약한다.
	3. airflow에 사용 가능한 워커가 있다면 즉시 태스크가 실행된다.
- 여기서 태스크 실행 시기는 트리거 규칙에 의해 결정된다.

### 트리거 규칙
- 태스크의 의존성과 같이 airflow가 태스크 실행 준비가 되어 있는지 여부를 결정하기 위한 필수적인 조건이다.
- 기본 트리거 규칙은 `all_success`이며, 태스크를 실행하려면 모든 업스트림 태스크가 모두 성공적으로 완료되어야 함을 의미한다.
```python
from airflow import DAG
from airflow.exceptions import AirflowException
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import PythonOperator


def t2(**context):
    raise AirflowException


with DAG(
    dag_id='trigger_rule',
    schedule=None,
) as dag:
    start = EmptyOperator(task_id='start')

    t1 = EmptyOperator(task_id='t1')

    t2 = PythonOperator(
        task_id='t2',
        python_callable=t2
    )

    t3 = EmptyOperator(task_id='t3')

    t4 = EmptyOperator(task_id='t4')

    start >> t1
    [t1, t2] >> t3 >> t4
```
- 위의 DAG에서 t2가 실패하므로 의존성에 따라 다운스트림인 t3와 t4는 업스트림 태스크가 실패했기 때문에 status가 `upstream_failed`로 표기된다.
	- 업스트림 태스크 결과가 다운스트림 태스크에도 영향을 미치는 동작 유형을 **전파(propagation)** 이라고 한다. 
- 만일 업스트림 결과가 skip이 되면, 다운스트림 태스크 모두 skip이 된다.
- 적절한 `trigger_rule`을 배치하여 파이프라인이 원하는대로 동작하도록 해야한다.

### 기타 트리거 규칙

| Trigger rule          | 동작                                                      | 사용 사례                                             |
| --------------------- | ------------------------------------------------------- | ------------------------------------------------- |
| all_success (default) | 모든 상위 태스크가 성공적으로 완료되면 트리거된다.                            | 일반적인 워크플로에 대한 기본 트리거 규칙                           |
| all_failed            | 모든 상위 태스크가 실패했거나 상위 태스크의 오류로 인해 실패했을 경우 트리거된다.          | 태스크 그룹에서 하나 이상 실패가 예상되는 상황에서 오류 처리 코드를 트리거한다.     |
| all_done              | 결과 상태에 관계없이 모든 상위 태스크가 실행을 완료하면 트리거된다.                  | 모든 태스크가 완료되었을 때 실행할 청소 코드를 실행한다.(시스템 종료나 클러스터 중지) |
| one_failed            | 하나 이상의 상위 태스크가 실패하자마자 트리거되며 다른 상위 태스크의 실행 완료를 기다리지 않는다. | 알림 또는 롤백과 같은 일부 오류 처리 코드를 빠르게 트리거한다.              |
| one_success           | 한 상위 태스크가 성공하자마자 트리거되며 다른 상위 태스크의 실행 완료를 기다리지 않는다.      | 하나의 결과를 사용할 수 있게 되는 즉시 다운스트림 연산/알림을 빠르게 트리거한다.    |
| none_failed           | 실패한 상위 태스크가 없지만, 태스크가 성공 혹은 건너뛴 경우 트리거된다.               | 조건부 브랜치의 결합                                       |
| none_skipped          | 건너뛴 상위 태스크가 없지만 태스크가 성공 또는 실패한 경우 트리거된다.                | 모든 업스트림 태스크가 실행된 경우, 해당 결과를 무시하고 트리거한다.           |
| always                | 업스트림 태스크의 상태와 관계없이 트리거된다.                               | 테스트 시                                             |
- `none_failed` 규칙은 모든 업스트림 태스크가 성곡 혹은 스킵을 모두 허용하기 때문에 브랜치를 결합하기에 적합하다.
- `all_done` 규칙은 결과에 관계없이 의존성 실행이 완료되는 즉시 실행되는 태스크를 정의할 수 있다. 리소스 정리나 인스턴스 종료와 같은 태스크를 실행하는데 사용할 수 있다.
- `one_failed` 혹은 `one_success` 와 같은 즉시 규칙은 트리거하기 전에 모든 업스트림 태스크가 완료될 때까지 기다리지 않고 하나의 업스트림 태스크의 성공/실패 확인 조건만 필요로 한다. 태스크의 조기 실패를 알리거나 태스크 그룹 중 하나의 태스크가 성공적으로 완료되는 즉시 대응할 수 있다.
- 트리거 규칙: https://airflow.apache.org/docs/apache-airflow/2.9.3/core-concepts/dags.html#trigger-rules

## 태스크 간 데이터 공유
> airflow의 XCom을 이용하여 태스크 간에 작은 데이터를 공유할 수 있다. XCom은 기본적으로 태스크 간에 메시지를 교환하여 특정 상태를 공유할 수 있게 한다.


### Xcom 사용 방법
```python
from airflow import DAG
from airflow.operators.python import PythonOperator


def _create_api_key(**context):
    api_key = hash('My API Key')
    context['task_instance'].xcom_push(key='api_key', value=api_key) # Xcom 저장


def _print_api_key(**context):
    api_key = context['task_instance'].xcom_pull(
        task_ids='create_api_key', key='api_key') # Xcom 호출
    print(api_key)


with DAG(
    dag_id='xcom',
    schedule=None,
) as dag:

    create_api_key = PythonOperator(
        task_id='create_api_key',
        python_callable=_create_api_key,
    )

    print_api_key = PythonOperator(
        task_id='print_api_key',
        python_callable=_print_api_key
    )

    create_api_key >> print_api_key
```
- airflow 컨텍스트의 task_instance의 xcom_push 메소드를 이용하여 키-값 형태로 저장할 수 있다.
- airflow 컨택스트의 task_instance의 xcom_pull 메소드를 이용하여 다른 태스크에서 XCom 값을 확인할 수 있다.
	- Xcom 값을 가져올 때 dag_id 및 실행 날짜를 정의할 수 있다. 기본 실행 날짜는 현재 DAG와 실행 날짜로 설정된다.
![[Airflow - XComs.png]]
- Airflow 웹 서버에서 Admin > XComs를 통해 XCom을 확인할 수 있다.

#### Templates에서 XCom 사용하기
```python
def _print_api_key_by_templates_dict(templates_dict, **context):
    api_key = templates_dict['api_key']
    print(api_key)

print_api_key_by_templates_dict = PythonOperator(
	task_id='print_api_key_by_templates_dict',
	python_callable=_print_api_key_by_templates_dict,
	templates_dict={
		'api_key': "{{ task_instance.xcom_pull(task_ids='create_api_key', key='api_key') }}"
	}
)
```

#### XCom 값 자동으로 설정하기
- `BashOperator`의 경우, `xcom_push` 값을 True로 하면 오퍼레이터에서 배시 명령에 의해 stdout에 기록된 마지막 행을 XCom 값으로 설정한다.
- `PythonOperator`의 경우, `python_callable` 인수에서 반환된 값을 XCom 값으로 설정합니다.
- 기본 키는 return_value로 등록된다.

### XCom 사용시 주의사항
#### XCom 사용 단점
1. 풀링 태스크는 필요한 값을 사용하기 위해 태스크 간에 묵시적인 의존성이 필요하다. 명시적 의존 태스크와 달리 DAG에 표시되지 않으며 태스크 스케쥴 시에 고려되지 않는다. 따라서 XCom에 의해 의존성이 있는 작업이 올바른 순서대로 실행할 수 있도록 해야 한다.
2. 숨겨진 의존성은 서로 다른 DAG에서 실행 날짜 사이에 XCom 값을 공유할 때 훨씬 복잡해지기 때문에 권장하지 않는다.
3. 오퍼레이터의 원자성을 무너뜨리는 패턴이 될 가능성이 있다.
	- 오퍼레이터를 통해 API 액세스 토큰을 가져온 후에 다음 태스크에서 XCom을 이용해 전달하려 하는 경우, 해당 토근 사용 시간이 만료되어 다음 태스크를 재실행하지 못할 수 있다. API 토큰을 새로 고침하는 작업 수행을 다음 태스크에서 수행하는 것이 더 안전할 수 있다.
4. XCom에 저장되는 모든 값은 직렬화를 지원해야 한다는 기술적 한계가 존재한다.
	- 람다 혹은 여러 다중 멀티프로세스 관련 클래스 같은 일부 파이썬 유형은 XCom에 저장할 수 없다.
5. 사용되는 백엔드에 의해 XCom 값의 크기가 제한될 수 있다. 기본적으로 XCom은 Airflow 메타스토어에 저당되며 다음과 같이 크기가 제한된다.
	- SQLite: Blob 유형으로 저장 / 2GB 제한
	- PostgreSQL: BYTEA 유형으로 저장 / 1GB 제한
	- MySQL: Blob 유형으로 저장 / 64KB 제한
사용 시에 후에 발생할 수 있는 오류를 방지하기 위해 태스크 간의 의존성을 명확하게 기록하고 사용법을 신중히 검토해야 한다.


## 커스텀 XCom 백엔드 사용하기
- 메타스토어를 사용하여 XCom을 저장 시에 제한 사항은 큰 데이터 볼륨을 저장할 때 확장할 수 없다는 것이다.
- 작은 값이나 결과값을 저장할 때는 XCom이 유용하지만 큰 데이터 세트를 저장할 때는 사용되지 않는다.
- 2 버전부터 XCom을 유연하게 활용하기 위해 커스텀 XCom 백엔드를 저장할 수 있는 옵션이 추가되었다.
	- 커스텀 클래스를 정의하여 XCom을 저장 및 검색할 수 있다.
	- 다만, 커스텀 클래스 사용을 위해서는 BaseXCom 기본 클래스가 상속되어야 하고, 값을 직렬화 및 역직렬화하기 위해 두 가지 정적 메서드를 각각 구현해야 한다.
```python
from typing import Any
from airflow.models.xcom import BaseXCom

class CustomXComBackend(BaseXCom):
	@staticmethod
	def serialize_value(value: Any):
		...
	@staticmethod
	def deserialize_value(result) -> Any:
		...
```
- 커스텀 백엔드 클래스에서 직렬화 메소드는 XCom 값이 오퍼레이터 내에서 게시될 때마다 호출되는 반면 역직렬화 메소드는 XCom 값이 백엔드에서 가져올 때마다 호출된다.
- 원하는 백엔드 클래스가 있으면 airflow 구성에서 `xcom_backend`를 사용해 클래스를 사용하도록 airflow를 구성할 수 있다.
- 커스텀 XCom 백엔드는 XCom 값 저장 선택을 다양하게 만든다.
	- 상대적으로 저렴하고 확장 가능한 클라우드 스토리지에 더 큰 XCom 값이 저장이 가능해진다. AWS S3나 Google의 GCS와 같은 서비를 위한 커스텀 백엔드를 구현할 수 있다.


## Taskflow API로 파이썬 태스크 연결하기
> airflow 2 버전부터 Taskflow API를 통해 파이썬 태스크 및 의존성을 정의하기 위한 새로운 데코레이터 기반 API를 추가적으로 제공한다.


### Taskflow API로 파이썬 태스크 단순화하기
```python
def _train_model(**context):
    model_id = str(uuid.uuid4())
    context["task_instance"].xcom_push(key="model_id", value=model_id)


def _deploy_model(**context):
    model_id = context["task_instance"].xcom_pull(
        task_ids="train_model", key="model_id"
    )
    print(f"Deploying model {model_id}")


with DAG(
    dag_id="10_xcoms",
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval="@daily",
) as dag:
	...

    train_model = PythonOperator(task_id="train_model", python_callable=_train_model)

    deploy_model = PythonOperator(task_id="deploy_model", python_callable=_deploy_model)

    ...
    join_datasets >> train_model >> deploy_model
```
- 위의 접근 방식의 문제는 먼저 함수를 정의한 후, `PythonOperator`를 이용해 airflow 태스크를 생성해야 한다는 것이다.
- 두 태스크 간에 모델 ID를 공유하기 위해 함수 내에서 `xcom_push`, `xcom_pull`을 명시적으로 사용하여 모델 ID 값을 전송 및 반환해야 한다.  데이터 의존성을 정의하는 것은 번거롭고 두 태스크에서 참조되는 공유된 키 값을 변경하면 중단될 수 있는 문제가 있다.

```python
from airflow import DAG
from airflow.decorators import task


with DAG(
    dag_id="12_taskflow",
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval="@daily",
) as dag:

    @task
    def train_model(): # train_model이라는 함수를 @task 데코레이터가 래핑하여 태스크를 정의한다.
        model_id = str(uuid.uuid4())
        return model_id

    @task
    def deploy_model(model_id: str):
        print(f"Deploying model {model_id}")

	# model_id 값을 XCom 사용 없이 명시적으로 다음 태스크로 전달할 수 있다.
    model_id = train_model()
    deploy_model(model_id)
```
- Taskflow API는 파이썬 함수를 태스크로 쉽게 변환하고, DAG 정의에서 태스크 간에 데이터 공유를 명확하게 함으로써, 이러한 유형의 태스크에 대한 정의를 단순화하는 것을 목표로 한다.
- Taskflow API로부터 추가된 새로운 `@task`라는 데코레이터로 태스크 정의를 단순한 함수로 변환할 수 있다.
- 기존 함수를 개발하고 PythonOperator로 연결하는 것보다 Taskflow 기반 접근 방식이 가독성이 좋고 일반적인 파이썬 코드와 유사한 결과를 제공한다.

### Taskflow API 작동 방식
```python
from airflow import DAG
from airflow.decorators import task


with DAG(
    dag_id="12_taskflow",
    start_date=airflow.utils.dates.days_ago(3),
    schedule_interval="@daily",
) as dag:

    @task
    def train_model(): # train_model이라는 함수를 @task 데코레이터가 래핑하여 태스크를 정의한다.
        model_id = str(uuid.uuid4())
        return model_id

    @task
    def deploy_model(model_id: str):
        print(f"Deploying model {model_id}")

	# model_id 값을 XCom 사용 없이 명시적으로 다음 태스크로 전달할 수 있다.
    model_id = train_model()
    deploy_model(model_id)
```
1. train_model 호출시, train_model 태스크를 위한 새로운 오퍼레이터 인스턴스를 생성한다.
2. train_model 함수가 동작하고 return 문에서 airflow는 태스크에서 반환된 XCom으로 자동으로 값을 설정한다.
3. deploy_model 태스크의 경우, 데코레이터된 함수를 호출하여 오퍼레이터 인스턴스를 생성뿐만 아니라 train_model의 반환값인 model_id를 전달한다.

### Taskflow API를 사용하지 않는 이유
- Taskflow API는 객체 지향 오퍼레이터 API보다 좀 더 일반적인 파이썬 함수 사용법에 가까운 구문을 사용하여 파이썬 태스크와 태스크 간 의존성을 좀 더 간단하게 구현할 수 있도록 제공한다.
- XCom을 사용해 태스크 간에 작업 결과 데이터를 전달하기 위해 PythonOperator를 많이 사용하는 DAG를 크게 간소화한다.
- XCom의 문제점인 태스크 간 의존성을 확인하지 못하는 문제를 Taskflow API에서는 해당 함수 내의 태스크 간의 의존성을 숨기지 않고 태스크 간의 값을 명시적으로 전달함으로써 전달할 수 있다.
- Taskflow API의 문제는 PythonOperator를 사용하여 구현되는 파이썬 태스크로만 제한되는 것이 문제이다.
	- 다른 오퍼레이터와 관련된 태스크는 일반 API를 사용하여 태스크 및 태스크 의존성을 정의해야 한다.
	- TaskAPI 와 일반 API 두 가지 스타일을 혼용해서 사용하는데는 문제가 없지만 완성 코드가 복잡해보일 수 있다.