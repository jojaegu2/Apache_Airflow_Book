---
Chapter: Airflow DAG 구조
---
## Airflow 환경 구성
1. Custom image 생성
	- BASE 이미지: [apache/airflow:2.9.3-python3.9](https://hub.docker.com/layers/apache/airflow/2.9.3-python3.9/images/sha256-c41c054f1660d62a584392c7922c047395a1134ee64aaf1216f0b1e33931e158?context=explore)
```Docker
# Dockerfile
FROM apache/airflow:2.9.3-python3.9
LABEL maintainer seunghyun.lim <limseunghyun95@gmail.com>

COPY requirements.txt /
RUN pip install -r /requirements.txt
```

2. 도커 이미지 빌드
	- `docker build . -t <image>`


3. 공식 docker-compose.yaml 다운로드
	- `curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.9.3/docker-compose.yaml'`

4. docker-compose.yaml 수정
	- `x-airflow-common > image` 를 위에서 지정한 이미지 이름으로 변경


### Custom 이미지를 생성한 이유
1. Python 라이브러리 종속성을 `requirements.txt` 같은 형태로 관리하기 위해서
2. 베이스 이미지만 변경함으로써 언제든지 버전 업그레이드 혹은 다운그레이드를 할 수 있다.

### build & compose up 스크립트
```shell
# build.sh
#!/bin/bash

image=airflow:2.9.3

docker build . -t $image &&
docker compose up
```
- `sh build.sh` 로 편하게 환경을 셋업할 수 있다.


## DAG 코드 예시
```python
import json
import os

import airflow
import requests
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

raw_data_path = '/tmp/launches.json'
image_dir = '/tmp/images'

# DAG 생성
dag = DAG(
    dag_id='download_rocket_launches',  # Webserver 에서 표시되는 dag 이름
    start_date=airflow.utils.dates.days_ago(14),  # 최초 dag 실행 시간
    schedule_interval=None  # dag 스케쥴
)

download_launches = BashOperator(
    task_id='download_launches',
    bash_command=f'curl -o {raw_data_path} -L "https://ll.thespacedevs.com/2.2.0/event/upcoming"', dag=dag
)


def _get_pictures():
    # 이미지 디렉토리 생성
    if not os.path.exists(image_dir):
        os.mkdir(image_dir)

    # 이미지 다운로드
    with open(raw_data_path) as f:
        launches = json.load(f)
        image_urls = [launch['feature_image'] for launch in launches['results']]
        for image_url in image_urls:
            try:
                response = requests.get(image_url)
                image_file_name = image_url.split('/')[-1]
                target_file = f'{image_dir}/{image_file_name}'
                with open(target_file, 'wb') as f:
                    f.write(response.content)
                print(f'Downloaded {image_url} to {target_file}')
            except requests.exceptions.MissingSchema:
                print(f'{image_url} appears to be an invalid URL')
            except requests.exceptions.ConnectionError:
                print(f'Could not connect to {image_url}')


get_pictures = PythonOperator(
    task_id='get_pictures',
    python_callable=_get_pictures, # 실행할 파이썬 함수
    dag=dag,
)


notify = BashOperator(
    task_id='nofity',
    bash_command=f'echo "There are now $(ls {image_dir} | wc - l) images."'
)


download_launches >> get_pictures >> notify
```
- `Schedule_interval`을 None으로 했기 때문에 자동으로 실행되지 않고 수동으로만 실행된다.
- 각 Operator에는 속한 DAG를 명시해준다.
- 각 `task_id`은 관리의 편의성을 위해 Operator 이름으로 지정한다.
- 테스크의 의존성 설정은 `>>(오른쪽 시프트 연산자)`을 이용한다.


## Task와 Operator 차이
- Operator는 단일 작업을 수행하는 역할로, 특정 목적을 위해 사용된다.
	- Ex) `BashOperator`, `PythonOperator`, `EmailOperator` 등
- DAG는 Operator 들의 실행에 대한 오케스트레이션 역할
	- Operator 시작, 정지, 다음 작업 시작, 의존성 보장 등
- Task는 Operator의 Wrapper 같은 개념으로, Webserver에는 Task id로 작업이 표시된다.


## Airflow Webserver
### DAGs
![[dags 2.9.3.png]]

- Meta Database에 적재된 DAG 목록들을 확인할 수 있다.
	- 최근 실행 시간, 다음 실행 시간, DAG 이름, DAG 소유자, 최근 실행 결과 등
- tag로 Dag를 검색할 수 있고, Dag 이름으로도 검색이 가능하다.

### DAGs Detail
![[dag detail 2.9.3.png]]
- Dag의 실행 결과를 상세하게 볼 수 있다.
	- 실행시간, 큐에 들어간 시간, Dag 실행 시간, Dag 종료 시간 등
- 좌측 그래프로 최근 실행 결과를 간단하게 확인할 수 있다.

### Dags graph
![[dags graph 2.9.3.png]]
- 태스크의 의존성 및 최근 실행 결과를 확인할 수 있다.

### Dags Calendar
![[calendar 2.9.3.png]]
- 월별로 Dag 실행 결과를 한눈에 볼 수 있다.


### Task log
![[task log 2.9.3.png]]
- 각 Task 별로 디테일한 정보를 확인할 수 있으며, 특히 Logs 탭에서는 실행 결과를 보여준다.


UI 와 관련된 정보: https://airflow.apache.org/docs/apache-airflow/stable/ui.html#ui-screenshots