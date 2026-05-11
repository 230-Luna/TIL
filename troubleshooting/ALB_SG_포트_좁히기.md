# ALB SG -> EKS Cluster SG 포트 범위 좁히기

## 발단

`modules/eks/main.tf`에 ALB SG에서 EKS Cluster SG로 0-65535 모든 포트를 허용하는 룰이 있었다.

```hcl
resource "aws_security_group_rule" "alb_to_eks_cluster" {
  type                     = "ingress"
  from_port                = 0
  to_port                  = 65535
  protocol                 = "tcp"
  security_group_id        = aws_eks_cluster.this.vpc_config[0].cluster_security_group_id
  source_security_group_id = var.alb_sg_id
  description              = "ALB to EKS cluster for TGB ip mode"
}
```

처음엔 TGB(TargetGroupBinding) ip 모드의 health check 동작 확인용으로 열어뒀다가 그대로 남아있던 상태였다.

---

## 원인 파악

### ALB가 실제로 쓰는 포트

`modules/alb/main.tf`의 Target Group 정의를 보면 ALB가 Pod로 보내는 트래픽은 딱 두 종류뿐이다.

```hcl
resource "aws_lb_target_group" "this" {
  port = 80
  health_check {
    path = "/healthz/ready"
    port = "15021"
  }
}
```

| 용도         | 포트  | 설명                                                     |
| ------------ | ----- | -------------------------------------------------------- |
| HTTP 트래픽  | 80    | 사용자 요청을 Istio Ingress Gateway로 포워딩             |
| Health Check | 15021 | Istio Ingress Gateway readiness probe (`/healthz/ready`) |

나머지 0-65535는 전부 불필요하게 열려있던 셈이었다.

### 왜 EKS Cluster SG인가

EKS의 `cluster_security_group_id`는 워커 노드 ENI에 자동 부착되고, **VPC CNI 사용 시 Pod ENI에도 자동 부착된다.** `target_type=ip` 모드에서 ALB가 Pod IP로 직접 트래픽을 보내기 때문에 이 SG를 통과하게 된다.

즉 0-65535 허용은 ALB SG를 가진 누구든 Pod의 모든 포트에 도달할 수 있다는 뜻이었다.

노출 위험 예시:

- Node.js 디버그 포트 (9229)
- Istio 사이드카 admin 포트 (15000)
- Prometheus 메트릭 포트 (15090)
- 애플리케이션 내부 디버그 엔드포인트

---

## 해결

기존 룰을 삭제하고 실제 사용 포트 두 개로 분리했다.

```hcl
# ALB -> Istio Ingress Gateway (HTTP 트래픽)
resource "aws_security_group_rule" "alb_to_istio_http" {
  type                     = "ingress"
  from_port                = 80
  to_port                  = 80
  protocol                 = "tcp"
  security_group_id        = aws_eks_cluster.this.vpc_config[0].cluster_security_group_id
  source_security_group_id = var.alb_sg_id
  description              = "ALB to Istio Ingress Gateway HTTP"
}

# ALB -> Istio Ingress Gateway (헬스체크)
resource "aws_security_group_rule" "alb_to_istio_healthcheck" {
  type                     = "ingress"
  from_port                = 15021
  to_port                  = 15021
  protocol                 = "tcp"
  security_group_id        = aws_eks_cluster.this.vpc_config[0].cluster_security_group_id
  source_security_group_id = var.alb_sg_id
  description              = "ALB health check to Istio Ingress Gateway readiness"
}
```

ALB는 80, 15021 두 포트만 닿을 수 있고 그 외는 차단된다.

---

## 향후 고려사항

현재는 ALB에서 TLS 종료(`certificate_arn` + 443 listener) 후 ALB -> Pod 구간은 HTTP라서 80만으로 충분하다. mTLS나 Pod에서 직접 TLS 종료하는 구조로 바꾼다면 443도 추가해야 한다.

```hcl
resource "aws_security_group_rule" "alb_to_istio_https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  security_group_id        = aws_eks_cluster.this.vpc_config[0].cluster_security_group_id
  source_security_group_id = var.alb_sg_id
  description              = "ALB to Istio Ingress Gateway HTTPS"
}
```

---

> TGB ip 모드에서 ALB -> Cluster SG 룰은 Target Group의 listener port + health check port만 열면 충분하다.
> EKS Cluster SG는 Pod ENI에도 부착되므로 광범위 허용은 Pod 포트 전체 노출과 같다.
