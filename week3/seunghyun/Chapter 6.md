---
Chapter: 워크플로 트리거
---
## 센서를 사용한 폴링 조건
- 워크플로를 시작하는 일반적인 사례는 새로운 데이터가 도착하는 경우이다.
-  센서는 특정 조건이 true인지 지속적으로 확인하고 true라면 성공한다. 만약에 false인 경우, 센서는 상태가 true가 될 때까지 혹은 타임아웃이 될 때까지 계속 확인한다.
- 워크플로에 센서를 통합하는 경우, 특정 시간까지 기다리지 않고 모든 데이터가 사용 가능하다고 가정하고, 데이터가 사용 가능한지 지속적으로 확인하게 된다. 때문에 워크플로 DAG 스케쥴 시간을 데이터가 도착하는 경계의 시작 부분에 배치한다.
- DAG가 시작되고 센서의 조건이 만족하면, 다음 태스크를 계속 수행하여 파이프라인을 수행한다.

### FileSensor 예시
```python
from airflow.sensors.filesystem import FileSensor

wait_for_file = FileSensor(
    task_id='wait_for_file',
    filepath='<file path>'
)
```
- `FileSensor`는 `filepath`에 지정된 경로에 파일이 존재하는지 확인하고 파일이 있으면 true를 반환하고, 그렇지 않으면 false를 반환한 후 해당 센서는 지정된 시간(기본 60초) 동안 대기했다가 다시 시도한다.
- 센서와 DAG는 모두 타임아웃의 설정이 가능하며, 센서 오퍼레이터는 제한 시간에 도달할 때까지 상태를 확인하며 대기한다.
- `FileSensor`는 `poke_interval`(기본 1분)에 한 번씩 센서는 주어진 파일이 있는지 **포크(poke)** 한다. 
- Poking은 센서를 실행하고 센서 상태를 확인하기 위해 airflow에서 사용하는 이름이다.

### 사용자 지정 조건 폴링
- 일부 큰 데이터 세트는 여러 파일로 구성될 수 있다. 여기서 `FileSensor`의 `filepath`에 `*.parquet` 와 같은 와일드 카드를 사용하여 여러 파일을 처리할 때 주의할 점이 있다.
	- 의도한 모든 파일이 전부 업로드 되지 않고 일부만 업로드가 되어도 Sensor는 True를 반환하기 때문에 원치 않은 파이프라인 결과를 얻을 수 있다.
	- 이를 위해 Fi`leSensor`는 글로빙(globbing)을 사용해 파일 혹은 디렉토리 이름과 패턴을 일치시키거나 `PythonSensor`로 가독성을 높이는 방법이 있다.
- `PythonSensor`는 `PythonOperator`와 유사하게 파이썬 콜러블을 지원한다. 파이썬 콜러블은 조건이 충족되면 true를, 실패한 경우 false로 반환한다.

### 원활하지 않은 흐름의 센서 처리
- 센서는 최대 실행 허용 시간(초)를 지정하는 timeout 인수를 허용한다. 다음 포크의 시작시 실행 시간이 타임아웃 설정값보다 초과되면 센서는 실패를 반환한다.
- 센서의 타임아웃 기본 값은 7일이다. 만일 데일리로 실행되는 DAG가 있고 센서가 false를 지속적으로 반환하고 있다면 매일마다 태스크는 의도치 않게 쌓일 것이다.
- airflow는 DAG의 `concurrency` 값을 통해 실행 태스크 수를 제한할 수 있다.
- Sensor Deadlock: DAG가 실행할 수 있는 최대 태스크 수에 도달해 차단된 상태
	- airflow 전역으로 설정한 최대 태스크 제한에 도달하면 전체 시스템이 정지될 수 있다.
- Sensor 클래스에는 `poke` 혹은 `reschedule`을 설정할 수 있는 `mode` 인수가 있다. 기
	- 기본적으로 `poke`로 설정되어 있어 최대 태스크 제한에 도달하면 새로운 태스크가 차단된다. 즉, 센서 태스크가 실행중인 동안 태스크 슬롯을 차지하게 된다. 설정한 대기 시간보다 포크를 수행한 후 아무 동작도 하지 않지만 여전히 태스크 슬롯을 차지하게 된다.
	- `reschedule` 모드는 포크 동작을 실행할 때만 슬롯을 차지하며, 대기 시간 동안은 슬롯을 차지 않는다.
![[Airflow - Sensor mode.png]]
- 동시 태스크의 수는 airflow 전역 설정 옵션으로 제어할 수 있다.

### 다른 DAG를 트리거하기
![[Airflow - 슈퍼마켓 예시.png]]
- create_metrics 가 실행은 process_supermarket_* 태스크의 성공 여부에 달려있다.
- create_metrics 태스크를 기다리지 않고 각 process_supermarket_* 태스크가 실행 후에 create_metrics_* 처럼 태스크를 나눌 수 있다. 이러면 하나의 태스크가 여러 태스크로 확장되어 DAG 구조가 복잡해지고 결과적으로 더 많은 반복 태스크가 발생한다.
- 비슷한 기능의 태스크 반복을 피하는 한 가지 옵션은 각 DAG를 여러 개의 작은 DAG로 분할하여 각 DAG가 일부 워크플로를 처리하는 방법이 있다.
	- 단일 DAG에서 여러 태스크를 보유하지 않고 하나의 DAG에서 다른 DAG를 여러 번 호출할 수 있다.
	- 이 방법은 워크플로의 복잡성과 같은 다양한 요소에 따라 달라진다.
	- 워크플로가 스케쥴에 따라 완료될 때까지 기다리지 않고 언제든지 수동으로 트리거할 수 있는 통계 지표를 생성하기 위해서는 DAG를 두 개로 분할하는 것이 좋다.

```python
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

dag1 = DAG(
	dag_id='dag1',
	...
)

for i in range(5):
	# ...
	trigger_dag = TriggerDagRunOperator(
		task_id=f'trigger_dag_{i}',
		trigger_dag_id='dag2', # 호출할 DAG id와 일치시킨다.
		dag=dag1
	)

dag2 = DAG(
	dag_id='dag2',
	schedule_interval=None # 트리거되는 DAG에는 schedule_interval이 필요없다.
)
```
- airflow 웹 서버의 트리 뷰의를 통해 스케쥴에 따라 트리거가 되었는지 아닌지 확인할 수 있다.
- DAG 실행은 `run_id` 필드가 있으며, 다음 중 하나로 실행된다.
	- schedule__: 스케쥴되어 DAG 실행이 시작되었음을 나타냄
	- backfill__: 백필 태스크에 의해 DAG 실행이 시작되었음을 나타냄
	- manual__: 수동으로 DAG 실행이 시작되었음을 나타낸다.(수동 DAG 실행이나 TriggerDagRunOperator에 의해 실행되는 예시)

#### TriggerDagRunOperator로 백필 작업
- 어느 태스크의 일부를 변경하고 변경된 부분부터 DAG를 다시 실행하려면, 단일 DAG에서는 해당 태스크 및 다운스트림 태스크 상태를 삭제하면 된다.
- 태스크 삭제는 동일한 DAG 안의 태스크만 지워진다. 또 다른 DAG 안에서 `TriggerDagRunOperator`의 다운스트림 태스크는 지워지지 않는다.
- `TriggerDagRunOperator`를 포함한 DAG에서 태스크를 삭제하면 이전에 트리거된 해당 DAG의 실행을 지우는 대신에 새 DAG 실행이 트리거 된다.

#### 다른 DAG의 상태를 폴링하기
- DAG가 아주 복잡해지는 경우 태스크를 명확하게 하기 위해 첫번째 DAG를 여러 개의 DAG로 분할하고, 각각의 해당 DAG에 대해 `TriggerDagRunOperator` 태스크를 수행할 수 있다.
- 여러 다운스트림 DAG를 트리거하는 하나의 DAG의 TriggerDagRunOperator를 사용할 수 있다.
![[Airflow - TriggerDagRunOperator 사용 예시.png]]
- 만일 다른 DAG가 실행되기 전에 여러 개의 트리거 DAG가 완료되어야 한다면 어떻게 해야할까요?
	- airflow는 단일 DAG 내에서 태스크 간의 의존성을 관리하지만, DAG간의 의존성을 관리하는 방법은 제공하지 않는다.
	- 다른 DAG에서 태스크 상태를 포크하는 센서인 `ExternalTaskSensor`를 적용한다. 이전의 다른 DAG가 모두 완료된 상태를 확인하는 프록시 역할을 한다.
	- `ExternalTaskSensor`은 다른 DAG의 태스크를 지정하여 해당 태스크의 상태를 확인하는 것으로 동작한다.
![[Airflow - ExternalTaskSensor.png]]
- DAG1에서 DAG2까지 어떠한 이벤트도 없기 때문에 DAG2는 DAG1의 태스크 상태를 확인할 때 몇 가지 단점이 존재한다.
	- airflow는 DAG가 다른 DAG를 고려하지 않는다. 기술적으로 기본 메타데이터를 쿼리하거나 디스크에서 DAG 스크립트를 읽어서 다른 워크플로의 실행 세부 정보를 추론할 수 있지만, airflow와는 직접적으로 결합되지 않는다.
- `ExternalTaskSensor`를 사용하는 경우, DAG를 정렬해야 한다. `ExternalTaskSensor`가 자신과 정확히 동일한 실행 날짜를 가진 태스크에 대한 성공만 확인하는 것이다. 
	- 스케쥴 간격이 맞지 않는 경우에는 `ExternalTaskSensor`가 다른 태스크를 검색할 수 있도록 offset을 설정할 수 있다.
	- 오프셋은 execution_delta 인수로 설정한다.
https://airflow.apache.org/docs/apache-airflow/2.9.3/_api/airflow/sensors/external_task/index.html#airflow.sensors.external_task.ExternalTaskSensor


## REST/CLI를 이용해 워크플로 시작하기
- DAG에서 다른 DAG를 트리거하는 방법 외에도 REST API 및 CLI 를 통해 트리거할 수 있다.
	- CI/CD 파이프라인의 일부로 airflow 외부에서 워크플로를 시작하려는 경우가 이에 해당된다.
	- AWS S3 버킷에 임의 시간에 저장되는 데이터를 확인하기 위해 airflow 센서를 통해 확인하는 대신, AWS Lambda 함수를 통해 DAG를 트리거할 수 있다.

`airflow dags trigger <dag id>`
- 실행 날짜가 현재 날짜 및 시간으로 설정된 DAG를 실행한다.
- dag_id에는 `manual__` 이라는 접두사가 붙게 된다.

`airflow dags trigger -c '{"key": value}' <dag id>`
- 태스크 컨텍스트 변수를 통해 트리거된 실행 DAG의 모든 태스크에서 사용할 수 있다.
- 단순하게 여러 변수를 적용하기 위해 태스크를 복제해 DAG를 구성할 경우, 파이프라인에 변수를 삽입할 수 있기 때문에 DAG 실행에 대한 구성이 간결해진다.

```bash
# URL is /api/v1

curl \
-u admin:admin \
-X POST \
"http://localhost:8080/api/v1/dags/<dag_id>/dagRuns" \
-H 'Content-Type: application/json' \
-d {'conf': {}}
```