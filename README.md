# 🏆 AWS 실전 인프라 구축: 마스터 시트

---

# [!!유용한 블로그!!](https://velog.io/@jinseo1234/series)
## https://velog.io/@zenru/S3-Query
## https://velog.io/@zenru/AWS-Assume-Role
## https://www.yalco.kr/@sql/1-5/
## [iam 참고](https://velog.io/@jinseo1234/fine-grained-iam-policy-%EA%B9%80%EC%A7%80%ED%9B%882-1775441378)

# [Athena 테이블 구성 참고](https://velog.io/@jinseo1234/query-from-s3-%EA%B9%80%EC%A7%80%ED%9B%882-1775441380)

# [쿼리문 참고](https://velog.io/@jinseo1234/athena-%EC%BF%BC%EB%A6%AC%EB%AC%B8-1775441356)

## 📋 요구사항

| # | 요구사항 | 핵심 |
|---|---------|------|
| 1 | Shared network storage | EFS + S3FS 영구 마운트 |
| 2 | Query from S3 | Athena Partition Projection |
| 3 | Fine-grained IAM policy | ABAC Condition + Assume Role |
| 4 | MySQL with Lambda (**생성만**) | RDS MySQL + Lambda (VPC 내) |
| 5 | bastion 권한에 | AdministratorAccess  |

**유의사항**: SG outbound `80/443` Any open 유지 / Tag 누락 시 채점 불가 / 채점은 Cloud Shell 기준

---

## 0. 시작 전

### 생성 순서

```
VPC/SG 확인 → S3 버킷 → EC2 확인 → EFS + Mount Target → IAM Role
→ Athena Settings → RDS MySQL → Lambda 실행역할 → Lambda 함수
→ 환경변수 export → 마운트/Athena/IAM 테스트
```

### 사전 점검 (1번 진입 전 전부 체크)

- [ ] S3 버킷 존재 / `/data/` `/results/` 경로 파악
- [ ] EC2 준비 + EFS와 동일 VPC
- [ ] EFS + **Mount Target** 생성 완료 (생성만으로 마운트 안 됨)
- [ ] EFS SG inbound `TCP 2049(NFS)` 열림
- [ ] Assume Role 대상 IAM Role 존재
- [ ] **Athena → Settings → Query result location** 설정 완료
- [ ] RDS MySQL이 Lambda와 동일 VPC
- [ ] Lambda 실행역할에 `AWSLambdaVPCAccessExecutionRole` 연결
- [ ] Lambda용 Subnet ID / SG ID 확보
- [ ] 모든 리소스 Tag 누락 없음

## 1. Shared Network Storage

> 🚨 `mount -a` 실행 후 무반응 = 99% SG의 `TCP 2049` 미개방

### 설치

```bash
sudo dnf install -y amazon-efs-utils   # Amazon Linux
# sudo apt install -y amazon-efs-utils # Ubuntu
sudo mkdir -p /efs /data
```

### /etc/fstab 설정

```bash
# nano 추천. vi라면: i → 입력 → Esc → :wq → Enter
sudo vim /etc/fstab
```

파일 하단에 추가 (꺾쇠 포함 실제 값으로 교체):

```
# EFS — tls: 암호화, iam: IAM 인증
# 예) fs-0c004845d6e9e7689:/ /efs efs _netdev,noresvport,tls,iam,nofail 0 0

<실제_EFS_ID>:/ /efs efs _netdev,noresvport,tls,iam,nofail 0 0

# S3 — allow-other: ec2-user 접근 허용, region: 제한 환경에서 자동탐색 실패 방지
# 예) s3://my-bucket /data mount-s3 _netdev,nosuid,nodev,nofail,rw,allow-other,region=ap-northeast-2 0 0

s3://<실제_버킷명> /data mount-s3 _netdev,nosuid,nodev,nofail,rw,allow-other,region=<실제_리전> 0 0
```

```bash
sudo mount -a
sudo chown ec2-user:ec2-user /efs /data
```

| 옵션 | 설명 |
|------|------|
| `_netdev` | 네트워크 활성화 후 마운트 |
| `nofail` | 마운트 실패해도 부팅 계속 |
| `tls` | 전송 암호화 |
| `iam` | IAM 인증 |
| `allow-other` | root 외 사용자 접근 허용 |
| `region=` | S3 리전 명시 (누락 시 마운트 실패 가능) |

---

## 2. Query from S3 (Athena)

> 🚨 **먼저**: Athena → Settings → Manage → Query result location → `s3://<버킷명>/results/` (끝 `/` 필수)

> 🚨 `0 records` = `storage.location.template`이 실제 S3 경로 구조와 불일치

### S3 경로 확인 후 선택

| Case | S3 폴더 구조 | `storage.location.template` |
|------|-------------|----------------------------|
| **A (Hive)** | `.../year=2025/month=04/day=05/` | `s3://<버킷>/data/year=${year}/month=${month}/day=${day}/` |
| **B (Date)** | `.../2025/04/05/` | `s3://<버킷>/data/${year}/${month}/${day}/` |

Case A는 `MSCK REPAIR TLE <테이블명>;` 가능. Case B는 반드시 아래 Projection 사용.

### Athena DDL

> ⚠️ 실제 데이터 파일에 없는 컬럼은 정의하지 마세요 (빈값/에러 발생)  
> ⚠️ `PARTITIONED BY` 타입은 projection 타입과 반드시 일치 → 전부 `INT`

```sql
CREATE EXTERNAL TABLE payments (
    id         INT,
    user_id    INT,
    name       STRING,
    amount     DOUBLE,
    status     STRING,
    country    STRING,
    event_time TIMESTAMP,
    date       DATE 
)
PARTITIONED BY (year INT, month INT, day INT)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://[버켓 이름]/data/'
TBLPROPERTIES (
    'projection.enabled'        = 'true', 
    'projection.year.type'      = 'integer',
    'projection.year.range'     = '2025,2026',
    'projection.month.type'     = 'integer',
    'projection.month.range'    = '01,12',     
    'projection.month.digits'   = '2',
    'projection.day.type'       = 'integer',
    'projection.day.range'      = '01,31',   
    'projection.day.digits'     = '2',
    'storage.location.template' = 's3://[버켓 이름]/data/year=${year}/month=${month}/day=${day}/' 
);

```

---

## 3. Fine-grained IAM Policy

> 🚨 `${aws:username}`은 IAM 예약어 — 절대 수정 금지. AWS가 자동으로 현재 접속자를 대입함

### Assume Role





# 1. Role 수행

## 🛠️ IAM Role 생성 절차 

1. **IAM 콘솔 접속**
   * AWS Management Console에서 **IAM** 서비스로 이동합니다.

2. **역할 생성 시작**
   * 왼쪽 메뉴에서 **Roles (역할)**를 선택한 후, 우측 상단의 **Create role (역할 생성)** 버튼을 클릭합니다.

3. **신뢰할 수 있는 엔티티 선택 (Select trusted entity)**
   * **Trusted entity type**: `AWS account`를 선택합니다.
   * **An AWS account**: `This account` (현재 계정)를 선택합니다.
   * **⚠️ 중요**: 하단의 **MFA (Multi-Factor Authentication)** 체크박스가 **해제**되어 있는지 확인합니다.

4. **권한 추가 (Add permissions)**
   * (별도의 정책 연결 없이) 하단의 **Next**를 눌러 넘어갑니다. (나중에 인라인 정책으로 추가)

5. **이름 설정 및 생성 (Name, review, and create)**
   * **Role name**: `MyS3AccessRole`을 입력합니다.
   * 하단의 **Create role** 버튼을 눌러 생성을 완료합니다.
   * 
```bash


aws sts assume-role --role-arn $MY_ROLE_ARN --role-session-name mysession




# 2. 출력된 값을 아래에 붙여넣기 (Linux/Mac)
export AWS_ACCESS_KEY_ID="<결과_AccessKeyId>"
export AWS_SECRET_ACCESS_KEY="<결과_SecretAccessKey>"
export AWS_SESSION_TOKEN="<결과_SessionToken>"



# Windows PowerShell
# $env:AWS_ACCESS_KEY_ID="<결과_AccessKeyId>"
# $env:AWS_SECRET_ACCESS_KEY="<결과_SecretAccessKey>"
# $env:AWS_SESSION_TOKEN="<결과_SessionToken>"

# 3. 확인 (출력에 AssumedRole 이름이 나와야 성공)
aws sts get-caller-identity
```

> ⏱️ 세션은 기본 **1시간 후 만료** → 갑자기 권한 에러 나면 위 과정 재실행

### S3 AC 정책

> ⚠️ 콘솔에서 **직접 입력(제발 타이핑)** 시 `[실제_버킷_이름]` → 실제 버킷명으로 교체. `${aws:username}`은 그대로.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowBucketActions",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::[실제_버킷_이름]-ap-northeast-2-an"
        },
        {
            "Sid": "AllowObjectActionsWithTag",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::[실제_버킷_이름]-ap-northeast-2-an/*",
            "Condition": {
                "StringEquals": {
                    "s3:ExistingObjectTag/Owner": "${aws:username}"
                }
            }
        }
    ]
}


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3Mount",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::skills-web-bucket",
                "arn:aws:s3:::skills-web-bucket/*"
            ]
        }
    ]
}


```

---

## 4. MySQL with Lambda (생성만)

> 🚨 Lambda를 VPC에 넣으면 `AWSLambdaVPCAccessExecutionRole` 없이는 ENI 생성 실패 → 무한 로딩  
> 🚨 VPC 내 Lambda는 인터넷 차단 → S3 접근 필요 시 S3 Gateway Endpoint 별도 생성

### RDS MySQL

```bash
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

콘솔 체크: Engine `MySQL` / VPC = Lambda와 동일 / Public access `No` / SG inbound `TCP 3306` from Lambda SG

### Lambda 실행역할 — VPC 정책 연결

```bash
aws iam attach-role-policy \
    --role-name <Lambda_실행역할명> \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
```

### Lambda 함수 생성

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

---

## ✅ 종료 전 최종 검증 (명령어로 직접 확인)

```bash
# 1. 마운트 확인
df -h | grep -E "/efs|/data"
# → /efs 와 /data 가 모두 보여야 합니다

# 2. EFS 쓰기 테스트
echo "efs-test" | sudo tee /efs/test.txt && cat /efs/test.txt

# 3. S3 마운트 파일 확인
ls /data

# 4. Assume Role 확인
aws sts get-caller-identity
# → "Arn" 에 "assumed-role" 이 포함되어야 합니다

# 5. Lambda VPC 설정 확인
aws lambda get-function-configuration \
    --function-name <Lambda_함수명> \
    --query 'VpcConfig' --region $MY_REGION
# → SubnetIds, SecurityGroupIds 가 비어 있지 않아야 합니다

# 6. RDS 상태 확인
aws rds describe-db-instances \
    --db-instance-identifier <DB_인스턴스명> \
    --query 'DBInstances[0].DBInstanceStatus' --region $MY_REGION
# → "available" 이어야 합니다
```

| 항목 | 검증 명령어 | 기대 결과 |
|------|------------|---------|
| EFS/S3 마운트 | `df -h` | `/efs`, `/data` 노출 |
| Athena 조회 | `SELECT` 실행 | 실제 데이터 출력 |
| IAM Assume | `get-caller-identity` | `assumed-role` 포함 |
| Lambda VPC | `get-function-configuration` | Subnet/SG 값 존재 |
| RDS 상태 | `describe-db-instances` | `available` |
