# Chapter 2. Airflow의 DAG 구조

주차: 1주차
발표: 창배님
생성 일시: 2024년 7월 23일 오후 4:47

### airflow DAG 작성

![Untitled](Chapter%202%20Airflow%E1%84%8B%E1%85%B4%20DAG%20%E1%84%80%E1%85%AE%E1%84%8C%E1%85%A9%207d394bdf19e445f4804856428aecf8c1/Untitled.png)

- bash 스크립트와 Python 스크립트를 혼용

```python
import json
import pathlib
import airflow
import requests
import requests.exceptions as requests_exceptions

from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

"""
워크플로 시작점
dag_id : DAG 이름
start_date : DAG 처음 실행 시작 날짜
schedule_interval : DAG 실행 간격
"""
dag = DAG( 
    dag_id="download_rocket_launches", # Airflow UI에 표시된
    start_date=airflow.utils.dates.days_ago(14),
    schedule_interval=None,
)

"""
BashOperator : curl 로 만든 URL 결과값 다운로드
task_id : 태스크 이름
"""
download_launches = BashOperator(
    task_id="download_launches",
    bash_command="curl -o /tmp/launches.json -L 'https://ll.thespacedevs.com/2.0.0/launch/upcoming'",
    dag=dag,
)

# 파이썬 함수 : 결과값을 파싱하고 모든 로켓 사진을 다운로드
def _get_pictures():
    pathlib.Path("/tmp/images").mkdir(parents=True, exist_ok=True)
    with open("/tmp/launches.json") as f:
        launches = json.load(f)
    image_urls = [launch["image"] for launch in launches["results"]]
    for image_url in image_urls:
        try:
            response = requests.get(image_url)
            image_filename = image_url.split("/")[-1]
            target_file = f"/tmp/images/{image_filename}"
            with open(target_file, "wb") as f:
                f.write(response.content)
            print(f"Downloaded {image_url} to {target_file}")
        except requests_exceptions.MissingSchema:
            print(f"{image_url} appears to be an invalid URL.")
        except requests_exceptions.ConnectionError:
            print(f"Could not connect to {image_url}.")

"""
PythonOperator : DAG에서 Python 함수 호출
python_callable : 파이썬 함수 호출
"""
get_pictures = PythonOperator(
    task_id="get_pictures",
    python_callable=_get_pictures,
    dag=dag,
)

"""
BashOperator 
bask_command : bash 명령어 실행
"""
notify = BashOperator(
    task_id="notify",
    bash_command='echo "There are now $(ls /tmp/images/ | wc -l) images."',
    dag=dag,
)

# 테스크 실행 순서 결정
download_launches >> get_pictures >> notify
```

<aside>
💡 ***의존성***
오퍼레이터는 서로 독립적으로 실행할 수 있지만, 순서를 정의해 실행할 수 있다.

</aside>

## 태스크와 오퍼레이터의 차이점

![Untitled](Chapter%202%20Airflow%E1%84%8B%E1%85%B4%20DAG%20%E1%84%80%E1%85%AE%E1%84%8C%E1%85%A9%207d394bdf19e445f4804856428aecf8c1/Untitled%201.png)

### 오퍼레이터

- 단일 작업을 수행
- 일반적인 작업 : BashOperator, PythonOperator
- 특수한 목적의 작업 : EmailOperator, SimpleHTTPOperator
- 다양한 서브 클래스 제공

### DAG

- 오퍼레이터 집합에 대한 실행을 오케스트레이션 (조정, 조율)
- 오퍼레이터의 시작과 정지, 오퍼레이터 완료 후 연속된 다음 태스크 시작 등 관리
- 오퍼레이터 간의 의존성 보장

### 태스크

- 오퍼레이터의 래퍼(Wrapper), 매니저(Manager)
- 사용자는 오퍼레이터를 활용해 수행할 작업에 집중 → 실행을 관리
- 사용자 관점에서 오퍼레이터와 태스크가 같은 의미 (혼용해서 사용)

## 임의의 파이썬 코드 실행

```python
# 파이썬 함수 : 결과값을 파싱하고 모든 로켓 사진을 다운로드
def _get_pictures():

		# 1. 저장할 디렉터리가 있는 지 확인 - 없으면 생성 
    pathlib.Path("/tmp/images").mkdir(parents=True, exist_ok=True)
    
    # 2. 로켓 발사에 대한 이미지 추출
    with open("/tmp/launches.json") as f:
        launches = json.load(f)
    image_urls = [launch["image"] for launch in launches["results"]]
    
    # 3. 반환된 이미지 URL에서 모든 이미지 다운로드
    for image_url in image_urls:
        try:
            response = requests.get(image_url)
            image_filename = image_url.split("/")[-1]
            target_file = f"/tmp/images/{image_filename}"
            with open(target_file, "wb") as f:
                f.write(response.content)
            print(f"Downloaded {image_url} to {target_file}")
        except requests_exceptions.MissingSchema:
            print(f"{image_url} appears to be an invalid URL.")
        except requests_exceptions.ConnectionError:
            print(f"Could not connect to {image_url}.")

"""
PythonOperator : DAG에서 Python 함수 호출
python_callable : 파이썬 함수 호출
"""
get_pictures = PythonOperator(
    task_id="get_pictures",
    python_callable=_get_pictures,
    dag=dag,
)
```

### PythonOperator

1. 오퍼레이터 자신(get_pictures)을 정의
2. python_callable은 인수에 호출이 가능한 일반함수(_*get_pictures* )를 가르킨다.

```python
def _get_pictures() : # PythonOpreator Callable
 # 여기서 작업 수행 .. 
get_pictures = PythonOperator(
		task_id = "get_pictures",
		python_callable=_get_pictures, # 실행할 함수를 가리킴
		dag = dag
)
```

### 스케줄 간격으로 실행하기

airflow에서는 DAG를 일정 시간 간격으로 실행할 수 있도록 스케줄 설정이 가능하다.

(시간, 일, 월에 한번 수행)

```python
dag = DAG(
		dag_id = "download_rocket_lauches",
		start_date = airflow.utils.dates.days_ago(14),
		schedule_interval = "@daily",
)
```

- schedule_interval을 daily 로 설정하면, Airflow 워크플로를 하루에 한번 실행하기 때문에 사용자가 직접 트리거할 필요가 없다.

> 나머지는 Airflow 웹 인터페이스를 기준으로 설명한 내용! :)
>