# EKS kubectl 접근 트러블슈팅 기록

## 초기 설정

### 목표: SSM EC2에서 kubectl로 EKS 클러스터 접근

**초기 구성**

- EKS 클러스터: private only (endpoint_public_access = false)
- SSM EC2: 프라이빗 서브넷, 공인 IP 없음
- SSM EC2 IAM Role: `AmazonSSMManagedInstanceCore` + `AmazonEKSClusterPolicy`

## 트러블슈팅 과정

1. kubeconfig 미설정 문제
   오류:

   ```bash
   dial tcp 127.0.0.1:8080: connect: connection refused
   ```

   원인: EC2 user_data에서 `aws eks update-kubeconfig`가 자동 실행됐어야 하는데 IAM 권한 부족으로 실패해서 kubeconfig가 없었음
   해결 시도: EC2 안에서 직접 `aws eks update-kubeconfig` 실행

2. IAM 권한 부족
   오류:

   ```bash
   AccessDeniedException: is not authorized to perform: eks:DescribeCluster
   ```

   원인: `AmazonEKSClusterPolicy`는 클러스터 관리용 정책이라 EC2에서 EKS API 호출하는 데 필요한 `eks:DescribeCluster` 권한이 없었음
   해결: `ssm.tf`에서 `AmazonEKSClusterPolicy` 제거하고 커스텀 인라인 정책 추가:

   ```hcl
   resource "aws_iam_role_policy" "ssm_eks_access" {
   policy = jsonencode({
   Statement = [{
   Effect = "Allow"
   Action = ["eks:DescribeCluster", "eks:ListClusters"]
   Resource = "\*"
   }]
   })
   }
   ```

3. EKS API 네트워크 타임아웃
   오류:

   ```bash
   dial tcp 10.0.12.248:443: i/o timeout
   ```

   원인: EKS cluster SG(control plane ENI SG)에서 SSM EC2 SG의 443 inbound가 허용 안 됐음
   해결: `modules/security_group/main.tf`에 추가

   ```hcl
   resource "aws_security_group_rule" "eks_from_ssm" {
   type = "ingress"
   from_port = 443
   to_port = 443
   protocol = "tcp"
   security_group_id = aws_security_group.eks_cluster.id
   source_security_group_id = aws_security_group.ssm_ec2.id
   }
   ```

4. EKS authentication mode 문제
   오류: RBAC 문제 해결을 위해 Access Entry 추가 후 `terraform apply` 시도 중 발생

   ```bash
   InvalidRequestException: The cluster's authentication mode must be set to one of [API, API_AND_CONFIG_MAP]
   ```

   원인: EKS Access Entry는 클러스터 authentication mode가 `API` 또는 `API_AND_CONFIG_MAP`이어야 하는데 기본값이 `CONFIG_MAP`이었음. Access Entry를 추가하려면 이 설정이 선행되어야 함

   해결 시도 1 — terraform apply로 변경:
   클러스터 재생성 시도했으나 노드 그룹이 붙어있어서 실패:

   ```bash
   ResourceInUseException: Cluster has nodegroups attached
   ```

   해결 시도 2 — AWS CLI로 직접 변경:

   ```bash
   aws eks update-cluster-config --name shoong-dev-cluster --access-config authenticationMode=API_AND_CONFIG_MAP
   ```

   변경 후 terraform apply 했으나 `bootstrap_cluster_creator_admin_permissions` 차이로 또 클러스터 재생성 시도

   최종 해결 — access_config에 명시적으로 추가:

   ```hcl
   access_config {
   authentication_mode = "API_AND_CONFIG_MAP"
   bootstrap_cluster_creator_admin_permissions = true
   }
   ```

   이렇게 하니 클러스터 재생성 없이 `update in-place`로 처리됨

5. EKS RBAC credentials 문제
   오류:

   ```bash
   the server has asked for the client to provide credentials
   ```

   원인: EKS는 IAM Role이 클러스터에 등록되어 있어야 kubectl 접근 가능한데, `shoong-dev-ssm-ec2-role`이 EKS Access Entry에 등록 안 됐었음

   해결 시도 1 — aws-auth ConfigMap 수정:
   EC2 안에서 직접 수정 시도했으나 파일 권한 문제로 실패

   ```bash
   sh: aws-auth.yaml: Permission denied
   ```

   해결 시도 2 — EKS Access Entry 추가:
   authentication mode를 `API_AND_CONFIG_MAP`으로 변경한 뒤 `modules/eks/main.tf`에 추가:

   ```hcl
    resource "aws_eks_access_entry" "ssm_ec2" {
    cluster_name = aws_eks_cluster.this.name
    principal_arn = var.ssm_ec2_role_arn
    type = "STANDARD"
    }

    resource "aws_eks_access_policy_association" "ssm_ec2" {
    cluster_name = aws_eks_cluster.this.name
    principal_arn = var.ssm_ec2_role_arn
    policy_arn = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
    access_scope {
    type = "cluster"
    }
    }
   ```

6. EKS VPC Endpoint 누락
   오류:

   ```bash
   dial tcp 10.0.11.113:443: i/o timeout
   ```

   원인: SSM EC2가 프라이빗 서브넷에 있어서 EKS API 엔드포인트 접근 시 VPC Endpoint가 필요한데 `com.amazonaws.us-east-1.eks` 엔드포인트가 없었음

   기존 VPC Endpoints:
   - ssm, ssmmessages, ec2messages (SSM용)
   - ecr.api, ecr.dkr (ECR용)
   - s3 (S3용)
   - eks 누락!

해결: `modules/vpc_endpoint/main.tf`에 추가:

```hcl
  resource "aws_vpc_endpoint" "eks" {
  vpc_id = var.vpc_id
  service_name = "com.amazonaws.${var.aws_region}.eks"
  vpc_endpoint_type = "Interface"
  subnet_ids = var.private_subnet_ids
  security_group_ids = [var.vpc_endpoint_sg_id]
  private_dns_enabled = true
  }
```

### 최종 결과

```bash
NAME STATUS ROLES AGE VERSION
ip-10-0-11-49.ec2.internal Ready <none> 150m v1.35.4-eks-40737a8
ip-10-0-12-108.ec2.internal Ready <none> 150m v1.35.4-eks-40737a8
```

## dev 환경 최종 구성:

- VPC (2 AZ)
- Security Group
- EKS 1.35 (private only)
- EKS Node Group (t3.small x2)
- RDS PostgreSQL 18.3
- ECR (5개 레포)
- ALB
- ACM
- Route53
- S3 (frontend)
- CloudFront + WAF
- VPC Endpoints (7개)
- SSM EC2

## 핵심 교훈:

- EKS private only 구성 시 VPC Endpoint 꼼꼼히 체크 필요 (eks 엔드포인트 추가 필요)
- EKS Access Entry 사용 시 authentication mode 명시적으로 설정 필요
- terraform으로 EKS 설정 변경 시 재생성 여부 plan에서 반드시 확인

1. EKS private only 구성 시 VPC Endpoint 확인!

- 클러스터 endpoint_public_access = false
- EC2가 프라이빗 서브넷
- NAT Gateway 없음 또는 제한적

이 경우 필수 체크:

- EKS VPC Endpoint (`com.amazonaws.<region>.eks`) 존재 여부
- Security Group에서 443 통신 허용 여부
- DNS (private DNS enabled)

확인 방법:

```bash
nslookup <EKS endpoint>
curl https://<EKS endpoint>
```
