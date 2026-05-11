# ESO (External Secrets Operator) 설정 트러블슈팅

## 환경
- EKS (us-east-1)
- External Secrets Operator v1
- AWS Parameter Store + Secrets Manager

---

## 이슈 1: CRD 버전 불일치

### 증상
```
resource mapping not found for name: "shoong-db-secret" namespace: "dev"
no matches for kind "ExternalSecret" in version "external-secrets.io/v1beta1"
ensure CRDs are installed first
```

### 원인
yaml 파일에 `external-secrets.io/v1beta1` 로 작성했는데,
실제 클러스터에 설치된 ESO는 `external-secrets.io/v1` 버전이었음.

### 해결
모든 ESO yaml 파일의 apiVersion을 `v1beta1` → `v1` 로 변경
```yaml
# 변경 전
apiVersion: external-secrets.io/v1beta1

# 변경 후
apiVersion: external-secrets.io/v1
```

---

## 이슈 2: ssm:DescribeParameters 권한 누락

### 증상
```
AccessDeniedException: User: arn:aws:sts::733181095558:assumed-role/shoong-dev-eso-role/...
is not authorized to perform: ssm:DescribeParameters
```

### 원인
`external-secret-config.yaml` 에서 `dataFrom.find` 방식을 사용하면
Parameter Store의 파라미터 목록을 먼저 조회해야 함.
이때 `ssm:DescribeParameters` 권한이 필요한데 IAM 정책에 누락되어 있었음.

### 해결
테라폼 ESO IAM 정책에 `ssm:DescribeParameters` 추가
```terraform
{
  Effect = "Allow"
  Action = [
    "ssm:GetParameter",
    "ssm:GetParameters",
    "ssm:GetParametersByPath",
    "ssm:DescribeParameters"   # 추가
  ]
  Resource = "arn:aws:ssm:us-east-1:*:parameter/shoong/dev/*"
}
```

---

## 이슈 3: ssm:DescribeParameters Resource 범위 오류

### 증상
`ssm:DescribeParameters` 권한을 추가했음에도 동일한 AccessDeniedException 지속 발생

### 원인
`ssm:DescribeParameters` 는 특정 파라미터 경로가 아닌
**계정 전체 레벨**에서 동작하는 API임.
따라서 Resource를 `arn:aws:ssm:us-east-1:*:parameter/shoong/dev/*` 로 지정하면
권한이 적용되지 않음.

### 해결
`ssm:DescribeParameters` 만 별도 Statement로 분리하여 Resource를 `*` 로 변경
```terraform
# 기존 Statement (GetParameter 등)
{
  Effect = "Allow"
  Action = [
    "ssm:GetParameter",
    "ssm:GetParameters",
    "ssm:GetParametersByPath"
  ]
  Resource = "arn:aws:ssm:us-east-1:*:parameter/shoong/dev/*"
}

# 추가 Statement (DescribeParameters는 * 필요)
{
  Effect = "Allow"
  Action = [
    "ssm:DescribeParameters"
  ]
  Resource = "*"
}
```

---

## 최종 결과
```
NAME               STATUS         READY   
shoong-config      SecretSynced   True    # Parameter Store 10개 값 동기화
shoong-db-secret   SecretSynced   True    # Secrets Manager DATABASE_URL 동기화
```

---

## 핵심 교훈
1. ESO 설치 버전과 yaml apiVersion 반드시 일치시킬 것
2. `dataFrom.find` 사용 시 `ssm:DescribeParameters` 권한 필요
3. `ssm:DescribeParameters` 는 Resource를 `*` 로 설정해야 함 (파라미터 경로 지정 불가)
