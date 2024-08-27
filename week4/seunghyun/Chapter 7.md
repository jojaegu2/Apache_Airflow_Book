---
Chapter: 외부 시스템과 통신하기
---
## 외부 시스템
> Airflow 및 Airflow가 구동되는 시스템 이외의 모든 기술을 의미
> ex) Apache Spark 클러스터, BigQuery, AWS S3 등

### 실습 사항
1. AWS S3 버킷 및 머신러닝 모델을 개발 및 배포하기 위한 AWS SageMaker를 사용한 머신러닝 모델 개발
2. Airbnb [공개 데이터](https://insideairbnb.com) 로부터 숙소와 리뷰 및 기타 기록 데이터를 AWS S3 버킷으로 저장하고 Pandas로 작업을 실행하여 가격 변동을 확인하하여 AWS S3 버킷에 다시 저장

## 클라우드 서비스에 연결하기
- 클라우드 프로바이더에서 제공하는 API를 통해 클라우드 서비스를 제어할 수 있다.
- 클라우드 프로바이더에서 제공하는 제공하는 파이썬 클라이언트
	- AWS: [Boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html)
	- GCP: [Cloud SDK](https://cloud.google.com/python/docs/reference)
	- Azure: [SDK for Python](https://learn.microsoft.com/ko-kr/azure/developer/python/sdk/azure-sdk-overview)
- 클라이언트는 요청에 필요한 세부 정보를 입력하면 클라이언트가 요청 및 응답 처리를 내부적으로 처리하는 편리한 기능을 제공
- Airflow 오퍼레이터는 클라우드 서비스에 요청하는 세부 정보를 전달하는 인터페이스로 기술적인 구현을 오퍼레이터 내부에서 처리한다.
- 오퍼레이터는 **내부적으로 클라우드의 SDK를 사용해 요청을 보내고 사용자에게 특정 기능을 제공하는 클라우드 SDK를 감싼 레이어를 제공**한다.
![[Airflow - Cloud 오퍼레이터.png]]

> [!info]  SDK와 API의 차이
> 
> SDK는 **Software Development Kit**의 약자로, 특정 소프트웨어 패키지, 프레임워크, 하드웨어 플랫폼 등을 개발하기 위한 도구 모음이다.  소프트웨어 개발을 위한 도구로 주로 새로운 애플리케이션을 개발하는데 사용된다.
> 	예시)  안드로이드 앱 개발을 위한 자바 SDK, IOS 개발을 위한 Swift SDK 등
> API는 **Application Programming Interface**의 약자로, 소프트웨어 간의 상호작용을 할 수 있게 하는 인터페이스이다. 소프트웨어 연결 및 통합을 목적으로 애플리케이션에 특정 기능을 추가하는데 사용된다.
> 	예시)  Google Translation API, OpenAPI 등

### 추가 의존성 패키지 설치
- 클라우드 프로바이더를 위한 Airflow 패키지

| 클라우드 프로바이더 | 패키지                                      |
| ---------- | ---------------------------------------- |
| AWS        | apache-airflow-providers-amazon          |
| GCP        | apache-airflow-providers-google          |
| Azure      | apache-airflow-providers-microsoft-azure |
- [다른 오퍼레이터 및 의존성 패키지 정보](https://airflow.apache.org/docs/apache-airflow-providers/packages-ref.html)

AWS operator 이용해보기 (`S3ListOperator`)
1. access key, secret key로 aws connection 생성하기
	- [관련 문서](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/connections/aws.html)
![[Airflow - AWS Connection.png]]
```python
from airflow import DAG
from airflow.providers.amazon.aws.operators.s3 import S3ListOperator
from airflow.operators.python import PythonOperator


def _print_s3_list(**context):
    print(f"s3 list: {context['task_instance'].xcom_pull(key='return_value')}")


with DAG(
    dag_id='chapter07_01_s3_example',
    schedule=None,
    start_date=None,
) as dag:

    get_s3_files = S3ListOperator(
        task_id='get_s3_files',
        bucket='seunghyun-pipeline-storage', # 버킷 이름
        aws_conn_id='aws_conn' # 위에서 생성한 Connection ID
    )

    print_s3_list = PythonOperator(
        task_id='print_s3_list',
        python_callable=_print_s3_list,
    )

    get_s3_files >> print_s3_list
```
- `S3ListOperator` 출력 예시
![[Airflow - S3ListOperator return value.png]]

### 머신러닝 모델 개발하기
- 손글씨 숫자 분류기 구현을 위한 데이터 파이프라인 개발을 여러 AWS 오퍼레이터로 제작
	- 손글씨 숫자 분류기 구현에서 사용할 모델은 [MNIST](https://yann.lecun.com/exdb/mnist/)
- 모델은 학습된 후 입력되지 않은 손으로 쓴 새로운 번호를 입력할 수 있어야 하고, 모델은 입력된 손글씨 번호를 분류할 수 있어야 한다.
![[Airflow - MNIST example.png]]
- 모델은 오프라인과 온라인이 존재
	- **오프라인**: 손으로 쓴 숫자의 큰 데이터 세트를 가져와서 숫자로 분류하도록 학습시키고 결과가 저장. 이 프로세스는 새로운 데이터가 수집될 때마다 주기적으로 수행
	- **온라인**: 모델을 로드하고 이전에 학습하지 않았던 데이터를 숫자로 분류하는 역할. 사용자가 직접 확인을 요구할 때 즉시 실행되어야 한다.
- Airflow 워크플로는 일반적으로 오프라인을 담당한다.
	- 모델 학습 과정은 데이터를  로드하고 모델에 적합한 형식으로 전처리하며 모델을 학습하는 것으로 구성
	- 주기적으로 모델을 재학습하는 것이 Airflow의 배치 프로세스 기능에 잘 맞아떨어진다.
- 온라인은 일반적으로 REST API 호출 또는 HTML 내에서 REST API 호출을 포함하는 웹페이지이다.
	- API 배포는 한 번 또는 CI/CD 파이프라인 일부로 포함된다.

#### MNIST 모델 학습을 위한 Airflow 파이프라인
1. 샘플 데이터를 S3에 버킷에 복사
2. 데이터를 모델에서 사용할 수 있는 형식으로 변환
3. SageMaker에서 모델을 학습
4. 모델을 배포
5. 새로운 입력되는 손글씨 숫자를 분류
![[Airflow - MNIST pipeline.png]]
- 위의 파이프라인은 한 번만 실행할 수 있으며 SageMaker 모델 또한 한번만 배포된다.
- Airflow의 장점은 파이프라인을 스케쥴하고, 원하는 경우 새 데이터나 모델 변경시 파이프라인을 재실행할 수 있다.
- 로우 데이터가 지속적으로 업데이트되는 경우, Airflow의 파이프라인은 주기적으로 로우 데이터를 다시 로드하고 새로운 데이터를 이용해 훈련된 모델을 재배포한다.
- 수동으로 모델을 조정할 수 있으며, 파이프라인은 수동으로 트리거할 필요없이 모델을 자동으로 재배포할 수 있다.
- AWS 서비스가 지속적으로 추가, 변경 또는 제거되기 때문에 AWS 오퍼레이터가 완벽하게 커버하지는 않지만 대부분은 오퍼레이터를 지원한다.
![[Airflow - MNIST pipeline dag.png]]

##### download_mnist_data
```python
    download_mnist_data = S3CopyObjectOperator(
        task_id='download_mnist_data',
        source_bucket_name='sagemaker-sample-data-eu-west-1',
        source_bucket_key='algorithms/kmeans/mnist/mnist.pkl.gz',
        dest_bucket_name=S3_BUCKET_NAME,
        dest_bucket_key=RAW_DATA,
        aws_conn_id='aws_conn',
    )
```
- `sagemaker-sample-data-eu-west-1` 퍼블릭 버킷에 `algorithms/kmeans/mnist/mnist.pkl.gz`경로에 있는 MNIST 데이터 세트를 자신의 버킷으로 복사
- AWS의 경우 액세스 키를 사용해 사용자의 개발 시스템에서 클라우드 리소스를 액세스할 수 있기 때문에 로컬에서 Airflow 태스크를 실행할 수 있다.
	- `airflow tasks test <dag_id> <task_id> <logical_date>`
![[Airflow - Task test.png]]
- 실행 결과
![[Airflow - Install mnist dataset.png]]

> [!info] AWS 오퍼레이터 내부에 사용되는 AWS boto3 클라이언트는 태스크가 실행되는 시스템에 대한 인증을 확인한다. 그러기 위해서는 AWS 자격 증명 구성해야 한다.
> - 로컬 자격증명을 airflow 환경에서 사용하는 방법
> 	- `~/.aws/credentials` 경로에 추가
> - export로 환경 변수 추가
> 	- Connection을 추가하고 각 오퍼레이터에 aws_conn_id를 지정하는 방법
```text
~/.aws/credentials
...
[myaws]
aws_access_key_id=<access_key>
aws_secret_access_key=<secret_access_key>

$ export AWS_PROFILE=myaws
$ export AWS_DEFAULT_REGION=<region_id>
```

##### extract_mnist_data
```python
def _extract_mnist_data():
    s3hook = S3Hook(aws_conn_id='aws_conn')  # S3와 통신하기 위해 S3Hook 초기화 (aws connection 전달)

    # S3의 데이터 세트를 메모리에 다운로드
    mnist_buffer = io.BytesIO()
    mnist_obj = s3hook.get_key( 
        bucket_name=S3_BUCKET_NAME,
        key=RAW_DATA,
    )  # 메모리 내 바이너리 스트림으로 데이터를 다운로드
    mnist_obj.download_fileobj(mnist_buffer)

    # gzip 파일의 압축을 풀고 데이터 세트를 추출, 변환 후 S3로 다시 데이터를 업로드
    mnist_buffer.seek(0)
    with gzip.GzipFile(fileobj=mnist_buffer, mode='rb') as f: # 압축 해제 및 피클링 해제
        train_set, _, _ = pickle.loads(f.read(), encoding='latin1')
        output_buffer = io.BytesIO()
        write_numpy_to_dense_tensor( # Numpy 배열을 RecordIO 레코드 포맷으로 변환F
            file=output_buffer,
            array=train_set[0],
            labels=train_set[1]
        )
        output_buffer.seek(0)
        s3hook.load_file_obj(
            output_buffer,
            key='mnist_data',
            bucket_name=S3_BUCKET_NAME,
            replace=True
        )  # 추출한 데이터를 S3로 다시 업로드


...
extract_mnist_data = PythonOperator(
        task_id='extract_mnist_data',
        python_callable=_extract_mnist_data
    )
```
- 파일을 자신의 S3 버킷에 복사한 후에 SageMaker KMeans 모델의 입력 형식인 RecordIO 포맷으로 변환
- Airflow는 학습을 위한 피처 세트를 관리할 수 있는 범용 오케스트레이션 프레임워크지만 데이터 분야에서 작업하기 위해서는 필요한 모든 기술에 대해 파악하고 무엇을 어떻게 연결하고 어떻게 사용해야하는지는 시간과 경험이 필요하다.
- 데이터를 인메모리 바이너리 스트림(In-memory binary stream)으로 다운로드하기 때문에 데이터가 파일 시스템에 저장되지 않아 작업 후 파일이 남아 있지 않는다.
	- MNIST 데이터는 15MB로 작기 때문에 어떤 시스템에도 동작하지만 대규모 데이터의 경우 데이터를 디스크에 저장하고 청크로 처리하는 것이 더 좋다.
- 테스트 결과
![[Airflow - Install mnist dataset.png]]


##### sagemaker_train_model
```python
sagemaker_train_model = SageMakerTrainingOperator(
	task_id='sagemaker_train_model',
	config={
		'TrainingJobName': (
			'mnistclassifier-{{ logical_date.strftime("%Y-%m-%d-%H-%M-%S)}}'
		),
	},
	wait_for_completion=True,
	print_log=True,
	check_interval=10,
)
```
- [SageMaker 설정 관련 문서](https://docs.aws.amazon.com/sagemaker/latest/dg/gs.html?icmpid=docs_sagemaker_lp/index.html)
- 외부 시스템인 SageMaker 연계해 작업해 봄으로써 유용한 내용
	- AWS `TrainingJobName`은 AWS 계정과 리전 내에서 고유하다. 동일한 이름의 `TrainingJobName`으로 오퍼레이터를 두 번 실행하면 오류가 발생한다. `logical_date` 같은 템플릿을 이용해 태스크의 멱등성을 유지하고 충돌을 방지한다.
	- `wait_for_completion` 및 `check_interval` 인수를 주의해서 사용해야 한다. `wait_for_completion` 값이 False로 설정되어 있으면 단순히 명령을 실행만 하고 그 외의 것은 신경 쓰지 않는다. AWS는 학습 작업을 시작하지만 학습 작업의 완료 여부를 학인할 수 없다. 따라서, True인 경우 SageMaker 오퍼레이터는 주이전 작업이 완료될 때까지 기다린다.` check_interval`에서 지정한 초마다 태스크가 실행 중인지 확인한다. 
- 모델 배포가 완료된 SageMaker 대시보드 화면
![[Pasted image 20240827210355.png]]

##### SageMaker 모델 접근하기
- SageMaker의 엔드포인트는 AWS API 등을 통해 접근할 수 있지만, 외부에서 접근할 수 없다.
- AWS에서 인터넷에 접근하기 위해서는 엔드포인트를 트리거하는 AWS Lambd 서비스를 개발 및 배포하고 API Gateway를 생성 및 연결해 외부에서 접속할 수 있는 HTTP 엔드포인트를 만든다.
- 모델 접근을 위한 인프라 구성은 주기적으로 배포하지 않고 한 번만 배포하면 되기 때문에 파이프라인에 통합하지 않고 별도로 진행
- AWS Chalice를 사용하는 사용자 API 예제
```python
import json
from io import BytesIO

import boto3
import numpy as np
from PIL import Image
from chalice import Chalice, Response
from sagemaker.amazon.common import numpy_to_record_serializer

app = Chalice(app_name='number-classifier')

@app.route('/', methods=['POST'], content_types=['image/jpeg'])
def predict():
	img = Image.open(BytesIO(app.current_request.raw_body)).convert('L')
	img_arr = np.array(img, dtype=np.float32)
	runtime = boto3.Session().client(
		service_name='sagemaker-runtime',
		region_name='<Your region>',
	)

	response = runtime.invoke_endpoint(. # SageMaker 엔드포인트 호출
		EndpointName='mnistclassifier',
		ContentType='application/x-recordio-protobuf',
		Body=numpy_to_record_serializer()(img_arr.flatten())
	)
	result = json.loads(response['Body'].read().decode('utf-8'))  # 결과 수신
	return Response(
	result, status_code=200, headers={'Content-Type': 'application/json'})
```
- API는 주어진 이미지를 RecordIO 형식으로 변환한다.
- RecordIO 객체를 airflow 파이프라인에 의해 배포된 SageMaker의 엔드포인트로 전달된 후 입력된 이미지의 에측 결과를 반환
```bash
# API 호출 예시
curl --request POST \
	--url <Your URL> \
	--header 'content-type: image/jpeg' \
	--data-binary @'/path/to/image.jpeg'
```

![[Pasted image 20240827211242.png]]


## 시스템 간 데이터 이동하기
- airflow의 일반적인 사용 사례는 정기적인 ETL 작업으로, 매일 데이터를 다운로드하고 다른 곳에서 반환한다.
- 프로덕션 데이터베이스에서 데이터를 내보내고 나중에 처리할 수 있도록 준비하는 등의 분석 목적으로 사용된다.
- 프로덕션 데이터베이스는 대부분 과거 이력 데이터를 반환할 수 없다. 따라서, 주기적으로 데이터를 내보내고 저장해 나중에 처리할 수 있도록 한다.
- 기존 데이터 덤프에 스토리지 저장 공간은 빠르게 증가하게 되고, 모든 데이터를 처리하기 위해서는 분산 처리가 필요하다.

### 예시
![[Pasted image 20240827211648.png]]
1. PostgresSQL 컨테이너에서 Airbnb 데이터 접근
2. 쿼리를 통해 데이터를 추출하고 S3에 적재
3. 도커 컨테이너에서 S3에 적재된 데이터를 읽고 Pandas 를 사용해 처리
4. S3에 결과 저장

- 두 시스템 간의 데이터 전송은 airflow의 일반적인 태스크로 작업 간에 데이터 변환 작업이 있을 수 있다.
	- MySQL 데이터베이스를 쿼리하여 결과를 GCS에 저장
	- STFP 서버에서 AWS S3 데이터 레이크로 데이터를 복사
	- HTTP REST API를 호출하여 출력을 저장
- airflow 에코시스템에서 이를 위한 오퍼레이터가 존재한다.
	- MySqlToGoogleCloudStorageOperator
	- SFTPToS3Operator
	- SimpleHttpOperator

### PostgresToS3Operator 구현하기
- MongoToS3Operator
	- MongoDB 데이터베이스에 쿼리를 실행하고 그 결과를 AWS S3 버킷에 저장하는 오퍼레이터
	![[Pasted image 20240827212535.png]]
	- 위의 오퍼레이터는 Airflow가 동작하는 시스템의 파일 시스템을 사용하지 않고 모든 결과를 메모리에 보관
		- 쿼리 결과가 매우 크면 airflow 시스템에서 사용 가능한 메모리가 부족질 수 있다.
- S3ToSFTPOperator
	![[Pasted image 20240827212656.png]]
	- SSHHook 및 S3 Hook 두 가지 훅을 인스턴스화
	- 중간 결과를 airflow 인스턴스의 로컬 파일 시스템의 임시 위치에 `NamedTemporaryFile`을 통해 저장
	- 전체 결과를 메모리에 적재하지 않기 때문에 디스크에 충분한 공간이 있는지 확인
- `MongoToS3Operator`와 `S3ToSFTPOperator`는 공통적으로 두 개의 훅을 가지고 있습니다.
	- 하나는 A와 통신하고 다른 하나는 시스템 Bdhk xhdtlsgksek
	- A와 B 간의 데이터를 검색하고 전송하는 방법은 다르며, 특정 오퍼레이터를 구현하는 사람에 따라 달라질 수 있다.
- Postgres의 경우, 데이터베이스 커서는 결과 청크를 가져오고 업로드하기 위해 반복 작업이 필요할 수 있다.
![[Pasted image 20240827212944.png]]
- `S3Hook`은 `aws_conn_id`로 초기화
	- `S3Hook`은 `AWSHook`의 서브클래스로 몇몇 메소드와 속성을 상속한다.
- `PostgresHook`은 `DbApiHook`의 서브 클래스로 postgres_conn_id로 초기화
	- 쿼리를 실행하고 `get_records`로 쿼리 결과를 받는다. 결과 값은 튜플 목록으로 받는다.
	- 결과를 문자열화하고 지정한 버킷과 키 정보를 가지고 인코딩된 데이터를 AWS S3에 적재하는 `load_string()`을 호출

- 데이터가 튜플 형태로 되어 있어 CSV나 Json과 같은 파일로 처리할 수 없기 때문에 CSV 파일 형태로 변환하여 S3에 적재
	- CSV 파일로 변환하면 Pandas나 Spark과 같은 데이터 처리 프레임워크가 출력 데이터를 쉽게 출력할 수 있다.
![[Pasted image 20240827213459.png]]
- 버퍼는 메모리에 있으며 처리 후 파일 시스템에 남기지 않기 때문에 편리
	- Postgres 쿼리의 출력값은 메모리에 위치하기 때문에 쿼리 결과의 크기를 메모리에 맞춰야 한다.
- 멱등성을 보장하기 위해 `replace=True`로 지정한다.


## 큰 작업을 외부에서 수행하기
- 만일 airflow의 모든 리소스를 사용하는 매우 큰 작업을 실행해야 한다고 하면, 작업은 다른 곳에서 수행하고 airflow는 작업이 시작되고 완료될때까지 대기하는 것이 좋다. 오케스트레이션과 실행이 완벽하게 분리되어 있어야 하고, airflow에 의해서 작업은 시작되지만 apache Spark와 같은 데이터 처리 프레임워크가 실제 작업을 수행하고 완료될 때까지 기다린다.
- Spark로 작업을 실행하는 방법
	- `SparkSubmintOperator` 사용하기: airflow 머신에서 spark 인스턴스를 찾기 위해 spark-submit 파일과 yarn 클라이언트 구성이 필요하다.
	- `SSHOperator` 사용하기: Spark 인스턴스에 대한 SSH 액세스가 필요하지만 airflow 인스턴스에 대한 spark 클라이언트 구성이 별도로 필요하지 않다.
	- `SimpleHTTPOperator` 사용하기: Apache Spark용 REST API인 Livy를 실행