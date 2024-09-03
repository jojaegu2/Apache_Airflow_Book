---
Chapter: 커스텀 컴포넌트 빌드
---
## 커스텀 오퍼레이션
- 특정 작업에 대해 Airflow가 오퍼레이터를 지원하지 않는 경우에는 커스텀 오퍼레이터로 원하는 작업을 실행하는 오퍼레이터를 구현할 수 있다.
- 공통으로 사용하는 커스텀 오퍼레이션을 생성하여 단순화시킬 수 있다.
- Airflow에서 특정 시스템에서 작업이 실행되어야할 때, 이를 위한 커스텀 오퍼레이터를 개발하여 빌드해왔다.

## PythonOperator로 작업하기
- 평점 데이터를 기반으로 영화를 추천해 주는 추천 시스템을 구현
	- 평점 데이터는 API를 통해 제공되며, 특정 기간의 영화 평점을 제공한다.
- 데이터를 일별로 가져오는 프로세스를 구축하고, 인기 순서로 영화 랭킹을 생성하고 랭킹으로 다운스트림에서 인기 영화를 추천한다.
![[Airflow - 영화 추천 프로젝트.png]]

### 영화 평점 API 시뮬레이션하기
- [MovieLens 데이터 세트](https://grouplens.org/datasets/movielens) 에는 62,000개의 영화에 대해 162,000 명의 유저가 평가한 2천 5백만 개의 평점 데이터로 구성되어있다.
- 데이터 세트는 플랫 파일(CSV 등과 같이, 같은 형식의 레코드들의 모임으로 이루어진 파일)로만 제공되고, 여러 엔드포인트에 데이터를 부분별로 제공하는 REST API를 Flask로 사용하여 이미 구현
- [REST API 소스 코드](https://github.com/K9Ns/data-pipelines-with-apache-airflow/tree/main/chapter08/docker/movielens-api) 를 docker compose 파일에 추가하고 빌드하게 되면, localhost:5000로 영화 평점 데이터를 조회할 수 있다.
	- 데이터는 JSON 포맷으로 제공되고 limit과 offset을 같이 제공
	- limit, offset을 조절하여 가져올 데이터 범위와 갯수를 정의할 수 있다.
		- `http://localhost:5000/ratings?offset=100` or `http://localhost:5000/ratings?limit=10000` 
	- 특정 기간의 데이터를 가져오기 위해서는 `start_date`와 `end_date`를 지정하면 된다.
		- `http://localhost:5000/ratings?start_date=2019-01-01&end_date=2019-01-02`
	- 필터링 기능을 통해 데이터 세트를 전부 로드할 필요 없이 증분 방식으로 데이터를 로드할 수 있다.
![[Airflow - movielens ratings.png]]

### API에서 평점 데이터 가져오기
- Airflow에서 평점 데이터 수집을 자동화 구현
	- API 접근을 위해 `requests` 라이브러리내 `Session` 클래스 사용
	- get 메소드를 이용해 GET HTTP API 요청을 수행하고 API로부터 평점 데이터를 가져옴.
	- params에 쿼리에 사용되는 파라미터 등 추가 인수 값을 get 메소드와 함께 전달
	- HTTP 요청 결과인 `response` 객체에 `raise_for_status()`를 통해 상태 코드로 요청의 정상 수행 여부를 확인. 만일 예상치 못한 상태 코드를  반환하는 경우, 예외를 발생시킬 수 있다.
	- 인증 정보(id, password)를 전달하기 위해 `Session.auth`에 인증 정보를 전달

- `requests.Session`으로 HTTP 세션 생성
```python
imoprt requests

# 민감한 정보이기 때문에 미리 지정한 환경 변수를 이용
MOVIELENS_USER = os.environ["MOVIELENS_USER"]
MOVIELENS_PASSWORD = os.environ["MOVIELENS_PASSWORD"]

def _get_session():
	'''Build a requests Session for the Movielens API'''

	session = requests.Session()  # Request 세션 생성
	session.auth = (MOVIELENS_USER, MOVIELENS_PASSWORD)  # 사용자 이름과 비밀 번호 기반하여 기본 HTTP 인증을 위한 세션 설정

	base_url = 'http://localhost:5000'

	return session, base_url
```
- 스크립트 실행에 필요한 환경 변수 설정만으로 편하게 수정할 수 있다.

- API 결과의 페이지 처리
```python
def _get_with_pagination(session, url, params, batch_size=100):
	'''
	Fetches records using a GET request with given URL/params, taking pagination into account
	'''
	offset = 0
	total = None

	with total is None or offset < total:  # 모든 레코드들을 받을 때가지 반복
		response = session.get(
			url,
			params={
				**params,
				**{'offset': offset, 'limit': batch_size },
			},
		)
		
		response.raise_for_status()  # 결과 상태 체크
		response_json = response.json()  # 결과 값이 json 형태
		
		yield from resonse.json['result']
		
		offset += batch_size
		total = response_json['total']
```
- 결과 레코드의 끝까지 도달할 때까지 새 페이지들을 반복적으로 요청
- `yield from` 을 사용하여 레코드들의 제너레이터를 전달함으로써 결과 페이지 결과를 `lazy loading` 가능하게 만듦

- 데이터를 가져오는 함수
```python
def _get_ratings(start_date, end_date, batch_size=100):
	session, base_url = _get_session()

	yield from _get_with_pagination(  # 페이지네이션 함수 활용
		session=session,
		url=base_url + '/ratings',
		params={'start_date': start_date, 'end_date': end_date},
		batch_size=batch_size
	)
	
ratings = _get_ratings(start_date, end_date)
next(ratings)  # 한 페이지에 대한 레코드만 가져오기
list(ratings)  # 전체 배치(batch)에 대한 데이터를 가져오기
```
- DAG에서는 `_get_ratings` 함수를 `PythonOperator`에서 호출하는 것으로 데이터를 가져오는 태스크를 구현할 수 있다.

- 데이터 가져오기 태스크 구현한 코드
```python
import logging
import requests


def _get_ratings(start_date, end_date, batch_size=100):
    session, base_url = _get_session()

    yield from _get_with_pagination(
        session=session,
        url=base_url + "/ratings",
        params={"start_date": start_date, "end_date": end_date},
        batch_size=batch_size,
    )


def _get_session():
    """Builds a requests Session for the Movielens API."""

    # Setup our requests session.
    session = requests.Session()
    session.auth = (MOVIELENS_USER, MOVIELENS_PASSWORD)

	base_url = 'http://localhost:5000'

    return session, base_url


def _fetch_ratings(templates_dict, batch_size=1000, **_):
    logger = logging.getLogger(__name__)

    start_date = templates_dict["start_date"]
    end_date = templates_dict["end_date"]
    output_path = templates_dict["output_path"]

    logger.info(f"Fetching ratings for {start_date} to {end_date}")
    ratings = list(
        _get_ratings(
            start_date=start_date, end_date=end_date, batch_size=batch_size
        )
    )
    logger.info(f"Fetched {len(ratings)} ratings")

    logger.info(f"Writing ratings to {output_path}")

    # Make sure output directory exists.
    output_dir = os.path.dirname(output_path)
    os.makedirs(output_dir, exist_ok=True)

    with open(output_path, "w") as file_:
        json.dump(ratings, fp=file_)
...

with DAG():
	fetch_ratings = PythonOperator(
	    task_id="fetch_ratings",
	    python_callable=_fetch_ratings,
	    templates_dict={
	        "start_date": "{{ data_interval_start }}",
	        "end_date": "{{ data_interval_end }}",
	        "output_path": "/data/python/ratings/{{ logical_date }}.json",
	    },
	)
```
- `start_date와 end_date, output_path`를 Templates로 전달하여 증분 처리하고 결과 데이터를 파티셔닝 할 수 있다.

### 평점 데이터로 랭킹 만들기
- 영화 랭킹 계산하는 함 구현
```python
import pandas

def rank_movie_by_rating(ratings, min_ratings=2):
	ranking = (
		ratings.groupby('movieId')
		.agg(
			avg_rating=pd.NamedAgg(column='rating', aggfunc='mean'),
			num_ratings=pd.NamedAgg(column='userId', aggfunc='nunique),
		)
		.loc[lambda df: df['num_ratings'] > min_ratings]
		.sort_values(['avg_rating', 'num_ratings'], ascending=False)
	)
	return ranking
```
- `pandas dataframe`으로부터 영화의 평균 평점과 평점 총 개수를 계산
- 최소 평점 개수 기준으로 영화를 필터링
- 평균 평점 기준으로 정렬

- 영화 랭킹 결과를 처리하는 태스크
```python
import os

...

def _rank_movies(templates_dict, min_rating=2, **context):
	input_path = templates_dict['input_path']
	output_path = templates_dict['output_path']
	
	ratings = pd.read_json(input_path)
	ranking = rank_movie_by_rating(ratings, min_ratings=min_ratings)
	
	output_dir = os.path.dirname(output_path)
	os.makedirs(output_dir, exist_ok=True)
	
	ranking.to_csv(output_path, index=True)

with DAG():
	rank_movies = PythonOperator(
		task_id='rank_movies',
		python_callable=_rank_movies,
		templates_dict={
			'input_path': '/data/python/ratings/{{ logical_date }}.json',
			'output_path': '/data/python/ranking/{{ logical_date }}.csv'
		}
	)
```
- `output_path`에 templates를 적용함으로서 랭킹 결과 파일을 파티셔닝할 수 있다.

## 커스텀 훅 빌드하기
- API로부터 영화 평점 데이터 가져올 때, API 주소와 인증 정보를 추가한 Session을 가져오고, 페이지 처리와 같은 기능들이 추가되었는데 이는 복잡한 작업이다.
	- 복잡한 작업을 처리하는 방법으로, 코드를 캡슐화하고 재활용 가능한 Airflow 커스텀 훅으로 만들 수 있다.
- 커스텀 훅을 구현함으로써, 같은 기능을 하는 훅을 하나의 코드로 관리하고 DAG의 여러 부분에서 커스텀 훅을 사용함으로써 간단한 DAG 구현이 가능하다.
- 커스텀 훅을 사용하면 데이터베이스와 UI를 통해 자격 증명과 Connection 관리 기능을 사용할 수 있다.

### 영화 평점 API에 커스텀 훅 설계하기
- Airflow에서 모든 훅은 추상 클래스인 `BaseHook` 클래스의 서브클래스로 생성한다.
- 훅을 구현하기 위해 연결(필요한 경우 )과 훅에 필요한 다른 추가적인 인수를 지정하는 `__init__` 메소드를 정의한다
	- 훅이 특정 연결에서 연결 세부 정보를 가져와야 하지만, 다른 추가 인수는 필요하지 않는다.
- 대부분의 Airflow 훅은 get_conn 메소드를 정의하는데, 외부 시스템과의 연결 설정을 책임진다.
- 자격 증명과 같이 민감한 정보는 하드코딩 대신 Airflow Connection에 추가하여 사용하는 것이 더 좋다.
	- BaseHook 클래스 내 `get_connection` 메소드를 통해 민감한 정보를 접근할 수 있다.
	- Connection 을 가져올 때마다 Airflow 메타 데이터베이스에서 Connection 정보를 가져오기 때문에 캐싱을 이용해 해결할 수 있다.
![[Airflow - HTTP Connection.png]]
```python
import requests
from airflow.hooks.base_hook import BaseHook

class MovielesHook(BaseHook):
	def __init__(self, _conn_id):
		super().__init__()  # BaseHook의 생성자 호출
		self._conn_id = _conn_id
		self._session = None
		self._base_url = None
	...
	
	def get_conn(self):
		if self._session is None:
			config = self.get_connection(self._conn_id)
			# Setup our requests session.
		    session = requests.Session()
		    session.auth = (config.id, config.password)

			base_url = 'http://localhost:5000'
		return self._session, self_base_url
	    
```

- 평점 가져오는 메소드를 훅에 추가
```python
...

class MovielensHook(BaseHook):
	...
	def get_ratings(self, start_date=None, end_date=None, batch_size=100):
		yield from self._get_with_pagination(
			endpoint='/ratings',
			params={'start_date': start_date, 'end_date': end_date},
			batch_size=batch_size
		)

	def _get_with_pagination(self, endpoint, params, batch_size=100):
		session, base_url = self.get_conn()

		offset = 0
		total = None
		while total is None or offset < total:
			response = session.get(
				url,
				params={
					**params,
					**{'offset': offset, 'limit': batch_size},
				}
			)

			response.raise_for_status()
			response_json = response.json()
			
			yield from response_json['result']
			
			offset += batch_size
			total = response_json['total']
```
- 위에서 PythonOperator와 함수로 구현한 코드를 그대로 커스텀 훅을 생성할 수 있다.

### 커스텀 훅으로 DAG 빌드
- 생성한 커스텀 훅을 DAG에서 불러올 수 있도록 어딘가에 저장해야 하는데, DAG 디렉토리 안에 패키지를 생성하고, 이 패키지 안에 있는 hook.py 라는 모듈에 훅을 저장하는 방법이 있다.
![[Airflow - custom hook dir.png]]
- `from custom.hooks import MovielensHook` 을 통해 훅을 불러올 수 있다.
- 훅의 메소드를 호출하는 것으로 영화 평점 데이터를 가져오는 부분을 구현할 수 있다.
```python
hook = MovielensHook(conn_id=conn_id)

ratings = hook.get_ratings(
	start_date=start_date,
	end_date=end_date,
	batch_size=batch_size
)
```
- 위의 코드를 수행하면 제너레이터를 반환하는데, 이를 사용하여 데이터를 출력 파일(JSON)로 저장한다.
- DAG에 훅을 사용하기 위해서는 훅 호출 코드를 PythonOperator에 래핑해야 한다.
```python
def _fetch_ratings(conn_id, templates_dict, batch_size=1000, **_):
    logger = logging.getLogger(__name__)

    start_date = templates_dict["start_date"]
    end_date = templates_dict["end_date"]
    output_path = templates_dict["output_path"]

    logger.info(f"Fetching ratings for {start_date} to {end_date}")
    hook = MovielensHook(conn_id=conn_id)
    ratings = list(
	    hook.get_ratings(
		    start_date=start_date, end_date=end_date, batch_size=batch_size
	    )
    )
    logger.info(f"Fetched {len(ratings)} ratings")

    logger.info(f"Writing ratings to {output_path}")

    # Make sure output directory exists.
    output_dir = os.path.dirname(output_path)
    os.makedirs(output_dir, exist_ok=True)

    with open(output_path, "w") as file_:
        json.dump(ratings, fp=file_)
...

with DAG():
	fetch_ratings = PythonOperator(
	    task_id="fetch_ratings",
	    python_callable=_fetch_ratings,
	    op_kwargs={'conn_id': '<connection ID>'},
	    templates_dict={
	        "start_date": "{{ data_interval_start }}",
	        "end_date": "{{ data_interval_end }}",
	        "output_path": "/data/python/ratings/{{ logical_date }}.json",
	    },
	)
```
- 커스텀 훅을 사용하여 똑같은 기능을 하는 태스크를 더 간단하고 간소화하여 구현할 수 있다. 또한, 다른 DAG에서 커스텀 훅에서 호출하여 사용할 수 있다.

## 커스텀 오퍼레이터 빌드하기
- 커스텀 훅으로 DAG의 복잡한 부분을 많이 변경했지만, 여전히 시간/종료 날짜와 평점 데이터의 파일 저장에 대해 상당히 많은 반복적인 코드를 작성해야 한다.
- 여러 개의 DAG에 이 기능들을 재사용하려고 할 때, 상당한 코드 중복과 이를 처리하기 위한 부가적인 노력이 필요하다
- Airflow에서는 커스텀 오퍼레이터를 직접 구현해서, 반복적인 태스크 수행 시 코드의 반복을 최소화 할 수 있다.

### 커스텀 오퍼레이터 정의하기
- 모든 오퍼레이터는 `BaseOperator` 클래스의 서브 클래스로 만들어야 한다.
```python
# 커스텀 오퍼레이터 베이스 코드
from airflow.models import BaseOperator
from airflow.utils.decorators imoprt apply_defaults


class MyCustomOperator(BaseOperator):
	@apply_defaults  # 기본 DAG 인수를 커스텀 오퍼레이터에게 전달하기 위한 데코레이터
	def __init__(self, conn_id, **kwargs):
		super.__init__(self, **kwargs)
		self.conn_id = conn_id
		...
```
- 커스텀 오퍼레이터에만 사용되는 인수들을 `__init__` 생성자에 지정할 수 있다.
	- 오퍼레이터에만 사용되는 인수들은 오퍼레이터마다 서로 다르지만, 일반적으로 커넥션 ID와 작업에 필요한 세부사항(시작/종료 날짜 등)이 포함된다.
- `BaseOperator` 클래스는 오퍼레이터의 일반적인 동작을 정의하는 제너릭 인수들을 많이 가지고 있다.
	- 제너릭 인수로 오퍼레이터가 생성한 `task_id, retries, retry_delay` 등과 같은 다른 많은 인수들이 포함될 수 있다.
	- 제너릭 인수들을 모두 나열하지 않도록 `BaseOperator` 클래스의 `__init__`에 인수를 전달할 때 파이썬의 `**kwargs` 구문을 사용한다.
- Airflow에서 사용되는 특정 인수를 전체 DAG의 기본 인수로 정의할 수 있다.
	- DAG 객체 내부의 `default_args` 속성 값을 사용하면 가능하다.
```python
# default args 적용 코드
default_args = {
	'retries': 1,
	'retry_delay': timedelta(minutes=5)
	...
}

with DAG(
	default_args=default_args
) ...
```
- 커스텀 오퍼레이터의 기본 인수들이 정상적으로 적용되었는지 확인하기 위해, airflow에서 지원하는 `apply_defaults`라는 데코레이터를 사용할 수 있다.
	- 이 데코레이터는 커스텀 오퍼레이터의 `__init__` 메소드에 적용된다.
- 오퍼레이터가 실제로 작업해야 하는 사항을 `execute` 메소드에 구현한다. `execute` 메소드는 Airflow가 DAG를 실행할 때 DAG 안에서 실행되는 오퍼레이터의 메인 메소드가 된다.
```python
class MyCustomOperator(BaseOperator):
	...
	def execute(self, context):
		...
```
- `execute` 메소드는 `context`라는 하나의 파라미터만 받으며, 이 파라미터는 airflow의 모든 콘텍스트 변수를 담고 있는 딕셔너리 객체이다.
- `execute` 메소드는 airflow 콘텍스트의 변수들을 이 파라미터에서 참조하고 해당 오퍼레이터가 해야 하는 작업을 수행한다.

### 평점 데이터를 가져오기 위한 오퍼레이터 빌드하기
- `_fetch_ratings` 함수와 유사하게 동작하며, 주어진 시작/종료 날짜 사이의 평점 데이터를 가져와서 JSON 파일로 저장하는 오펴레이터이다.
```python
# dags/custom/operator.py
class MovielensFetchRatingsOperator(BaseOperator):
	@apply_defaults
	def __init__(self, conn_id, output_path, start_date, end_date, **kwargs):
		super(MovielensFetchRatingsOperator, self).__init__(**kwargs)
		
		self._conn_id = conn_id
		self._output_path = output_path
		self._start_date = start_date
		self._end_date = end_date

	def execute(self, context):
		hook = MovielensHook(self._conn_id). # 커스텀 훅 불러오기
		
		try:
			self.log.info('...')
			ratings = list(
			hook.get_ratings(
				start_date=self._start_date, end_date=self._end_date)
			)
			self.log.info('...')
		finally:
			hook.close()  # 사용한 커스텀 훅 종료
		
		...
		
		output_dir = os.path.dirname(self._output_path)
		os.makedirs(output_dir, exist_ok=True)
		
		with open(self._output_path, 'w') as f:
			json.dump(ratings, fp=f)
```
- `execute` 메소드 내부 구현은 `PythonOperator`에서 callable한  `_fetch_ratings` 을 거의 그대로 사용
	- 오퍼레이터 인스턴스를 생성할 때 받은 변수 값을 사용하기 위해, self에서 파라미터를 가져오는 점만 기존과 다른 점이다.
	- 로깅할 때 `BaseOperator`에서 제공하는 logger를 사용하도록 수정하였는데, 이 logger는 `self.log` 속성으로 사용할 수 있다.
	- 훅을 정상적으로 종료할 수 있게 예외 처리를 추가하였는데 API 세션을 닫는 것을 잊어버렸을 때 생기는 리소스의 낭비를 막을 수 있으므로, 훅을 사용하는 코드를 구현할 때 좋은 습관이 될 수 있다.

### 커스텀 오퍼레이터 호출
```python
fetch_ratings = MovielensFetchRatingsOperator(
	task_id='fetch_ratings',
	conn_id='<conn_id>',
	start_date='2020-01-01',
	end_date='2020-01-02',
	output_path='/data/2020-01-01.json'
)
```
- 하드 코딩된 값에 대해서 오퍼레이터가 동작하도록 되어 있어지만 이를 템플릿 가능한 오퍼레이터 변수로 만들 수 있다.
```python
class MovielensFetchRatingsOperator(BaseOperator):
	'''
	template_fields = ('_start_date', '_end_date', '_output_path') # 커스텀 오퍼레이터에서 템플릿화할 인스턴스 변수들을 선언
	'''

	@apply_defaults
	def __init__(
		self,
		conn_id,
		output_path='/data/python/ratings/{{ logical_date }}.json',  # 템플릿으로 기본값을 지정
		start_date='{{ data_interval_start }}',
		end_date='{{ data_interval_end }}',
		**kwargs
	):
		...
```
- 템플릿을 사용하여 `execute` 메소드를 호출하기 전에 값들을 미리 템플릿화할 수 있다.
```python
fetch_ratings = MovielensFetchRatingsOperator(
	task_id='fetch_ratings',
	conn_id='<conn_id>',
	start_date='{{ data_interval_start }}',
	end_date='{{ data_interval_end }}',
	output_path='/data/python/ratings/{{ logical_date }}.json'
)
```


## 커스텀 센서 빌드하기
- 센서는 커스텀 오퍼레이터와 매우 유사하고 `BaseOperator` 대신 `BaseSensorOperator` 클래스를 상속한다.
- 센서는 오퍼레이터의 한 유형으로, `BaseSensorOperator`는 센서에 대한 기본 기능을 제공하고, 오퍼레이터의 `execute` 메소드 대신 `poke` 메소드를 구현해야 한다.
```python
class MyCustomSensorOperator(BaseSensorOperator):
	...
	
	def poke(self, context):
		...
```
- poke 메소드는 결과 값이 Boolean 타입인데, 센서의 상태를 True/False로 나타낸다.
	- 상태가 False면, 센서의 상태를 다시 체크할 때까지 몇 초 대기 상태로 들어간다.
	- 이 프로세스는 상태 값이 True가 되거나 타임 아웃이 될 때까지 반복한다.
	- 센서가 True를 반환하면 다운스트림 태스크를 시작한다.
- 기본 내장 센서가 많이 있지만, 특정 유형의 상태를 체크하는 별도의 센서가 필요할 때가 있다.
	- 데이터가 사용 가능한지 체크하는 센서 등
- 커스텀 센서를 빌드하기 위해서는 커스텀 오퍼레이터처럼 `__init__` 생성자 메소드를 정의한다.
	- 커넥션 Id와 데이터의 시작/종료 날짜의 범위를 입력으로 받아들어야 한다.
```python
from airflow.sensors.base imoprt BaseSensorOperator
from airflow.utils.decorators import apply_defaults

class MovielensRatingsSensor(BaseSensorOperator):
	template_fileds = ('_start_date', '_end_date')

	@apply_defaults
	def __init__(self, conn_id, start_date='{{ data_interval_start }}', end_date='{{ data_interval_end }}', **kwargs):
		super().__init__(**kwargs)
		self._conn_id = conn_id
		self._start_date = start_date
		self._end_date = end_date
```
- 특정 기간안에 영화 평점 데이터가 존재하는지 체크하고, 상태 값을 반환하는 `poke` 메소드만 구현해주면 된다.
- 미리 생성해두었던 커스텀 훅을 이용해 레코드를 가져오고, `get_ratings` 에서 반환된 제너레이터에서 `next`를 호출하여, 제너레이터가 빈 값인 경우 StopIteration이라는 예외를 발생시킨다. 따라서, try/except를 사용하여 예외 사항을 체크하고, 예외가 발생하지 않으면 True를 반환하고 그렇지 않으면 False를 반환한다.
```python
class MovielensRatingsSensor(BaseSensorOperator):
	def poke(self, context):
		hook = MovielensHook(self._conn_id)

		try:
			next(
				hook.get_ratings(
					start_date=self._start_date,
					end_date=self._end_date,
					batch_size=1
				)
			)  # 전체 레코드에서 하나만 가져오기
			self.log.info('....')
		except StopIteration:
			self.log.info('....')
			return False
		fianlly:
			hook.close()
		return True
```
- 위의 커스텀 센서를 영화 랭크를 생성하기 전에 추가하여 기간 내에 데이터가 존재하는지 확인하고 없으면 대기하도록 하여 랭크 생성하는데 올바른 데이터를 전달할 수 있다.

## 컴포넌트 패키징하기
- 커스텀 컴포넌트를 DAG 디렉토리 내에 있는 서브 패키지에 포함시켰는데 이는 다른 프로젝트에서 사용하거나 공유할 경우, 혹은 컴포넌트들을 엄격히 테스트할 경우에는 이상적인 방법이 아니다.
- 파이썬 패키지에 코드를 넣는 것으로 더 나은 방법으로 커스텀 컴포넌트를 배포할 수 있다.
	- Airflow 구성 환경에 커스텀 컴포넌트를 설치할 때에 다른 패키지와 비슷한 방법으로 작업할 수 있다는 장점이 있다.
	- DAG와는 별도로 코드를 유지함으로써, 커스텀 코드에 대한 CI/CD 프로세스를 구성할 수 있고 다른 사람과 이 코드를 더 쉽게 공유하고 협업할 수 있다.

### 파이썬 패키지 부트스트랩 작업하기
- `setuptools` 를 사용하여 간단한 파이썬 패키지를 생성할 수 있다.
1. 패키지를 빌드하기 위해서 디렉토리 생성
	`mkdir -p <package name>`
	`cd <package name>`
2. 소스 코드를 보관할 패키지의 기본 구조를 생성
	- 디렉토리에 src 라는 하위 디렉토리를 생성하고, src 디렉토리 안에 패키지 이름으로 디렉토리를 생성한다.
	`mkdir -p src/<package name>`
	- 해당 디렉토리를 패키지로 만들기 위해 `src/<package name>` 안에 `__init__.py`를 생성한다.
	`touch src/<package name>/__init__.py`
3. `src/<package name>` 디렉토리 안에 커스텀 컴포넌트의 소스 파일을 추가한다.
![[Pasted image 20240827173457.png]]
4. 디렉토리 하위에 `setup.py` 을 생성하여 포함시킨다. `setup.py` 파일은 `setuptools`가 패키지를 어떻게 설치할지 지시하는 파일이다.
	- `setup.py` 파일에서 가장 중요한 부분은 `setuptools.setup` 호출이며, 패키지에 대한 메타데이터를 전달한다.
	- `setuptools.setup`에서 중요한 필드 리스트
		- name - 패키지 이름
		- version - 패키지 버전 번호
		- install_requires - 패키지에 필요한 종속 라이브러리 목록
		- packages/package_dir - 설치시 포함되어야 할 패키지들과 패키지들의 위치를 setuptools 에게 전달
	- `setuptools.setup`에서 선택적 필드
		- author - 패키지의 저자
		- author_email - 저자의 이메일
		- description - 패키지에 대한 짧고 가독성 있는 설명. 설명이 길어지면 long_description 을 사용
		- url - 온라인에서 패키지를 찾을 수 있는 위치
		- license - 패키지 코드를 배포할 때 적용하는 라이선스
![[Pasted image 20240827173611.png]]
- 정교한 패키지에는 테스트, 설명 문서 등 과 같은 내용들이 더 추가된다.
- [파이썬 패키지의 셋업 항목에 대한 템플릿](https://github.com/audreyfeldroy/cookiecutter-pypackage)

### 패키지 설치하기
- `python -m pip install ./<package name>`


### 패키지 설치 방법
1. 깃헙 저장소에 배포하여  설치
`python -m pip install git+https://github.com/..`
2. PyPI와 같이 pip 패키지 피드를 사용해서 설치
`python -m pip install <pagekage name>`
3. 파일을 직접 설치
	- 파일의 위치를 지정하여 설치할 수 있다. 이 경우에는 설치할 패키지의 디렉토리에 대한 접근 권한을 Airflow에게 주어야 한다.