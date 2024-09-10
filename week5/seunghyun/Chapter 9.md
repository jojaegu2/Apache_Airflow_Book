---
Chapter: 테스트하기
---
## 테스트
- 실제 서비스에 배포되기 전에 정상 동작 한다는 것을 확신하기 위해 **테스트**를 진행한다.
- 테스트 없이 배포를 하게 된다면, 코드는 배포 과정에서 매번 수정이 필요하기 때문에 코드의 품질과 신뢰도를 떨어뜨리게 된다.

### Airflow 테스트
- Airflow는 외부 시스템과 통신이 발생하는 특성과 오케스트레이션 시스템의 특성도 가지고 있다.
	- 로직을 실제로 수행하는 태스크를 실행하고 종료시키는 역할을 하지만, airflow 자신은 어떤 로직도 수행하지 않는다.
- 정상적으로 동작하지 않은 airflow 로직을 어떻게 확인할 수 있는지, 어떻게 수정 및 대응해야 하는지 알아볼 필요가 있다.

## 테스트 시작하기
- 테스트는 다양한 단계에서 적용할 수 있다.
- 개별태스크 단위는 단위 테스트로 테스트할 수 있다.
- 테스트는 올바른 동작을 검증할 수 있지만, 여러 단위로 구성된 시스템의 동작을 모두 검증하지 않는다. 이를 위해 여러 구성 요소의 동작을 함께 검증하는 통합 테스트를 작성한다.
- 개별 테스트 -> 통합 테스트 -> 승인 테스트(비즈니스 요구사항에 대한 적합성 평가) 순으로 진행된다.

- 파이썬에서 테스트를 위한 프레임워크로 **pytest**, **unittest** 가 제공된다.
- Github에서 지원하는 CI/CD 시스템인 **GitHub Action**을 이용해 테스트 과정에 대해 연습해볼 수 있다.
	- Github 의외에도 GitLab, Bitbucket, CircleCI, Travis CI 등과 같이 널리 사용되는 모든 CI/CD 시스템은 프로젝트 디렉토리의 루트에 YAML 파일 형식으로 파이프라인을 정의하여 작동한다.

### 모든 DAG에 대한 무결성 테스트
- airflow 관점에서 테스트를 위한 첫 번째 단계는 **DAG 무결성 테스트(Integrity Test)** 이다.
	- 모든 DAG의 무결성(DAG에 사이클이 포함되어 있는지, DAG의 태스크 ID가 고유한지 등)에 대해 DAG가 정상적으로 구현되었는지를 확인한다.
	- DAG를 호출할 때 airflow는 자체적으로 DAG 무결성 테스트를 수행한다.
	- 배포하지 않고 DAG에 간단한 실수가 있는지 확인하려면 테스트 도구 모음에서 DAG 무결성 테스트를 수행하는 것이 좋다.
- 일반적으로 테스트 영역을 구성하는 방법은 프로젝트의 최상단 디렉토리에 별도의 `tests/` 디렉토리를 생성하여 검사 대상 코드를 그대로 복사하여 구성한다.
	- tests 디렉토리 추가 전
		- ![[Airflow - dir.png|300]]
	- tests 디렉토리  추가 후
		  ![[airflow - dir after add tests.png|300]]
	- 모든 테스트 파일은 파일 이름은 그대로 따르고 `tests` 접두사를 붙인다. 테스트할 파일의 이름이 반드시 동일할 필요는 없지만, 해당 파일이 무슨 일을 하는지 직관적으로 파악하기 위해 일반적으로 `test`라는 접두사를 붙인다.
	- 여러 파일들을 한꺼번에 테스트하거나 여러 종류의 테스트를 진행할 때에는, 일반적으로 테스트 대상에 따라 이름이 지정된 파일에 배치된다.
	- tests 디렉토리는 모듈로 동작하는 구조가 아니기 때문에 테스트 작업은 서로 의존적이지 않으며 독립적으로 실행할 수 있어야 한다.
		- Pytest는 디렉토리와 파일을 스캔하고 테스트를 자동으로 검색하기 때문에 `__init__.py` 파일로 모듈을 만들 필요가 없다.
> [!info] Pytest 테스트 파일 이름
> `Pytest`는 주어진 디렉토리를 스캔하고 `test_` 접두사 혹은 `_test` 접미사 파일을 검색한다.

**무결성 테스트 코드**
```python
import glob
import importlib.util
import os


import pytest
from airflow.models import DAG

# DAG가 위치한 경로와 파일들을 탐색
DAG_PATH = os.path.join(os.path.dirname(__file__), '..', '..', 'dags/**/*.py')
DAG_FILES = glob.glob(DAG_PATH, recursive=True)


@pytest.mark.parametrize('dag_file', DAG_FILES)  # 찾은 모든 DAG 파일에 대해 테스트를 실행
def test_dag_integrity(dag_file):
    # 파일 로드하기
    module_name, _ = os.path.splitext(dag_file)
    module_path = os.path.join(DAG_PATH, dag_file)
    mod_spec = importlib.util.spec_from_file_location(module_name, module_path)
    module = importlib.util.module_from_spec(mod_spec)
    mod_spec.loader.exec_module(module)

    # 파일에서 발견된 모든 DAG 클래스의 객체
    dag_objects = [
        var for var in vars(module).values() if isinstance(var, DAG)
    ]

    # DAG 객체를 성공적으로 찾았는지 확인
    # assertion을 추가하여 /dags 경로에 있는 모든 파이썬 파일에 DAG 객체가 적어도 하나 이상 포함되어 있는지에 대한 유효성 검사를 진행
    assert dag_objects

    # DAG 객체에 순환 주기 존재 여부 확인
    # 1.10 버전 이전에는 DAG의 구조가 변경될 때마다 순환 주기의 문제가 없는지 확인했다. 이 작업은 태스크가 추가될 때마다 부하가 생긴다.
    # 태스크가 많은 DAG의 경우, 변경 시점마다 DAG를 검사하면 부하가 발생한다. 이런 이유로 DAG 주기 검사는 전체 DAG를 파싱한 후 한번만 수행되도록, 구문 분석이 끝나고 캐싱되는 시점으로 이동되었다.
    for dag in dag_objects:
        dag.test_cycle()

    '''
    DAG 이름의 접두사와 접미사를 가지고 테스트할 수 있다.
    for dag in dag_objects:
        assert dag.dag_id.startwith(('접두사'))
    '''

# pytest tests/
```


### CI/CD 파이프라인 설정하기
- CI/CD 파이프라인은 코드 저장소를 통해 코드가 변경할 때 사전 정의된 스크립트를 실행하는 시스템
- **지속적인 통합(Continuous Integration)**은 변경된 코드가 코딩 표준과 테스트 조건을 준수하는지 확인하고 검증하는 것을 말한다.
	- 파이썬 코드를 푸시할 때, Flake8, Pylint, Black과 같은 코드 검사기를 통해 코드의 품질을 확인하며 일련의 테스트를 실행한다.
- **지속적 배포(Continuous Deployment)**은 사람의 간섭 없이 완전히 자동화된 코드를 프로덕션 시스템에 자동으로 배포하는 것을 말한다.
- **CI/CD의 목표는 수동으로 검증하고 배포하지 않고도 코딩 생산성을 극대화하는 것**이다.
- GitHub Action을 통해 GitHub에 CI/CD 파이프라인을 지정할 수 있다.
- 파이프라인 전체가 성공적으로 동작하기 위해서는 각 단계가 성공적으로 완료 되어야한다. 그 다음 Git 저장소에 '성공한 파이프라인만 마스터에 병합(only merge to master with a successful pipeline)'과 같은 규칙을 적용할 수 있다.

**.github/workflows/airflow-tests.yaml** 예시
```yaml
name: python static checks and tests

# 완전 자동화된 CD 시스템에서는 마스터와 같은 특정 브랜치의 단계를 위한 필터가 포함되어 있어서 해당 브랜치 코드의 파이프라인이 성공하는 경우에만 단계를 실행하고 코드를 배포한다.
on: [push] # GitHub에 push를 할 때마다 전체 CI/CD 파이프라인을 실행하도록 지시

jobs:
  testing:
    runs-on: ubuntu-18.04
    steps: # GitHub Action에서 수행하는 일
      # 단계의 코드 조각
      - uses: actions/checkout@v1
      - name: Setup Python
        uses: actions/setup-python@v1
        with:
          python-version: 3.9
          architecture: x64

      - name: Install Flake8
        run: pip install Flake8
      - name: Run Flake8
        run: Flake8

      - name: Install Pylint
        run: pip install pylint
      - name: Run Pylint
        run: find . -name '*.py' | xargs pylint --output-format=colorized

      - name: Install Black
        run: pip install Black
      - name: Run Black
        run: find . -name '*.py' | xargs black --check


      - name: Install dependencies
        run: pip install apache-airflow pytest

      - name: Test DAG integrity
        run: pytest tests/
```

### 단위 테스트 작성하기
- 8장의 `get_ratings()` 메소드를 가진 `MovielensHook`에서 메소드는 여러 인수중에 `batch_size`를 입력받고 인수 값으로 API에서 요청한 배치의 크기를 제어한다. `batch_size`에서 유효한 값은 양수이고, 만일 음수를 전달하면 API는 잘못된 배치 크기를 처리하고 400 혹은 422와 같은 HTTP 오류를 반환할 것이다.
	- 이러한 문제를 일으키지 않게 하려면 `batch_size`를 전달하기 전에 해당 값이 유효한지 먼저 판단하는 테스트를 진행하면 해결할 수 있다.
- 8장의 예시를 이어서, 주어진 두 날짜 사이에 상위 N개의 인기 영화를 반환하는 `MovielensPopularityOperator`를 구현
movielens_operator.py
```python
class MovielensPopularityOperator(BaseOperator):
	def __init__(self, conn_id, start_date, end_date, min_ratings=4, top_n=5, **kwargs):
		super().__init__(**kwargs)
		self._conn_id = conn_id
		self._start_date = start_date
		self._end_date = end_date
		self._min_ratings = min_ratings
		self._top_n = top_n

	def execute(self, context):
		with MovielensHook(self._conn_id) as hook:
			ratings = hook.get_ratings(start_date=self._start_date, end_date=self._end_date) # 각 줄에서 점수 확인

			rating_sums = defaultdict(Counter)
			for rating in ratings: # movieId 마다 평점을 합산
				rating_sums[rating['movieId']].update(
					count=1,
					rating=rating['rating']
				)
				
			averages = {
				movie_id: (
					rating_counter['rating'] / rating_counter['count'],
					rating_counter['count']
				)
				for movie_id, rating_counter in rating_sums.items()
				if rating_counter['count'] >= self._min_ratings
			} # 각 movieId마다 평균 점수와 최소 점수 값을 추출
			
			return sorted(
				averages.items(),
				key=lambda x: x[1],
				reverse=True
			)[:self._top_n] # 점수의 평균과 점수를 개수로 정렬하여 top_n개의 점수를 반환
```
- 위의 Operator의 정확성을 어떻게 테스트해야 하나?
	- 주어진 값으로 오퍼레이터를 실행하고 결과가 예상대로인지 확인하여 전체적으로 테스트를 할 수 있다.
		- airflow 시스템 외부와 단위 테스트 내부에서 오퍼레이터를 단독으로 실행하기 위해 몇 가지 pytest 컴포넌트가 필요하다. 이것으로 다른 상황에서 오퍼레이터를 실행하고 올바르게 작동하는지 확인할 수 있다.

### Pytest 프로젝트 구성하기
- pytest를 사용하면 테스트 스크립트에 `test_` 접두사가 있어야 한다. 디렉토리 구조와 마찬가지로 파일 이름도 복제해 사용하므로 `test_movielens_operator.py` 라는 파일로 저장된다.
- BashOperator 테스트 방법
	- BashOperator를 인스턴스화하고 빈 컨텍스트를 전달하고 `execute()` 메소드를 호출한다.
		- 만일 테스트에 태스크 인스턴스 컨텍스트의 처리가 필요한 경우에는 필요한 키와 값으로 채운 딕셔너리 형태로 전달해야 한다.
	- airflow에서 오퍼레이터를 실행하면, 템플릿 변수를 확인하고 태스크 인스턴스 컨텍스트를 설정하여 오퍼레이터에 제공하는 등 여러 가지 작업을 실행 전후에 수행한다.
	- 테스트는 실제 운영 설정에서 실행하지 않고 `execute()` 메소드를 직접 호출한다.
- `pytest tests/dags/<path>/test_operator.py::test_example` 명령어로 테스트를 진행한다.
```python
def test_example():
	task = BashOperator(
		task_id='test',
		bash_command='echo "hello"',
		xcom_push=True,
	)
	result = task.execute(context={})
	assert result == 'hello'
```
- `MovielensPopularityOperator` 테스트하기
```python
def test_movielenspopularityoperator():
	task = MovielensPopularityOperator(
		task_id='test_id',
		start_date='2024-08-01',
		end_date='2024-08-02',
		top_n=5,
	)
	result = task.execute(context={})
	assert len(result) == 5
```

- 위의 테스트를 진행하면 메타스토어의 커넥션 ID를 가리키는 필수 인수인 `conn_id`가 누락되어 테스트가 실패한다.
	- 테스트 사이에 발생하는 정보를 데이터베이스를 이용해 전달하는 방법은 권장되지 않고 **목업(mocking)**을 이용해 해결한다.
![[airflow - Movielens operator failed.png|500]]
- 목업은 특정 작업이나 객체를 모조로 만드는 것이다.
	- 테스트 중에 발생하지 않는 데이터베이스에 대한 호출을 실제 발생시키지 않는 대신, 특정 값을 반환하도록 지시하여 임의의 값을 전달해 속이거나 모조하게 된다. 이를 통해 외부 시스템에 실제 연결하지 않고도 테스트를 개발하고 실행할 수 있다.
	- 테스트 중인 모든 내부 정보를 파악해야 하므로, 때로는 외부 시스템의 코드를 자세히 살펴보아야 하는 경우도 있다.
- Pytest 에는 목업과 같은 개념을 사용할 수 있도록 제공 플러그인 셋(Not Official)인 `pytest-mock` 파이썬 패키지를 제공한다.
	- pytest-mock은 내장된 목업 패키지를 편리하게 제공하기 위한 파이썬 패키지로 사용하려면 `mocker`라는 인수를 테스트 함수에 전달해야 한다.
		- `mocker`는 pytest-mock 패키지를 사용하기 위한 시작 점이다. 
```python
def test_movielenspopularityoperator(mocker): # 목업 객체는 런타임 시에 임의의 값으로 존재
	mocker.path.object( # 목업 객체로 객체의 속성을 패치
		MovielensHook, # 패치할 객체
		'get_connection', # 패치할 함수
		return_value=Connection(
			conn_id='test',
			login='airflow',
			password='airflow,
		) # 반환되는 값
	)
	task = MovielensPopularityOperator(
		task_id='test_id',
		conn_id='test',
		start_date='2024-08-01',
		end_date='2024-08-02',
		top_n=5,
	)
	result = task.execute(context=None)
	assert len(result) == 5
```
- `MovielensHook`에서의 `get_connection()` 메소드 호출은 몽키패치(런타임 시에 기능을 대체하여 Airflow 메타스토어를 쿼리하는 대신 지정된 객체를 반환함)이며 테스트 실행 시, `MovielensHook.get_connection()`은 실패하지 않는다.
	- 테스트 중에 존재하지 않는 데이터베이스 호출은 수행하지 않고 미리 정의된 예상 연결 객체를 반환하기 때문
- 테스트 시 패치된 객체를 호출할 때, 패치된 객체를 여러 속성을 보유하는 변수에 할당할 수 있다. `get_connection()` 메소드가 한 번만 호출되고, `get_connection()`에서 제공된 `conn_id` 인수가 `MovielensPopularityOperator`에 제공된 것과 동일한 값을 보유하고 있는지 확인함으로써 테스트시 실제 호출되었는지 확인할 수 있다.
목업 함수 동작 검증하기
```python
mock_get = mocker.patch.object( # 동작을 캡처하기 위한 목업 변수 할당
   MovielensHook,
   'get_connection',
   return_value=Connection(...),
)
task = MovielensPopularityOperator(..., conn_id='testconn')
task.execute(...)
assert mock.get.call_count == 1 # 한번만 호출된 것인지 확인
mock_get.assert_called_with('testconn') # 예상되는 conn_id로 호출된 것을 확인
```
- `mocker.patch.object`의 반환값을 `mock_get`이라는 변수에 할당하면 목업된 객체에 대한 모든 호출을 캡처하고 지정된 입력, 호출 수 등을 확인할 수 있다. 
	- get.call_count를 통해 호출된 횟수를 확인할 수 있다.
	- `conn_id 'testconn'`을 `MovielensPopularityOperator`에서 제공하며, 이 `conn_id`가 airflow 메타스토어에서 요청될 것으로 예상하며 `assert_called_with()`를 사용하여 검증한다.
- `mock_get` 객체는 검증하기 위해 더 많은 속성을 보유한다.
![[airflow - mock_get.png]]
- 파이썬에서 목업 환경 구성 중 잘못된 것은 잘못된 객체를 목업으로 구현하는 것이다. 위의 예제에는 `get_connection()`메소드를 목업으로 구현했다. 이 메소드는 `BaseHook`에서 상속된 `MovielensHook`에서 호출된다. `get_connection()`메소드는 `BaseHook`에 정의되어 있다. 따라서, 직관적으로는 `BaseHook.get_connection()`을 목업으로 구현하려고 할 것이다. 하지만 이는 올바른 방법이 아니다.
	- 올바른 방법은 정의되는 위치가 아니라 호출되는 위치에서 목업을 구성해야 한다.
올바른 목업 객체 구현
```python
from airflowbook.operators.movielens_opeartor import (
  MovielensPopularityOperator,
  MovielensHook,
) # 호출되는 위치에서 목업 메소드를 가져와야 한다.

def test_movielenspopularityoperator(mocker):
	mock_get = mocker.patch.object(
		MovielensHook,
		'get_connection',
		return_value=Connection(...),
	)
	task = MovielensPopularityOperator(...) # MovielensPopularityOperator 코드 내에서 MovielensHook.get_connection()를 호출
```


### 디스크의 파일로 테스트하기
- JSON 리스트를 가진 파일을 읽고 이를 CSV 형식으로 쓰는 오퍼레이터가 있다고 가정하면
```python
class JsonToCsvOperator(BaseOperator):
	def __init__(self, input_path, output_path, **kwargs):
		super().__init__(**kwargs)
		self._input_path=input_path
		self._output_path=output_path

	def execute(self, context):
		with open(self._input_path, 'r') as json_file:
			data = json.load(json_file)
			
		columns = {key for row in data for key in row.keys()}

		with open(self._output_path, 'w') as csv_file:
			writer = csv.DictWirter(csv_file, fieldnames=columns)
			writer.writeheader()
			writer.writerows(data)
```
- 위의 오퍼레이터를 테스트하기 위해 테스트 디렉토리에 고정 파일을 저장하여 테스트를 위한 입력으로 사용한다면 출력 파일은 어떻게 저장해야 하나?
	- 파이썬에는 임시 저장소와 관련된 작업을 위한 `tempfile` 모듈이 있다. 사용 후 디렉토리와 그 내용이 지워지기 때문에 파일 시스템에 항목이 남지 않는다.
	- pytest는 tmp_dir(os.path 객체 제공) 및 tmp_path(pathlib 객체 제공)라는 tempfile 모듈에 대한 편리한 사용 방법을 제공한다.
`JsonToCsvOperator` 테스트 코드
```python
import csv
import json
from pathlib import Path

from airflowbook.operators.json_to_csv_operator import JsonToCsvOperator


def test_json_to_csv_operator(tmp_path: Path): # tmp_apth는 고정으로 사용
	input_path = tmp_path / 'input.json'
	output_path = tmp_path / 'output.csv'

	input_data = [
		{'name': 'bob', 'age': '41', 'sex': 'M'},
		{'name': 'alice', 'age': '32', 'sex': 'F'},
		{'name': 'alice', 'age': '32', 'sex': 'F'},
	]
	
	with open(input_path, 'w') as f:
		f.write(json.dumps(input_data))
		
	operator = JsonToCsvOperator(
		task_id='test',
		input_path=input_path,
		output_path=output_path,
	)
	
	operator.execute(context={})
	
	with open(output_path, 'r') as f:
		reader = csv.DictReader(f)
		result = [dict(row) for row in reader]
	
	assert result == input_data 
	# 이후 tmp_path와 임시 파일은 제거
```
- `tmp_path`인수는 각 테스트 시 호출되는 함수를 나타낸다. `pytest`에서는 이를 [`fixture`](https://docs.pytest.org/en/stable/explanation/fixtures.html)라고 한다.
	- 단위 테스트의 unittest의 setUp() 및 tearDown() 메소드와 유사하지만, fixture는 여러 조건을 합치거나 매치시킬 수 있기 때문에 더 유연하게 사용할 수 있다.
- fixture는 기본적으로 모든 테스트 영역에서 적용 가능하다. 경로를 출력하여 서로 다른 테스트를 실행하거나, 동일한 테스트를 두 번 실행하면 이를 확인할 수 있다.
	- `print(tmp_path.as_posix())`시,
		![[airflow - pytest fixture.png]]

## 테스트에서 DAG 및 태스크 컨텍스트로 작업하기
- 일부 오퍼레이터는 실행을 위해 더 많은 컨텍스트 혹은 작업 인스턴스 컨텍스트가 필요하다.
	- 코드를 수행하는데 필요한 어떤 태스크 컨텍스트도 오퍼레이터에게 제공되지 않기 때문에 단순히 `operator.execute(context={})` 형태로 오퍼레이터를 실행할 수 없다.
- 사용자의 템플릿 변수를 통해 제공되어 동작하는 오퍼레이터를 구현했다고 가정하면
```python
class MovielensDownloadOperator(BaseOperator):
	template_fields = ('_start_date', '_end_date', '_output_path')

	def __init__(self, conn_id, start_date, end_date, output_path, **kwargs):
		super().__init__(**kwargs)
		self._conn_id = conn_id
		self._start_date = start_date
		self._end_date = end_date
		self._output_path = output_path

	def execute(self, context):
		with MovielensHook(self._conn_id) as hook:
			ratings = hook.get_ratings(
				start_date=self._start_date,
				end_date=self._end_date
			)
		with open(self._output_path, 'w') as f:
			f.write(json.dumps(ratings))
```

![[Pasted image 20240903165811.png]]

- 이 오퍼레이터는 태스크 인스턴스 콘텍스트를 실행해야 하므로 이전 예제와 같이 테스트할 수 없다.
	- 예를 들어, `output_path` 인수를 `/output/{{ logical_date }}.json` 형태로 제공할 수 있지만 `operator.execute(context = {})`로 테스트할 때는 logical_date 변수를 사용할 수 없다.
- airflow 자체에서 태스크를 시작할 때 사용하는 메소드인 `operator.run()`을 호출한다. 이를 사용하려면 airflow가 태스크를 실행할 때 DAG 객체를 참조하기 때문에 오퍼레이터에게 DAG를 제공해야 한다.
![[Pasted image 20240903170049.png]]
![[Pasted image 20240903170056.png]]
![[Pasted image 20240903170108.png]]