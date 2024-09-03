---
Chapter: 컨테이너에서 태스크 실행하기
---
## 다양한 오퍼레이터를 사용할 때 고려해야 할 점
- 오퍼레이터는 다양한 유형의 시스템에서 작업을 조정할 수 있는 뛰어난 유연성을 제공하는 airflow의 강력한 기능 중 하나이다.
	- 여러 오퍼레이터가 있는 DAG는 복잡하기 때문에 만들고 관리하는 것이 어렵다.
- 영화 추천 DAG에서는 세 가지의 태스크로 구성되어 있다.
	- 영화 API에서 영화 평점 데이터를 가져오고
	- 가져온 평점 데이터에 따라 영화의 순위를 지정하고
	- 데이터를 MySQL 데이터베이스로 푸시하여 다운스트림에서 사용한다.
- 영화 추천 DAG에서는 API에 접근하기 위한 `HttpOperator`, 파이썬 평점에 따라 순위를 지정할 `PythonOperator`, 결과를 저장하기 위한 `MySQLOperator`를 사용한다.

### 오퍼레이터 인터페이스 및 구현하기
- 각 태스크에서 서로 다른 오퍼레이터를 사용할 때 어려운 점은 효율적인 구성을 위해 각 오퍼레이터별로 인터페이스와 내부 동작에 익숙해져야 한다는 것입니다. 또한 오퍼레이터에서 버그가 발생하면 근본적인 문제를 추적하고 수정하는데 시간과 리소스를 투자해야 한다.
- 다수의 서로 다른 오퍼레이터를 사용한 다양한 DAG로 airflow가 구성되어 있다고 하면, 모든 오퍼레이터를 함께 작업 구성하는 것은 어려울 수 있다.

### 복잡하며 종속성이 충돌하는 환경
- 다양한 오퍼레이터를 사용할 때 또 다른 어려운 점은 오퍼레이터마다 각각의 종속성을 요구한다는 것이다.
	- `HttpOperator`는 HTTP 요청을 수행하기 위해 파이썬 라이브러리인 `request`에 종속적인 반면, `MySQLOperator`는 MySQL과 통신하기 위해 파이썬 및 시스템 레벨에서 종속성을 갖는다.
	- `PythonOperator`가 호출하는 코드는 자체적으로 많은 종속성(사용하는 패키지 등)을 가질 수 있다.
- airflow 설정 방식 때문에 모든 종속성을 airflow 스케쥴러를 실행하는 환경뿐만 아니라 airflow 워커 자체에도 설치되어야 한다.
- 다양한 오퍼레이터를 사용할 때는 다용한 종속성을 위한 많은 모듈이 설치되어야 하기 때문에 잠재적인 충돌이 발생하고 환경 설정 및 유지 관리가 상당히 복잡해진다.
	- 파이썬은 동일한 환경에 동일한 패키지의 여러 버전을 설치할 수 없기 때문에 더욱 문제가 된다.
![[airflow - 종속성 충돌.png|500]]

### 제너릭 오퍼레이터 지향하기
- 다양한 오퍼레이터와 그 종속성을 사용하고 유지하는 것이 어렵기 때문에, airflow 태스크를 실행하기 위해 하나의 제너릭 오퍼레이터를 사용하는 것이 더 낫다는 주장을 한다.
	- 장점으로는 한 종류의 오퍼레이터만 익숙해지면 된다이다. 즉, 다양한 airflow DAG는 한 가지 유형의 태스크로만 구성되기 때문에 이해하기가 훨씬 쉬어진다. 또한 모든 사용자가 동일한 오퍼레이터를 사용하여 태스크를 실행하면, 많이 사용되는 오퍼레이터에서는 그만큼 버그가 발생할 가능성이 적어진다. 그리고 단일 오퍼레이터에 필요한 하나의 airflow 종속성 집합만 관리하면 된다.

## 컨테이너
- 컨테이너는 애플리케이션에 필요한 종속성을 포함하고 서로 다른 환경에서 균일하게 쉽게 배포할 수 있는 기술
- 각각에 대한 종속성 패키지를 설치하고 관리하지 않고도 동시에 다양한 태스크를 실행을 가능하게 하는 것이 컨테이너이다.

### 컨테이너란 무엇인가?
- 과거 소프트웨어 애플리케이션을 개발할 때 가장 큰 문제 중 하나는 배포였다. 배포할 때에는 일반적으로 운영 체제 간의 차이, 설치된 종속성 및 라이버리의 변형, 하드웨어의 차이 등을 포함하여 다양한 요소를 조정 및 고려해야 한다.
- 가상화를 통해 이러한 복잡성을 관리할 수 있다. 가상화는 클라이언트 운영 체제 위에서 실행되는 가상 머신에 애플리케이션을 설치하는 것으로, 애플리케이션은 가상 OS 만 바라보게 된다. 즉, 호스트 OS를 수정하는 대신에 가상 OS가 애플리케이션의 요구사항을 충족하는지 확인하면 된다. 따라서 애플리케이션을 구현하기 위해 필요한 종속성을 가진 애플리케이션을 가상 OS에 설치 후 배포할 수 있다.
- 가상 머신의 단점은 호스트 OS 위에서 게스트 OS를 실행해야 하기 때문에 상당히 무겁다. 또한 모든 가상머신은 자체 게스트 OS를 실행하므로, 단일 시스템에서 여러 개의 VM 애플리케이션을 실행하려면 매우 큰 리소스가 필요하다
- 컨테이너는 가상 머신과 달리 호스트 OS의 커널 레벨의 기능을 사용하여 애플리케이션을 가상화하기 때문에 훨씬 가볍다. 즉, 컨테이너는 가상 머신과 동일한 방식으로 애플리케이션 및 종속성을 분리할 수 있지만, 각 애플리케이션을 위한 자체 게스트 OS를 구동할 필요 없이 호스트 OS에서 간단하게 활용할 수 있다.
- 컨테이너 엔진은 컨테이너와 호스트 OS 간의 상호 작용을 관리하고 실행하기 위한 서비스이다. 컨테이너 엔진에서는 다양한 애플리케이션 컨테이너와 이미지를 관리하고 실행하기 위한 API를 제공한다. 또한 사용자가 컨테이너를 빌드하고 상호작용할 수 있도록 도와주는 CLI를 제공한다. 가장 잘 알려진 컨테이너 엔진은 Docker로 비교적 사용하기 쉽고 대규모 커뮤니티 운영으로 많은 인기를 얻고 있다.
![[airflow - 컨테이너.png]]

### 도커 컨테이너 실행하기
- 컨테이너 엔진으로 도커를 사용하기 위해 도커 데스크톱을 설치한다. 도커를 설치하고 실행하면 첫번째 컨테이너를 실행할 수 있다.
	- `docker run debian:buster-slim echo Hello, world!`
	- 도커 클라이언트는 도커 데몬에 접속
	- 도커 데몬은 도커 허브 레지스트리에서 기본 데비안 OS 바이너리 및 라이브러리가 포함된 데비안 도커 이미지를 가져온다.
	- 도커 데몬은 해당 이미지를 사용하여 새 컨테이너를 생성한다.
	- 컨테이너는 컨테이너 내부에서 `echo Hello, world` 명령을 실행
	- 도커 데몬은 명령에 대한 출력을 도커 클라이언트로 전송하여 터미널에 표시된다.

### 도커 이미지 생성하기
- Dockerfile 작성
```Dockerfile
FROM <Base Image>

...
```
- 도커 파일의 각 라인은 기본적으로 이미지를 작성할 때 도커에게 특정 작업을 수행하도록 지시한다.
- 기본 이미지를 `FROM` 명령으로 알려주고, 그 다음 나머지 명령(`COPY, CMD, ENTRYPOINT, ENV` 등)은 도커에게 기본 이미지에 레이어를 추가하는 방법을 알려준다.
- `docker build . -t <image name>:<tag>`로 이미지를 빌드한다.
- `docker run <image name>:<tag>`로 빌드한 이미지를 기반으로 컨테이너를 실행한다.

## 컨테이너와 Airflow
### 컨테이너 내의 태스크
- Airflow에서 컨테이너 기반 오퍼레이터(`DockerOperator`, `KubernetesPodOperator`)를 사용하여 태스크를 정의하고 태스크를 컨테이너로 실행할 수 있다. 이 오퍼레이터들은 실행되면 컨테이너 실행을 시작하고 컨테이너가 정상적으로 실행 완료될 때까지 기다린다.
- 각 태스크의 결과는 실행된 명령과 컨테이너 이미지 내부의 코드 구현에 따라 달라진다. 도커 기반 접근은 `DockerOperator`로 기존의 오퍼레이터를 대체하고 적절한 종속성을 가진 서로 다른 컨테이너에서 명령을 실행할 수 있다.

## 왜 컨테이너를 사용하는가?
1. 간편한 종속성 관리
	- 여러 태스크를 위해 서로 다른 이미지를 생성하면 각 태스크에 필요한 종속성을 해당 이미지에만 설치할 수 있다. 그런 다음 이미지 내에서 태스크가 실행되므로 태스크 간의 종속성 충돌이 발생하지 않는다. 또한 태스크가 더 이상 워커 환경에서 실행될 필요가 없기 때문에 Airflow 워커 환경에 태스크에 대한 종속성을 설치할 필요가 없다. 그러기 때문에 종속성 관리를 보다 쉽게 할 수 있다.
		![[airflow - containered task.png|400]]
2. 다양한 태스크 실행 시에도 동일한 접근 방식을 제공
	- 컨테이너화된 각 태스크가 동일한 인터페이스를 가질 수 있는 장점 때문에 태스크 실행을 컨테이너를 이용한다. 컨테이너를 통해 모두 동일한 오퍼레이터에 의해 실행되는 동일한 태스크로 운영할 수 있기 때문이다. 이미지 구성 및 실행 명령에 약간의 차이만 있을 뿐이다. 이러한 획일성을 통해 하나의 오퍼레이터만 학습한 후 DAG를 더욱 손쉽게 개발할 수 있다. 그리고 오퍼레이터에 관련 문제가 발생할 경우, 고민할 필요 없이 해당 오퍼레이터만 문제를 확인하고 수정하면 된다.
3. 향상된 테스트 가능성
	- Airflow DAG와는 별도로 개발 및 유지 관리할 수 있다는 장점이 있다. 즉, 각 태스크 이미지는 자체적으로 개발 라이프사이클을 가지며, 기대하는 대로 작동하는지 확인하는 전용 테스트 환경을 구성할 수 있다. 또한 테스크가 컨테이너로 분리되어 있기 때문에, Airflow의 오케스트레이션 계층을 분리해 확장 테스트가 가능하다. `PythonOperator`를 사용하는 경우, 긴밀하게 연결되어 있기 때문에 오케스트레이션 게층의 탄력성/확장성 테스트가 어렵다.

## 도커에서 태스크 실행하기
### DockerOperator 소개
- Airflow로 컨테이너에서 태스크를 실행하는 방법은 `apache-airflow-providers-docker` 프로바이더 패키지의 `DockerOperator`를 사용하는 것이다.
- DockerOperator 사용 예제
```python
rank_movies = DockerOperator(
	task_id='rank_movies',
	image='Movielens/movielens-ranking', # 사용할 이미지
	command=[
		'rank_movies.py',
		'--input_path',
		'/data/ratings/{{ logical_date }}.json',
		'--output_path',
		'/data/rakings/{{ logical_date }}.csv'
	], # 컨테이너에 실행할 명령
	volumes=['/tmp/airflow/data:/data'], # 컨테이너 내부에 마운트할 볼륨을 정의
)
```
- 특정한 인수를 사용하여 지정된 컨테이너 이미지를 실행하고, 컨테이너가 시작 작업을 완료할 때까지 대기하는 것이며, `docker run` 명령과 동일하다.
![[airflow - dockeroperator how to run.png]]

### 태스크를 위한 컨테이너 이미지 생성하기
- DockerOperator를 사용하여 태스크를 실행하려면 태스크에 필요한 도커 이미지를 빌드해야 한다. 주어진 태스크에 대한 이미지를 빌드하기 위해서는 태스크 실행에 필요한 코드와 종속성을 가지고 Dockerfile을 생성하고 `docker build` 명령어를 사용해 필요한 이미지를 생성한다.
1. Python으로 태스크에서 수행해야 할 내용에 대한 스크립트 작성
```python
def main(...):
	# 수행

if __name__ == '__main__':
	main(...)
```
2. Dockerfile 작성
```Dockerfile
FROM python:3.8-slim

COPY requrements.txt requrements.txt
RUN pip install -r requirements.txt

COPY scripts/<file>.py scripts/<file>.py
ENV chmod +x scripts/<file>.py
```
3. 이미지 빌드
4. `DockerOperator`로 태스크 구성하기 
```python
from airflow import DAG

from airflow.providers.docker.operators.docker import DockerOperator

with DAG(...) as dag:
	task = DockerOperator(
		task_id='task',
		image='<image name>:<tag>',
		command=[
			'<script name>', # 이미지 내 실행할 스크립트를 지정
			'--start_date',
			'{{ logical_date }}', # 템플릿으로 date를 전달
			'--user',
			os.environ['USER'], # 환경 변수를 통해 민감한 정보를 전달
			'--password',
			os.environ['PASSWORD']
		],
		volume=['local path:container path'], # 데이터를 저장할 볼륨 마운트
		network_mode='airflow', # 컨테이너가 airflow 도커 네트워크에 연결하면 API를 호출할 수 있다.(동일한 네트워크에서 실행)
	)
```

### 도커 기반 워크플로
- 도커 컨테이너를 기반으로하는 DAG를 구축하는 워크플로는 태스크를 위해 먼저 도커 이미지와 컨테이너를 만들어야 하는 점에서 다른 DAG에 사용한 접근 방식과는 차이가 있다. 
- 도커 기반 워크플로는 다음과 같다
	1. 실행할 코드와 종속성을 가지고 도커파일을 작성
	2. 도커 데몬은 개발 머신 혹은 CI/CD 환경 시스템에 해당하는 이미지 구축
	3. 도커 데몬은 이미지를 나중에 사용할 수 있도록 컨테이너 레지스트리에 게시
	4. 빌드 이미지를 참조하는 `DockerOperator`를 사용하여 DAG를 작성
	5. DAG가 활성화된 후 Airflow는 DAG 실행을 시작하고 각 실행에 대한 `DockerOperator` 태스크를 스케쥴한다,
	6. Ariflow 워커는 DockerOperator 태스크를 선택하고 컨테이너 레지스트리에서 필요한 이미지를 가져온다.
	7. 각 태스크를 위해, airflow 워커는 워커에 설치된 도커 데몬을 사용하여 해당 이미지와 인수로 컨테이너를 실행
![[airflow - dockeroperator cycle.png]]
- 도커 이미지 내에 저장된 태스크를 실행하기 위한 소프트웨어 개발과 전체 DAG 개발을 효과적으로 분리할 수 있다. 이로 인해 이미지를 자체 라이프사이클 내에 개발할 수 있으며 DAG와는 별도로 이미지를 테스트할 수 있다.

## 쿠버네티스에서 태스크 실행
- 도커는 컨테이너화된 태스크를 단일 시스템에서 실행할 수 있는 편리한 접근 방식을 제공한다. 하지만, 여러 시스템에서 태스크를 조정하고 분산하는데 도움이 되지 않기 때문에 접근 방식의 확장성이 제한된다.
- 도커의 한계로 쿠버네티스와 같은 컨테이너 오케스트레이션 시스템을 통해 클러스터 전반에 걸쳐 컨테이너화된 애플리케이션을 확장할 수 있게 된다. 

### 쿠버네티스 소개
- 쿠버네티스는 컨테이너화된 애플리케이션의 배포, 확장 및 관리에 초점을 맞춘 오픈  소스 컨테이너 오케스트레이션 플랫폼이다. 
- 쿠버네티스는 도커에 비해 컨테이너를 여러 작업 노드에 배치를 관리하여 확장할 수 있도록 지원하는 동시에 스케쥴링 시 필요한 리소스, 스토리지 및 특수한 하드웨어 요구사항 등을 고려한다.
- 쿠버네티스는 쿠버네티스 마스터(혹은 컨트롤 플레인)과 노드로 구성된다.
	- 쿠버네티스 마스터는 API 서버, 스케쥴러 및 배포, 스토리지 등을 관리하는 기타 서비스를 포함하여 다양한 컴포넌트를 실행한다.
	- 쿠버네티스 API 서버는 `kubelet` 혹은 쿠버네티스 파이썬 SDK와 같은 클라이언트에서 쿠버네티스를 쿼리하고 명령을 실행하여 컨테이너를 배포하는데 사용된다. 
	- 쿠버네티스 마스터는 쿠버네티스 클러스터에서 컨테이너화된 애플리케이션을 관리하는 것이 주요 목적이다.
	- 워커 노드는 스케쥴러가 할당한 컨테이너 애플리케이션을 실행하는 역할을 한다. 쿠버네티스에서 실행하는 애플리케이션을 파드라고 하며 단일 시스템에서 함께 실행해야하는 컨테이너가 하나 이상 포함된다.
	- airflow에서 태스크는 단일 파드 내부의 컨테이너로 실행된다.
- 쿠버네티스는 보안 및 스토리지 관리를 위한 내장된 기본 기능을 제공한다.
	- 쿠버네티스 마스터에서 스토리지 볼륨을 요청하고 이를 컨테이너 내부에 영구 스토리지로 마운트할 수 있다. 스토리지 볼륨은 도커 볼륨 마운트 작업과 유사하게 동작하지만, 쿠버네티스가 관리하는 차이가 있다. 즉, 스토리지에 대해 신경 쓸 필요 없이 간단하게 요청하고 제공된 볼륨을 사용하면 된다.
![[airflow - k8s architecture.png|600]]

### 쿠버네티스 설정하기
- 클라이언트가 로컬에 설치되어 있고 쿠버네티스 클러스터에 대한 접근 권한이 있는지 확인
- airflow 관련 리소스 및 태스크 파드가 포함될 쿠버네티스 네임스페이스 생성
	- `kubectl create namespace airflow`
- Airflow DAG의 작업 결과를 저장하기 위한 몇 가지 스토리지 리소스를 정의한 YAML 파일 생성
```YAML
apiVersion: v1
kind: PersistentVolume # 영구 볼륨을 정의하기 위한 쿠버네티스 명세, 가상 디스크로 파드에 데이터 저장 공간을 제공
metadata:
	name: data-volume # 볼륨에 할당할 이름
	Lables:
		type: local
spec:
	storageClassName: manual
	capacity:
		storage: 1Gi # 볼륨 크기
	accessModes:
		- ReadWriteOnce # 한 번에 하나의 컨테이너 읽기/쓰기 액세스를 허용
	hostPath:
		path: '/tmp/data' # 스토리지가 보관될 호스트의 파일 경로
---
apiVersion: v1
kind: PersistentVolumeClaim # 특정 볼륨 내에서 일부 스토리지를 예약해 영구 볼륨을 할당하기 위한 쿠버네티스 스펙
metadata:
	name: data-volume # 스토리지 공간에 할당할 볼륨의 이름
spec:
	storageClassName: manual
	accessModes:
		- ReadWriteOnce # 스토리지 할당을 허용하는 접근 모드
	resources:
		requests:
			storage: 1Gi # 스토리지 할당 크기
```
- 스토리지에 사용되는 두 가지 리소스를 정의
	- 첫번째는 쿠버네티스 볼륨
	- 두번째는 스토리지 할당으로 쿠버네티스에서 컨테이너에서 사용할 스토리지가 필요함을 알려준다.
	- 이 할당으로 airflow를 통해 실행하는 모든 쿠버네티스 파드에서 데이터를 저장하는데 사용할 수 있다.
- `kubectyl --namespace airflow apply -f <data-volume.yaml>` 로 스토리지 리소스 배포
- MovieLens API에 대한 배포 및 서비스 리소스를 YAML 파일로 정의
```YAML
apiVersion: apps/v1
kind: Deployment # 컨테이너 배포 생ㅇ성을 위한 쿠버네티스 명세
metadata:
	name: movielens-deployment # 배포 이름
	labels:
		app: movielens # 배포용 레이블
spec:
	replicas: 1
selector:
	matchLables:
		app: movielens
template:
	metadata:
		lables:
			app: movielens
	spec:
		containers:
			- name: movielens
			image: <movielens-api>:<tag>
			ports:
				- containerPort: 5000
			env:
				- name: API_USER
				value: airflow
				- name: API_PASSWORD
				value: airflow
---
apiVersion: v1
kind: Service
metadata:
	name: movielens
spec:
	selector:
		app: movielens
	ports:
	- protocol: TCP
	port: 80
	targetPort: 5000
```
- `kubectl --namespace airflow apply -f <resource/api.yaml>`로 API 배포하기

### KubenetesPodOperator 사용하기
- 필요한 쿠버네티스 리소스를 생성한 후 DAG를 쿠버네티스 클러스터에서 사용할 수 있다.
- 쿠버네티스에서 태스크를 실행하려면 `apache-airflow-providers-cncf-kubernetes` 에서 제공하는 `KubernetesPodOperator`를 사용
- `KubernetesPodOperator` 사용 예제
```python
task = KubernetesPodOperator(
	task_id='task',
	image='image', # 사용할 이미지
	cmd=['...'], # 컨테이너 내부에서 실행할 실행 파일
	arguments=[
		..., 
	], # 실행 파일에 전달할 인수
	namespace='airflow', # 파드를 실행할 쿠버네티스 네임스페이스
	name='task', # 파드에 사용할 이름
	cluster_context='docker-desktop', # 사용할 클러스터 이름
	in_cluster=False # 쿠버테니스 내부에서 airflow 자체를 실행하지 않음을 지정
	volumes=[volume], # 파드에서 사용할 볼륨
	volume_mounts=[volume_mount], # 파드에서 사용할 볼륨 마운트
	image_pull_policy='Never', # airflow가 도커 허브에서 이미지를 가져오는 대신 로컬에서 빌드된 이미지를 사용하도록 지정
	is_delete_operator_pod=True, # 실행이 끝나면 자동으로 파드 삭제 여부
)

# 볼륨 및 볼륨 마운트 생성
from kubernetes.client import models as k8s

...

volume_claim = k8s.V1PersistentVolumeClaimVolumeSource(
	claim_name='data-volume'
)
volume = k8s.V1Volume(
	name='data-volume',
	persistent_volme_claim=volume_claim
) # 이전에 생성된 스토리지 볼륨 및 할당에 대한 참조
volume_mount = k8s.V1VolumeMount(
	name='data-volume',
	mount_path='/data', # 볼륨을 마운트할 위치
	sub_path=None,
	read_only=False, # 쓰기가 가능한 볼륨으로 마운트
)
```
- `V1Volume` 객체는 쿠버네티스 리소스로 만든 영구 볼륨인 `data-volume`을 참조한다. 그 다음 볼륨 구성을 참조하고 파드의 컨테이너에서 이 볼륨을 마운트할 위치를 지정하는 `V1VolumeMount` 구성 객체를 생성
- `volume`과 `volume_mount`를 `KubernetesPodOperator`에 전달한다.
- DAG 구현
```python
from kubernetes.client import models as k8s

from airflow import DAG

from airflow.providers.cncf.kubernets.operators.kubernets_pod import KubernetesPodsOperator

with DAG(...) as dag:
	volume_clain = k8s.V1PersistentVolumeClaimVolumeSource(...)
	volume = k8s.V1Volume(...)
	volume_mount = k8s.V1VolumeMount(...)

	task1 = KubernetsPodOperator(...)
	task2 = KubernetsPodOperator(...)

	task1 >> task2
```


### 도커 기반 워크플로와 차이점
- 쿠버네티스 기반 워크플로와 도커 기반 접근 방식은 비교적 유사하지만 몇가지 차이점이 있다.
	- 태스크 컨테이너가 더 이상 airflow 워커 노드에서 실행되지 않고 쿠버네티스 클러스터 내에서 별도의 노드에서 실행된다. 즉, 워커에 사용되는 모든 리소스는 최소화 되며, 쿠버네티스의 기능을 사용하여 적절한 리소스가 있는 노드에 태스크가 배포되었는지 확인할 수 있다.
	- 어떤 스토리지도 더 이상 airflow 워커가 접근하지 않지만, 쿠버네티스 파드에서는 사용할 수 있어야 한다. 이를 일반적으로 쿠버네티스를 통해 제공되는 스토리지를 사용하는 것을 의미한다. 파드에서 스토리지에 대한 적절한 접근 권한이 있다면, 다른 유형의 네트워크/클라우드 스토리지를 사용할 수 있다.
- 쿠버네티스는 도커에 비해 확장성, 유연성 및 스토리지, 보안 등과 같은 기타 리소스 관리 관점에서 상당한 장점을 가지고 있고 airflow 전체를 쿠버네티스에서 실행할 수 있다. 즉, airflow 전체를 확장 가능한 컨테이너 기반 인프라에서 구동 설정이 가능하다.
![[airflow - k9s workflow.png]]