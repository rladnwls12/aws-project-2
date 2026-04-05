# 🏆 2026 AWS 실전 인프라 구축

> **현장에서 발생하는 모든 실전 변수에 대응하는 최종 가이드**  
> S3 경로 차이, Assume Role 세션 설정, fstab 상세 옵션까지 포함

---

## 목차

- [0. 시작 전: 환경 변수 세팅](#0-시작-전-환경-변수-세팅)
- [1. IAM & Assume Role](#1-iam--assume-role)
- [2. Athena & S3 데이터 분석](#2-athena--s3-데이터-분석)
- [3. 영구 마운트 설정](#3-영구-마운트-설정-fstab--권한)
- [트러블슈팅 & 감점 방지](#-트러블슈팅--감점-방지)

---

## 0. 시작 전: 환경 변수 세팅

> 터미널 접속 직후 문제지에 적힌 리소스 이름을 아래 변수에 할당하세요.  
> 이후 모든 작업은 이 변수를 참조합니다. **오타 방지 1순위.**

```bash
# === [대회 지정 리소스 이름 입력] ===
export MY_BUCKET="skills-web-bucket"                              # S3 버킷명
export MY_TABLE="payments"                                        # Athena 테이블명
export MY_EFS_ID="fs-0c004845d6e9e7689"                          # EFS ID (콘솔에서 확인)
export MY_ROLE_ARN="arn:aws:iam::600440344359:role/S3_web_bucket" # Assume Role 대상 ARN
```

---

## 1. IAM & Assume Role

### 1-1. Assume Role 실행 및 세션 적용

사용자에게 직접 권한을 주지 않고 **역할(Role)** 을 통해 리소스에 접근하는 방식입니다.

```bash
# 1. 역할 수행 (Assume Role)
# 결과값에서 AccessKeyId, SecretAccessKey, SessionToken을 추출해야 함
aws sts assume-role --role-arn $MY_ROLE_ARN --role-session-name mysession

# 2. 임시 세션 토큰 적용 (Windows PowerShell)
$env:AWS_ACCESS_KEY_ID="[결과값_ID]"
$env:AWS_SECRET_ACCESS_KEY="[결과값_Key]"
$env:AWS_SESSION_TOKEN="[결과값_Token]"

# 3. 현재 권한 확인 (AssumedRole 이름이 떠야 성공)
aws sts get-caller-identity
```

### 1-2. S3 ABAC 정책 (Condition 활용)

객체의 태그와 사용자 이름을 비교하여 권한을 제어합니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::${MY_BUCKET}"
        },
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::${MY_BUCKET}/*",
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

## 2. Athena & S3 데이터 분석

데이터 경로 구조에 따라 **두 가지 방식** 중 하나를 선택해야 합니다.

### 💡 [선택 가이드] S3 경로를 먼저 확인하세요

| Case | S3 경로 예시 | 사용 방법 |
|------|-------------|-----------|
| **A (Hive)** | `.../data/year=2025/month=04/day=05/` | `MSCK REPAIR` 가능 |
| **B (Date)** | `.../data/2025/04/05/` | **반드시 Partition Projection 사용** |

### 2-1. 최적화된 Athena DDL (Partition Projection)

```sql
CREATE EXTERNAL TABLE ${MY_TABLE} (
    id         INT,
    user_id    INT,
    name       STRING,
    amount     DOUBLE,
    status     STRING,
    country    STRING,
    event_time TIMESTAMP,
    date       DATE
)
PARTITIONED BY (year INT, month STRING, day STRING)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://${MY_BUCKET}/data/'
TBLPROPERTIES (
    'projection.enabled'          = 'true',
    'projection.year.type'        = 'integer',
    'projection.year.range'       = '2025,2026',
    'projection.month.type'       = 'integer',
    'projection.month.range'      = '01,12',
    'projection.month.digits'     = '2',
    'projection.day.type'         = 'integer',
    'projection.day.range'        = '01,31',
    'projection.day.digits'       = '2',

    -- [주의] S3 경로 구조에 따라 아래 두 줄 중 하나만 사용
    -- Case A (Hive): 'storage.location.template' = 's3://${MY_BUCKET}/data/year=${year}/month=${month}/day=${day}/'
    -- Case B (Date): 'storage.location.template' = 's3://${MY_BUCKET}/data/${year}/${month}/${day}/'
    'storage.location.template'   = 's3://${MY_BUCKET}/data/${year}/${month}/${day}/'
);
```

---

## 3. 영구 마운트 설정 (fstab & 권한)

서버가 재부팅되어도 마운트가 유지되도록 `/etc/fstab`을 구성합니다.

### 3-1. /etc/fstab 설정

```bash
sudo vi /etc/fstab
```

```fstab
# S3FS 마운트 (allow-other 필수: ec2-user가 접근 가능하려면 반드시 추가)
s3://${MY_BUCKET} /data mount-s3 _netdev,nosuid,nodev,nofail,rw,allow-other 0 0

# EFS 마운트 (tls: 전송 암호화, iam: IAM 인증 사용)
${MY_EFS_ID}:/ /efs efs _netdev,noresvport,tls,iam,nofail 0 0
```

#### fstab 옵션 설명

| 옵션 | 설명 |
|------|------|
| `_netdev` | 네트워크가 활성화된 이후에 마운트 시도 |
| `nofail` | 마운트 실패 시에도 부팅 계속 진행 |
| `allow-other` | root 외 다른 사용자도 파일 접근 가능 |
| `tls` | 전송 계층 암호화 적용 |
| `iam` | IAM 역할을 통한 인증 사용 |

### 3-2. 권한 문제 해결

마운트 직후 소유자를 변경하지 않으면 쓰기 권한 오류가 발생할 수 있습니다.

```bash
sudo mount -a                          # fstab 내용 전체 마운트
sudo chown ec2-user:ec2-user /data     # S3 마운트 경로 권한 부여
sudo chown ec2-user:ec2-user /efs      # EFS 마운트 경로 권한 부여
```

---

## 🚨 트러블슈팅 & 감점 방지

### 자주 발생하는 문제 TOP 3

**1. EFS 마운트 실패**

90% 확률로 보안 그룹(SG)의 **2049번 포트(NFS)** 가 미개방된 경우입니다.  
→ EC2 인바운드 규칙에서 `TCP 2049` 허용 여부를 먼저 확인하세요.

**2. S3 파일이 안 보임**

`mount-s3` 실행 시 `allow-other` 옵션 누락 시 `root`만 파일을 볼 수 있습니다.  
→ `fstab`에 `allow-other` 옵션이 포함되어 있는지 반드시 확인하세요.

**3. Athena 조회 결과 0 records**

- `LOCATION` 경로 끝에 `/` 누락 여부 확인
- `storage.location.template`이 실제 S3 폴더 구조(`year=` 유무)와 일치하는지 확인

### 원리 이해가 경쟁력

단순히 코드를 복붙하는 것보다, 각 옵션이 왜 필요한지 설명할 수 있어야 합니다.

- `tls` + `iam` → 보안 요구사항(전송 암호화 + 인증) 준수
- `_netdev` + `nofail` → 네트워크 스토리지의 연결 안정성 확보
- Partition Projection → Hive 메타스토어 없이 동적 파티션 스캔 가능
