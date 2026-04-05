# 🏆 2026 AWS 실전 인프라 구축: 무결점 마스터 시트

> **현장에서 발생하는 모든 실전 변수에 대응하는 최종 가이드**  
> S3 경로 차이, Assume Role 세션 설정, fstab 상세 옵션까지 포함

---

## 📋 요구사항 체크리스트

| # | 요구사항 | 키워드 | 완료 |
|---|---------|--------|------|
| 1 | Shared network storage | EFS + S3FS fstab 영구 마운트 | ☐ |
| 2 | Query from S3 | Athena + Partition Projection | ☐ |
| 3 | Fine-grained IAM policy | ABAC + Condition + Assume Role | ☐ |
| 4 | MySQL with Lambda (**생성만**) | RDS MySQL + Lambda (VPC 내) | ☐ |

### ⚠️ 유의사항 요약

> **✏️ 이 가이드의 `<이렇게 생긴 부분>`은 전부 실제 값으로 교체해야 합니다.**  
> `< >` 꺾쇠 괄호까지 지우고 내용만 입력하세요.  
> 예) `<S3_버킷명>` → `skills-web-bucket` (괄호 없이)

- SG outbound `80/443` → Any open 유지
- Tag 정보 누락 시 채점 불가 → 리소스 생성 시 Tag 반드시 확인
- 채점은 **Cloud Shell** 기준 → 권한·접속 문제 없도록 주의
- 종료 전 테스트·부하 중지

---

## 0. 시작 전: 환경 변수 세팅

> 터미널 접속 직후 문제지에 적힌 리소스 이름을 아래 변수에 할당하세요.  
> 이후 모든 작업은 이 변수를 참조합니다. **오타 방지 1순위.**

> **🚨 주의**: 여기서 설정한 변수는 **터미널 명령어 전용**입니다.  
> AWS 콘솔(웹 UI) 입력 시에는 **실제 값을 직접 타이핑**해야 합니다.

```bash
# === [지정 리소스 이름 입력] ===
export MY_BUCKET="<S3_버킷명>"
export MY_TABLE="<Athena_테이블명>"
export MY_EFS_ID="<EFS_ID>"                                  # 콘솔에서 확인 (fs-xxxxxxxx)
export MY_ROLE_ARN="arn:aws:iam::<계정ID>:role/<역할명>"      # Assume Role 대상 ARN
export MY_REGION="<리전>"                                     # 예: ap-northeast-2
```

---

## 1. Shared Network Storage

> **요구사항 1**: EFS(공유 파일시스템) + S3(오브젝트 스토리지)를 EC2에 영구 마운트

### 1-1. EFS 마운트 헬퍼 설치

```bash
sudo yum install -y amazon-efs-utils   # Amazon Linux
# sudo apt install -y amazon-efs-utils # Ubuntu
```

### 1-2. 마운트 포인트 생성

```bash
sudo mkdir -p /efs /data
```

### 1-3. /etc/fstab 영구 마운트 설정

> **🚨 절대 주의 — fstab에 환경변수 사용 금지**  
> `fstab`은 시스템 부팅 시 로드되므로 `export`로 선언한 환경변수를 인식하지 못합니다.  
> `${MY_BUCKET}`, `${MY_EFS_ID}` 형태로 쓰면 **마운트 실패** → **부팅 장애** 유발.  
> 반드시 **실제 값을 직접 입력**하세요.

```bash
# nano 사용 추천 (vi보다 쉬움): sudo nano /etc/fstab
# vi를 써야 한다면: 입력 시작 → i  /  저장 후 종료 → Esc → :wq → Enter
sudo vi /etc/fstab
```

아래 내용을 파일 하단에 추가 (`<...>` 부분을 실제 값으로 교체, 꺾쇠 괄호도 삭제):

```
# EFS 마운트 (tls: 전송 암호화, iam: IAM 인증)
# 예시: fs-0c004845d6e9e7689:/ /efs efs _netdev,noresvport,tls,iam,nofail 0 0
<실제_EFS_ID>:/ /efs efs _netdev,noresvport,tls,iam,nofail 0 0

# S3 마운트 (allow-other: ec2-user 접근 허용 / region 명시 필수: 제한 환경에서 자동 탐색 실패함)
# 예시: s3://skills-web-bucket /data mount-s3 _netdev,nosuid,nodev,nofail,rw,allow-other,region=ap-northeast-2 0 0
s3://<실제_버킷명> /data mount-s3 _netdev,nosuid,nodev,nofail,rw,allow-other,region=<실제_리전> 0 0
```

```bash
sudo mount -a                          # fstab 전체 마운트
sudo chown ec2-user:ec2-user /efs      # EFS 소유권 변경
sudo chown ec2-user:ec2-user /data     # S3 마운트 소유권 변경
```

#### fstab 옵션 설명

| 옵션 | 설명 |
|------|------|
| `_netdev` | 네트워크 활성화 이후 마운트 시도 |
| `nofail` | 마운트 실패 시에도 부팅 계속 진행 |
| `tls` | 전송 계층 암호화 적용 |
| `iam` | IAM 역할을 통한 인증 |
| `allow-other` | root 외 다른 사용자도 파일 접근 가능 |
| `region=<리전>` | S3 버킷 리전 명시 (제한 환경에서 자동 탐색 실패 방지) |

---

## 2. Query from S3 (Athena)

> **요구사항 2**: S3에 저장된 데이터를 Athena로 쿼리

### 2-0. 🚨 1순위: Athena Query result location 설정

> **Athena는 결과 저장 경로가 지정되지 않으면 어떤 쿼리도 실행되지 않습니다.**  
> 테이블 생성 전에 반드시 먼저 설정하세요.

콘솔 경로: **Athena → Settings → Manage → Query result location**

```
s3://<실제_버킷명>/results/
```

끝에 `/` 포함 필수. 버킷이 없다면 미리 생성하거나 기존 버킷의 하위 경로 사용.

### 💡 [선택 가이드] S3 경로를 먼저 확인하세요

| Case | S3 폴더 예시 | `storage.location.template` |
|------|-------------|----------------------------|
| **A (Hive)** | `.../year=2025/month=04/day=05/` | `.../data/year=${year}/month=${month}/day=${day}/` |
| **B (Date)** | `.../2025/04/05/` | `.../data/${year}/${month}/${day}/` |

- **Case A**: `MSCK REPAIR TABLE <테이블명>;` 으로 파티션 등록 가능
- **Case B**: 반드시 아래 Partition Projection 방식 사용

### 2-1. Athena DDL (Partition Projection)

> **⚠️ 컬럼 정의 주의사항**  
> - 데이터 파일 내에 실제 `date` 컬럼이 없다면 컬럼 목록에서 **`date DATE` 삭제**하세요. 없는 컬럼을 정의하면 쿼리 시 해당 열이 비어있거나 에러가 납니다.  
> - `PARTITIONED BY`의 타입은 projection 설정과 반드시 일치해야 합니다. `integer` 프로젝션에는 **`INT`** 로 통일하세요.

```sql
CREATE EXTERNAL TABLE <MY_TABLE> (
    id         INT,
    user_id    INT,
    name       STRING,
    amount     DOUBLE,
    status     STRING,
    country    STRING,
    event_time TIMESTAMP
    -- date DATE  ← 데이터 파일에 실제 date 컬럼이 있을 때만 추가
)
PARTITIONED BY (year INT, month INT, day INT)   -- projection 타입(integer)과 일치시켜야 함
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://<MY_BUCKET>/data/'
TBLPROPERTIES (
    'projection.enabled'        = 'true',
    'projection.year.type'      = 'integer',
    'projection.year.range'     = '2025,2026',
    'projection.month.type'     = 'integer',
    'projection.month.range'    = '1,12',
    'projection.month.digits'   = '2',
    'projection.day.type'       = 'integer',
    'projection.day.range'      = '1,31',
    'projection.day.digits'     = '2',

    -- ↓ S3 경로 구조에 따라 하나만 선택
    -- Case A: 'storage.location.template' = 's3://<MY_BUCKET>/data/year=${year}/month=${month}/day=${day}/'
    -- Case B: 'storage.location.template' = 's3://<MY_BUCKET>/data/${year}/${month}/${day}/'
    'storage.location.template' = 's3://<MY_BUCKET>/data/${year}/${month}/${day}/'
);
```

> **Partition Projection이란?**  
> S3에 데이터가 추가될 때마다 `MSCK REPAIR`를 수동 실행할 필요 없이,  
> 설정된 범위 내의 파티션을 Athena가 자동으로 인식하는 방식.

---

## 3. Fine-grained IAM Policy

> **요구사항 3**: Condition 기반 세밀한 권한 제어 + Assume Role 세션 관리

### 3-1. Assume Role 실행 및 세션 적용

```bash
# 1. Assume Role
aws sts assume-role --role-arn $MY_ROLE_ARN --role-session-name mysession

# 2. 임시 자격증명 환경변수 적용 (Linux/Mac)
export AWS_ACCESS_KEY_ID="<결과값_AccessKeyId>"
export AWS_SECRET_ACCESS_KEY="<결과값_SecretAccessKey>"
export AWS_SESSION_TOKEN="<결과값_SessionToken>"

# Windows PowerShell의 경우
# $env:AWS_ACCESS_KEY_ID="<결과값_AccessKeyId>"
# $env:AWS_SECRET_ACCESS_KEY="<결과값_SecretAccessKey>"
# $env:AWS_SESSION_TOKEN="<결과값_SessionToken>"

# 3. 현재 권한 확인 (AssumedRole 이름이 출력되어야 성공)
aws sts get-caller-identity
```

### 3-2. S3 ABAC 정책 (객체 태그 기반 접근 제어)

> **⚠️ 콘솔 입력 시 주의**  
> - `Resource`의 `[실제_버킷_이름]` 부분은 **실제 버킷명으로 직접 교체**하세요.  
> - `${aws:username}`은 **IAM 전용 예약어** → 절대 수정하지 마세요.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::[실제_버킷_이름]"
        },
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::[실제_버킷_이름]/*",
            "Condition": {
                "StringEquals": {
                    "s3:ExistingObjectTag/Owner": "${aws:username}"
                }
            }
        }
    ]
}
```

---

## 4. MySQL with Lambda (생성만)

> **요구사항 4**: RDS MySQL 인스턴스 + Lambda 함수 생성 (연동 설정까지는 불필요)

### 4-1. RDS MySQL 생성 체크포인트

- Engine: **MySQL**
- Template: **Free tier** (또는 문제 지정)
- DB instance identifier: `<문제_지정명>`
- Credentials: username / password 메모 필수
- VPC: Lambda와 **동일한 VPC**에 배치
- Public access: **No** (Lambda에서 VPC 내부로 접근)
- SG inbound: `3306` → Lambda SG 또는 VPC CIDR 허용

```bash
# CLI로 생성하는 경우
aws rds create-db-instance \
    --db-instance-identifier <DB_인스턴스명> \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --master-username <유저명> \
    --master-user-password <비밀번호> \
    --allocated-storage 20 \
    --no-publicly-accessible \
    --region $MY_REGION
```

### 4-2. Lambda 실행 역할 필수 정책

> **🚨 VPC 내 Lambda는 이 정책이 없으면 생성 자체가 실패하거나 ENI를 만들지 못합니다.**

Lambda 실행 역할에 아래 AWS 관리형 정책을 반드시 연결하세요:

| 정책명 | 용도 |
|--------|------|
| `AWSLambdaBasicExecutionRole` | CloudWatch 로그 기본 권한 |
| `AWSLambdaVPCAccessExecutionRole` | VPC 내 ENI 생성 권한 (**VPC Lambda 필수**) |

```bash
# 역할에 VPC 정책 연결
aws iam attach-role-policy \
    --role-name <Lambda_실행역할명> \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
```

### 4-3. Lambda 함수 생성 (생성만)

```bash
aws lambda create-function \
    --function-name <Lambda_함수명> \
    --runtime python3.12 \
    --role arn:aws:iam::<계정ID>:role/<Lambda_실행역할명> \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip \
    --vpc-config SubnetIds=<서브넷ID>,SecurityGroupIds=<SG_ID> \
    --region $MY_REGION
```

> **💡 생성만 요구하는 경우 최소 구성**  
> - Runtime, Role, Handler, VPC Config 설정 후 더미 핸들러로 생성  
> - RDS 연결 코드는 작성하지 않아도 감점 없음

> **⚠️ VPC 내 Lambda는 인터넷 접근이 차단됩니다**  
> Lambda를 VPC에 배치하면 외부 인터넷 통신이 불가능합니다.  
> - S3와 통신이 필요하다면 → **S3 Gateway Endpoint** 생성 필요  
> - 외부 API 호출이 필요하다면 → **NAT Gateway** 필요  
> 이번 과제가 **생성만**이라면 VPC 배치 여부 자체가 채점 포인트이므로 VPC 설정에 집중하세요.

---

## 🚨 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `mount -a` 실행 시 멈춤 | SG에서 NFS 포트 미개방 | 인바운드 규칙 `TCP 2049` 허용 |
| S3 마운트 실패 (권한 제한 환경) | `region` 옵션 누락 | fstab에 `region=<실제_리전>` 추가 |
| S3 파일이 안 보임 | `allow-other` 옵션 누락 | `fstab`에 `allow-other` 추가 후 재마운트 |
| S3 마운트 후 쓰기 불가 | 소유권이 `root`로 설정됨 | `sudo chown ec2-user:ec2-user /data` 실행 |
| fstab 저장 후 마운트 실패 | fstab에 환경변수 사용 | 실제 값(EFS ID, 버킷명)으로 직접 교체 후 재시도 |
| Athena 쿼리가 아예 실행 안 됨 | Query result location 미설정 | Athena Settings → Query result location 먼저 지정 |
| Athena `0 records scanned` | 경로 불일치 | `LOCATION` 끝 `/` 확인, `storage.location.template` 구조 재확인 |
| Athena 컬럼 빈값 / 에러 | 없는 컬럼 정의 | 데이터 파일 실제 스키마 확인 후 불필요한 컬럼 제거 |
| Lambda 생성 실패 | 실행 역할에 VPC 정책 누락 | `AWSLambdaVPCAccessExecutionRole` 정책 연결 |
| Lambda → S3 통신 불가 | VPC 내 인터넷 차단 | S3 Gateway Endpoint 생성 |
| Lambda → RDS 연결 불가 | VPC / SG 불일치 | Lambda와 RDS가 동일 VPC인지, SG `3306` 허용 여부 확인 |
| 갑자기 권한 에러 (작업 중) | Assume Role 세션 만료 (기본 1시간) | `aws sts assume-role` 재실행 후 토큰 환경변수 재적용 |

### 원리 이해 포인트

- `tls` + `iam` → 전송 암호화 + IAM 인증으로 보안 요구사항 준수
- `_netdev` + `nofail` → 네트워크 스토리지 연결 안정성 확보
- Partition Projection → 수동 파티션 등록 없이 Athena 자동 인식
- ABAC Condition → 리소스 태그 기반 세밀한 접근 제어
