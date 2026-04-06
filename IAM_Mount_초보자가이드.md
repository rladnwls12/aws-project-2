# IAM + EFS/S3 마운트 초보자 완전 가이드

> 배경지식 없이 이 문서만 보고 따라할 수 있도록 작성했습니다.  
> **꺾쇠(`<>`) 안의 값은 반드시 실제 값으로 교체하세요.**

---

https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/mountpoint-installation.html

## 목차

1. [핵심 개념 3줄 요약](#1-핵심-개념-3줄-요약)
2. [만들어야 할 역할 목록](#2-만들어야-할-역할-목록)
3. [EC2 인스턴스용 역할 만들기](#3-ec2-인스턴스용-역할-만들기)
4. [Lambda 실행용 역할 만들기](#4-lambda-실행용-역할-만들기)
5. [Assume Role 대상 역할 만들기 (MyS3AccessRole)](#5-assume-role-대상-역할-만들기-mys3accessrole)
6. [EC2 인스턴스에 역할 붙이기](#6-ec2-인스턴스에-역할-붙이기)
7. [EC2에서 EFS + S3 마운트하기](#7-ec2에서-efs--s3-마운트하기)
8. [역할이 안 보일 때 해결법](#8-역할이-안-보일-때-해결법)
9. [최종 확인 체크리스트](#9-최종-확인-체크리스트)

---

## 1. 핵심 개념 3줄 요약

| 용어 | 한 줄 설명 | 비유 |
|------|-----------|------|
| **역할 (Role)** | EC2, Lambda 같은 서비스에 붙이는 권한 묶음 | 사원증 |
| **정책 (Policy)** | 실제 권한 내용이 담긴 JSON 문서 | 사원증에 적힌 출입 구역 목록 |
| **Assume Role** | 다른 역할을 빌려 그 권한으로 잠깐 작업하는 것 | 임시 출입증 발급 |

> **왜 역할이 필요한가?**  
> EC2 서버가 S3나 EFS에 접근하려면 "이 서버는 허가된 서버입니다"라는 증명이 필요합니다.  
> 그게 바로 **IAM 역할**입니다. 역할 없이는 `mount` 명령이 권한 오류로 실패합니다.

---

## 2. 만들어야 할 역할 목록

| # | 역할 이름 (예시) | 용도 | Trusted Entity |
|---|----------------|------|---------------|
| A | `EC2-EFS-S3-Role` | EC2가 EFS/S3 마운트할 때 사용 | AWS service → EC2 |
| B | `LambdaExecutionRole` | Lambda 함수 실행 시 사용 | AWS service → Lambda |
| C | `MyS3AccessRole` | Assume Role 테스트용 (ABAC 정책 연결) | AWS account → This account |

---

## 3. EC2 인스턴스용 역할 만들기

### 3-1. 역할 생성

1. AWS 콘솔 상단 검색창에 **`IAM`** 입력 → 클릭
2. 왼쪽 메뉴 → **Roles** → 오른쪽 상단 **Create role** 클릭
3. **Trusted entity type** : `AWS service` 선택
4. **Use case** : `EC2` 선택 → **Next**
5. 정책 추가 화면에서 아래 2개를 검색해서 체크:
   - `AmazonEFSFullAccess`
   - `AmazonS3FullAccess`
6. **Next** → **Role name** 에 `EC2-EFS-S3-Role` 입력 → **Create role**

### 3-2. 인라인 정책 추가 (S3 마운트 전용)

> 위에서 AmazonS3FullAccess를 붙였다면 이 단계는 생략 가능합니다.  
> 최소 권한으로 제한하고 싶을 때만 사용하세요.

1. 방금 만든 역할 클릭 → **Add permissions** → **Create inline policy**
2. **JSON** 탭 클릭 → 기존 내용 전부 지우고 아래 붙여넣기

```json
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
                "arn:aws:s3:::<실제_버킷명>",
                "arn:aws:s3:::<실제_버킷명>/*"
            ]
        }
    ]
}
```

3. **Next** → Policy name 입력 → **Create policy**

---

## 4. Lambda 실행용 역할 만들기

1. IAM → **Roles** → **Create role**
2. **Trusted entity type** : `AWS service`
3. **Use case** : `Lambda` 선택 → **Next**
4. 정책 추가 화면에서 검색 후 체크:
   - `AWSLambdaBasicExecutionRole` (기본 CloudWatch 로그 권한)
   - `AWSLambdaVPCAccessExecutionRole` ← **이게 없으면 Lambda가 VPC 안에서 ENI를 못 만들어서 무한 로딩됩니다**
5. **Next** → Role name: `LambdaExecutionRole` → **Create role**

> **왜 VPCAccessExecutionRole이 필요한가?**  
> Lambda를 VPC 안에 배치하면, Lambda가 VPC 내부에 네트워크 카드(ENI)를 직접 만들어야 합니다.  
> 이 정책이 없으면 ENI 생성 권한이 없어서 함수가 시작조차 못합니다.

나중에 CLI로 역할에 정책을 붙이는 방법:

```bash
aws iam attach-role-policy \
    --role-name LambdaExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
```

---

## 5. Assume Role 대상 역할 만들기 (MyS3AccessRole)

이 역할은 **사람(IAM 사용자)이 임시로 빌려서 쓰는** 역할입니다.

### 5-1. 역할 생성

1. IAM → **Roles** → **Create role**
2. **Trusted entity type** : `AWS account` 선택
3. **An AWS account** : `This account` 선택
4. **⚠️ 중요**: 아래 **MFA** 체크박스가 **해제** 상태인지 확인
5. 정책 추가 없이 **Next** → Role name: `MyS3AccessRole` → **Create role**

### 5-2. 인라인 정책 추가 (ABAC S3 접근 정책)

1. `MyS3AccessRole` 클릭 → **Add permissions** → **Create inline policy**
2. **JSON** 탭 → 아래 붙여넣기

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowBucketActions",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::<실제_버킷명>"
        },
        {
            "Sid": "AllowObjectActionsWithTag",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::<실제_버킷명>/*",
            "Condition": {
                "StringEquals": {
                    "s3:ExistingObjectTag/Owner": "${aws:username}"
                }
            }
        }
    ]
}
```

> **`${aws:username}` 절대 수정 금지!**  
> 이건 오타가 아니라 IAM 예약어입니다. AWS가 접속한 사용자 이름을 자동으로 대입합니다.

3. **Next** → Policy name 입력 → **Create policy**

---

## 6. EC2 인스턴스에 역할 붙이기

> EC2 인스턴스에 역할을 붙여야 그 서버가 S3, EFS에 접근할 수 있습니다.

### 6-1. 인스턴스 생성 시 붙이는 방법

1. EC2 → **Launch instance**
2. 아래로 스크롤 → **Advanced details** 펼치기
3. **IAM instance profile** 드롭다운에서 `EC2-EFS-S3-Role` 선택

### 6-2. 기존 인스턴스에 붙이는 방법

1. EC2 콘솔 → **Instances** → 대상 인스턴스 체크
2. 상단 **Actions** → **Security** → **Modify IAM role**
3. **IAM role** 드롭다운에서 `EC2-EFS-S3-Role` 선택 → **Update IAM role**

---

## 7. EC2에서 EFS + S3 마운트하기

> SSH로 EC2 인스턴스에 접속한 상태에서 진행합니다.

### 7-1. 필요한 패키지 설치

```bash
# Amazon Linux 2 / Amazon Linux 2023
sudo dnf install -y amazon-efs-utils

# Ubuntu라면
# sudo apt install -y amazon-efs-utils

# S3 마운트 도구 (mount-s3)는 별도 설치 필요
# Amazon Linux에는 기본 내장된 경우 많음. 없으면:
# sudo dnf install -y mountpoint-s3
```

### 7-2. 마운트 포인트 디렉터리 생성

```bash
sudo mkdir -p /efs /data
```

### 7-3. /etc/fstab 편집

> `fstab` = 부팅할 때 자동으로 마운트할 목록을 적는 파일

```bash
sudo nano /etc/fstab
```

파일 맨 아래에 아래 두 줄 추가 (꺾쇠 포함 실제 값으로 교체):

```
# EFS 마운트
<실제_EFS_ID>:/ /efs efs _netdev,noresvport,tls,iam,nofail 0 0

# S3 마운트
s3://<실제_버킷명> /data mount-s3 _netdev,nosuid,nodev,nofail,rw,allow-other,region=<실제_리전> 0 0
```

**EFS ID 형식 예시**: `fs-0c004845d6e9e7689`

저장 방법 (nano 기준):
- `Ctrl + O` → Enter (저장)
- `Ctrl + X` (종료)

### 7-4. 마운트 실행

```bash
sudo mount -a
```

> `mount -a` = fstab에 적힌 항목 전부 지금 마운트

### 7-5. 소유권 변경 (ec2-user가 쓸 수 있도록)

```bash
sudo chown ec2-user:ec2-user /efs /data
```

### 7-6. 마운트 확인

```bash
df -h | grep -E "/efs|/data"
```

`/efs` 와 `/data` 가 둘 다 보이면 성공입니다.

---

## 8. 역할이 안 보일 때 해결법

### 상황 A: 인스턴스 프로파일 드롭다운에 역할이 안 보임

**원인**: 역할에 **Instance Profile**이 아직 연결 안 된 경우 (드물지만 발생)

**해결**:
1. IAM → Roles → 역할 이름 클릭 → 상단에 **Instance Profile ARN** 이 있는지 확인
2. 없다면 CLI로 직접 생성:

```bash
# 인스턴스 프로파일 생성
aws iam create-instance-profile --instance-profile-name EC2-EFS-S3-Role

# 역할을 프로파일에 연결
aws iam add-role-to-instance-profile \
    --instance-profile-name EC2-EFS-S3-Role \
    --role-name EC2-EFS-S3-Role
```

### 상황 B: Modify IAM role 화면에서 역할이 드롭다운에 안 뜸

**원인 1**: 역할 이름이 길거나 검색이 느린 경우  
**해결**: 드롭다운 검색창에 역할 이름 앞 3~4글자만 입력해서 필터링

**원인 2**: 역할의 Trusted Entity가 `EC2`가 아닌 경우  
**해결**: IAM → 해당 역할 → **Trust relationships** 탭 확인  
아래 내용이 있어야 합니다:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

없다면 **Edit trust policy** 클릭 → 위 내용으로 교체 → **Update policy**

**원인 3**: 방금 역할을 만들었는데 아직 콘솔에 반영 안 된 경우  
**해결**: 브라우저 새로고침 후 다시 시도

### 상황 C: `mount -a` 실행 후 멈추거나 오류 남

| 증상 | 원인 | 해결 |
|------|------|------|
| 무반응 (hang) | EFS 보안그룹에 TCP 2049 미개방 | EFS SG → Inbound rules → TCP 2049 추가 |
| `Permission denied` | EC2 역할에 EFS 권한 없음 | 역할에 `AmazonEFSFullAccess` 정책 추가 |
| S3 마운트 실패 | `region=` 옵션 누락 | fstab에 `region=ap-northeast-2` 명시 |

---

## 9. 최종 확인 체크리스트

```bash
# 1. EFS + S3 마운트 확인
df -h | grep -E "/efs|/data"
# → /efs 와 /data 둘 다 출력되어야 함

# 2. EFS 쓰기 테스트
echo "test" | sudo tee /efs/test.txt && cat /efs/test.txt
# → "test" 출력되면 성공

# 3. S3 파일 목록 확인
ls /data
# → S3 버킷 내 파일 목록 출력되어야 함

# 4. Assume Role 확인
aws sts get-caller-identity
# → "Arn" 에 "assumed-role" 포함되어야 함

# 5. Lambda VPC 설정 확인
aws lambda get-function-configuration \
    --function-name <Lambda_함수명> \
    --query 'VpcConfig' --region <리전>
# → SubnetIds, SecurityGroupIds 값이 비어 있지 않아야 함

# 6. RDS 상태 확인
aws rds describe-db-instances \
    --db-instance-identifier <DB_인스턴스명> \
    --query 'DBInstances[0].DBInstanceStatus' --region <리전>
# → "available" 이어야 함
```

---

## 빠른 트러블슈팅 요약

| 문제 | 가장 먼저 확인할 것 |
|------|-------------------|
| `mount -a` 가 멈춤 | EFS 보안그룹 → Inbound TCP **2049** 열려있는지 |
| Lambda 무한 로딩 | 역할에 `AWSLambdaVPCAccessExecutionRole` 붙어있는지 |
| Athena 0 records | `storage.location.template` 경로가 실제 S3 경로와 일치하는지 |
| Assume Role 실패 | `MY_ROLE_ARN` 환경변수가 올바른지, 역할 Trust Policy에 현재 계정이 있는지 |
| 역할이 드롭다운에 안 보임 | Trust relationships에 `ec2.amazonaws.com` 있는지 / 새로고침 |
