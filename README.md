# 🐾 Petclinic Kubernetes 인프라 구성

Spring Petclinic 애플리케이션을 AWS EKS 위에 배포한 인프라 구성 문서입니다.

---

## 📐 전체 아키텍처
<img width="1051" height="1491" alt="EKS 개인프로젝트" src="https://github.com/user-attachments/assets/d667309e-4cc5-4758-9c03-837b15cecb00" />

```
외부 요청
    │
    ▼
NLB (Network Load Balancer)
    │
    ▼
ingress-nginx Controller
    │  (모든 요청 → web-svc)
    ▼
web-svc (nginx + php-fpm)
    │
    ├── /health     → 헬스체크
    ├── *.php       → php-fpm
    ├── /petclinic  → was-svc (Spring Petclinic 백엔드)
    └── /           → 정적 파일 서빙
    │
    ▼
was-svc (Tomcat)
    │
    ▼
AWS RDS (MySQL 8.0)
```

---

## 🏗️ 인프라 구성

### VPC

| 항목 | 값 |
|---|---|
| VPC 이름 | eks-petclinic-vpc |
| CIDR | 10.0.0.0/16 |

### 서브넷

| 이름 | 유형 | 가용 영역 | CIDR | ELB 태그 |
|---|---|---|---|---|
| public1 | Public | ap-northeast-2a | 10.0.0.0/20 | `kubernetes.io/role/elb = 1` |
| public1 | Public | ap-northeast-2c | 10.0.16.0/20 | `kubernetes.io/role/elb = 1` |
| private1 | Private | ap-northeast-2a | 10.0.128.0/20 | `kubernetes.io/role/internal-elb = 1` |
| private2 | Private | ap-northeast-2c | 10.0.144.0/20 | `kubernetes.io/role/internal-elb = 1` |

### NAT Gateway

| 항목 | 값 |
|---|---|
| 배치 위치 | Public Subnet |
| 용도 | Private 서브넷 → 인터넷 아웃바운드 허용 |

---

## ☸️ EKS 클러스터 구성

### IAM Role

#### AmazonEKSAutoClusterRole (클러스터 Role)

| 정책 이름 | 유형 | 용도 |
|---|---|---|
| AmazonEKSClusterPolicy | AWS 관리형 | EKS 클러스터 기본 운영 권한 |
| AmazonEKSBlockStoragePolicy | AWS 관리형 | EBS 볼륨 관리 (PVC 동적 프로비저닝) |
| AmazonEKSComputePolicy | AWS 관리형 | 노드 그룹 및 컴퓨팅 리소스 자동 프로비저닝 |
| AmazonEKSLoadBalancingPolicy | AWS 관리형 | ALB/NLB 생성 및 관리 |
| AmazonEKSNetworkingPolicy | AWS 관리형 | VPC CNI 기반 Pod 네트워크 구성 |
| EKSDeleteTags | 고객 인라인 | EKS가 생성한 리소스의 태그 삭제 권한 |

#### Node Role

| 정책 이름 | 유형 | 용도 |
|---|---|---|
| AmazonEKSWorkerNodeMinimalPolicy | AWS 관리형 | 노드가 EKS 클러스터에 등록되기 위한 최소 권한 |
| AmazonEC2ContainerRegistryPullOnly | AWS 관리형 | ECR Private 이미지 Pull 전용 권한 |
| AmazonElasticContainerRegistryPublicReadOnly | AWS 관리형 | ECR Public 이미지 읽기 권한 |

---

## 🔐 클러스터 접근 설정

Bastion 호스트에서 `kubectl`을 사용하기 위해 **EKS Cluster Access Management(Access Entry)** 방식을 사용했습니다.
기존 `aws-auth` ConfigMap 방식은 레거시로, Access Entry 방식이 현재 권장됩니다.

### aws-auth 방식 (레거시)

```yaml
# kubectl edit configmap aws-auth -n kube-system
mapRoles: |
  - rolearn: arn:aws:iam::102120298168:role/petclinic-bastion-role
    username: petclinic-bastion
    groups:
      - system:masters
```

### Access Entry 방식 (권장)

```bash
# Access Entry 생성
aws eks create-access-entry \
  --cluster-name <클러스터명> \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/petclinic-bastion-role \
  --region ap-northeast-2

# 정책 연결
aws eks associate-access-policy \
  --cluster-name <클러스터명> \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/petclinic-bastion-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ap-northeast-2
```

### kubeconfig 설정

```bash
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name <클러스터명>
```

---

## 🗄️ RDS 구성

| 항목 | 값 |
|---|---|
| DB 엔진 | MySQL 8.0 |
| 인스턴스 클래스 | db.m5.large |
| 퍼블릭 액세스 | 비활성화 |
| 연결 방식 | 동일 VPC 내 EKS에서만 접근 |

### DB 접속 정보 관리

K8s Secret을 사용하여 DB 접속 정보를 관리하고, Pod에 환경변수로 주입합니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: petclinic
type: Opaque
stringData:
  db.url: <RDS 엔드포인트>
  db.user: <사용자명>
  db.pw: <비밀번호>
```

---

## 🌐 Ingress 구성

ingress-nginx Controller를 Helm으로 설치하고 NLB와 연결했습니다.

### ingress-nginx 설치

```bash
# Helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ingress-nginx 설치
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-subnets"="subnet-036f6db07f107038c\,subnet-0e5e29286b8581a69"
```

### Ingress 리소스

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: petclinic
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

### 트래픽 흐름

| 경로 | 처리 |
|---|---|
| `/health` | nginx에서 200 반환 (헬스체크) |
| `*.php` | php-fpm으로 처리 |
| `/petclinic` | nginx → was-svc 프록시 |
| `/` | 정적 파일 서빙 |

---

## 🖥️ 서비스 구성

```bash
kubectl get svc -n petclinic

NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   
was-svc   ClusterIP   172.20.206.122   <none>        80/TCP    
web-svc   ClusterIP   172.20.65.134    <none>        80/TCP    
```

---

## 💡 개선 방향

- `aws-auth` ConfigMap → **EKS Access Entry** 방식으로 전환 (완료)
- K8s Secret → **AWS Secrets Manager** 연동으로 보안 강화
- ingress-nginx + NLB → **AWS Load Balancer Controller + ALB** 로 전환 시 AWS 네이티브 서비스 활용 가능
