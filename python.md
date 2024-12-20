JFrog을 통해 의존성과 아티팩트를 관리하는 경우, Python 패키지와 Lambda 배포 파일을 JFrog Artifactory에 업로드하는 단계를 추가할 수 있습니다. 이에 따라 전체 폴더 구조와 파일 내용을 아래와 같이 업데이트했습니다.

---

## 1. **최종 폴더 트리**

```plaintext
my-python-app/
├── lambda_function.py         # Lambda 핸들러 파일 (루트에 위치)
├── setup.py                   # setuptools 설정 파일
├── pyproject.toml             # PEP 518 기반 설정 파일
├── requirements.txt           # 의존성 목록
├── jfrog_upload.sh            # JFrog 아티팩트 업로드 스크립트
├── buildspec.yml              # CodeBuild 빌드 스펙 파일
├── pipeline.yml               # CodePipeline 정의 파일 (옵션)
├── tests/                     # 테스트 코드 디렉토리
│   ├── __init__.py
│   └── test_lambda.py         # Lambda 테스트 파일
└── README.md                  # 프로젝트 설명 파일
```

---

## 2. **각 파일 내용**

### **1) `lambda_function.py`**
Lambda 핸들러 파일입니다.

```python
import json
import numpy as np
import matplotlib.pyplot as plt

def lambda_handler(event, context):
    # NumPy 예제
    array = np.array([1, 2, 3, 4, 5])
    squared = array ** 2

    # Matplotlib 예제 (이미지 생성)
    plt.plot(array, squared)
    plt.title("NumPy & Matplotlib in AWS Lambda")
    plt.savefig("/tmp/plot.png")  # Lambda는 /tmp 디렉토리만 쓸 수 있음

    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": "Hello from AWS Lambda!",
            "squared_numbers": squared.tolist()
        })
    }
```

---

### **2) `setup.py`**
`setuptools`를 사용해 빌드하고, 결과물을 JFrog Artifactory에 업로드할 수 있습니다.

```python
from setuptools import setup, find_packages

setup(
    name="my-python-app",
    version="0.1.0",
    packages=find_packages(),
    install_requires=[
        "numpy",
        "matplotlib",
    ],
)
```

---

### **3) `pyproject.toml`**
Python 프로젝트 메타데이터를 정의합니다.

```toml
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.build_meta"
```

---

### **4) `requirements.txt`**
Lambda에서 사용할 의존성을 관리합니다.

```plaintext
numpy
matplotlib
```

---

### **5) `jfrog_upload.sh`**
JFrog에 빌드 결과를 업로드하는 스크립트입니다.

```bash
#!/bin/bash

# 환경 변수 설정
JFROG_URL="https://your-company.jfrog.io/artifactory"
REPO_NAME="lambda-artifacts"
PACKAGE_NAME="lambda_function.zip"

# JFrog CLI 설치 확인
if ! command -v jf &> /dev/null
then
    echo "JFrog CLI not found. Installing..."
    curl -fL https://getcli.jfrog.io | sh
    mv jf /usr/local/bin/
fi

# 아티팩트 업로드
jf rt u $PACKAGE_NAME $REPO_NAME/$PACKAGE_NAME \
    --url $JFROG_URL \
    --user $JFROG_USER \
    --password $JFROG_PASSWORD

echo "Upload complete!"
```

---

### **6) `buildspec.yml`**
CodeBuild 빌드 스펙 파일로 JFrog 업로드 및 S3 업로드를 포함합니다.

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.13
    commands:
      - echo "Installing dependencies and build tools..."
      - pip install --upgrade pip setuptools wheel
  pre_build:
    commands:
      - echo "Installing application dependencies..."
      - pip install numpy matplotlib -t ./build  # 종속성 설치 후 빌드 디렉토리에 복사
  build:
    commands:
      - echo "Packaging Lambda function..."
      - cp lambda_function.py ./build  # Lambda 핸들러 복사
      - cd build && python ../setup.py sdist bdist_wheel  # setup.py로 패키지 빌드
      - zip -r ../lambda_function.zip .  # ZIP 아카이브 생성
  post_build:
    commands:
      - echo "Updating Lambda function..."
      - aws lambda update-function-code \
          --function-name my-python-lambda \
          --zip-file fileb://lambda_function.zip
artifacts:
  files:
    - lambda_function.zip
  discard-paths: yes

```

---

### **7) `tests/test_lambda.py`**
Lambda 핸들러를 테스트합니다.

```python
import json
from lambda_function import lambda_handler

def test_lambda_handler():
    event = {}
    context = {}
    response = lambda_handler(event, context)
    body = json.loads(response["body"])

    assert response["statusCode"] == 200
    assert "message" in body
    assert "squared_numbers" in body
    assert body["squared_numbers"] == [1, 4, 9, 16, 25]
```

---

### **8) `README.md`**
프로젝트 설명 문서입니다.

```markdown
# My Python App for AWS Lambda

This project demonstrates how to use NumPy and Matplotlib in AWS Lambda with CI/CD integration using AWS CodePipeline, CodeBuild, and JFrog.

## Requirements
- Python 3.9+
- AWS CLI
- JFrog CLI

## Installation
```bash
pip install -r requirements.txt
```

## Build and Deploy
1. Run the `buildspec.yml` with AWS CodeBuild.
2. Artifacts will be uploaded to:
   - JFrog: `<JFROG_URL>/lambda-artifacts/lambda_function.zip`
   - S3: `s3://my-lambda-artifact-bucket/lambda_function.zip`

## Testing
```bash
pytest
```
```

---

## 3. **CI/CD 흐름 요약**

1. **GitHub Source 연결**:
   - CodePipeline의 Source 단계에서 GitHub 리포지토리를 연결합니다.

2. **CodeBuild 빌드**:
   - 의존성을 설치하고 Lambda 패키지를 생성한 뒤, JFrog과 S3에 업로드합니다.

3. **Lambda 배포**:
   - CodePipeline의 Deploy 단계에서 Lambda 함수에 S3의 아티팩트를 연결합니다.

---

이 구조는 JFrog와 AWS Lambda 배포를 최적화합니다. 추가적인 요청이 있으면 말씀해주세요! 😊
