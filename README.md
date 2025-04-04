<img src="https://capsule-render.vercel.app/api?type=waving&color=41ab5d&height=300&section=header&text=AWS-EKS-Monitoring-System&fontSize=50&fontColor=FFFFFF&animation=fadeIn&width=1200" width="1200" />


# 🌐 EKS 클러스터 모니터링 시스템 구축 가이드

AWS EKS 클러스터의 리소스 상태 및 노드 자원 사용량을 외부 EC2 인스턴스에서 Prometheus와 Grafana를 사용하여 실시간으로 모니터링할 수 있는 환경을 구축한 사례입니다.

---

# 🚩 프로젝트 개요



<br>

|<img src="https://github.com/DoomchitYJ.png" width="220" />|<img src="https://github.com/wns5120.png" width="220" />|<img src="https://github.com/EOTAEGYU.png" width="220" />|<img src="https://github.com/letsgojh0810.png" width="220" />|
|:-:|:-:|:-:|:-:|
|박영진<br/>[@DoomchitYJ](https://github.com/DoomchitYJ)|유호준<br/>[@wns5120](https://github.com/wns5120)|어태규<br/>[@EOTAEGYU](https://github.com/EOTAEGYU)|한정현<br/>[@letsgojh0810](https://github.com/letsgojh0810)|

<br>

## 📍 Contents
- [🔧 기술 스택](#-기술-스택)
- [📁 구성요소](#-구성요소)
- [⚙️ 구성 아키텍처](#-구성-아키텍처)
- [⬇️ 설치 및 설정 요약](#-설치-및-설정-요약)
- [✅ 확인 항목](#-확인-항목)
- [📌 참고 사항](#-참고-사항)
- [✨ 향후 확장 아이디어](#-향후-확장-아이디어)

<br>


## 🔧 기술 스택
<div>
  <img src="https://img.shields.io/badge/ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white">
  <img src="https://img.shields.io/badge/springboot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white">
  <img src="https://img.shields.io/badge/gradle-02303A?style=for-the-badge&logo=gradle&logoColor=white">
  
  <img src="https://img.shields.io/badge/docker-2496ED?style=for-the-badge&logo=docker&logoColor=white">
  <img src="https://img.shields.io/badge/kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white">
  <img src="https://img.shields.io/badge/yaml-CB171E?style=for-the-badge&logo=yaml&logoColor=white">
</div>


## 📦 구성 요소

### ✅ AWS EKS 클러스터
- 클러스터 이름: `ce-20-myeks`
- Helm 설치 후 [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) 배포
  - 구성 요소:
    - `kube-state-metrics`: Kubernetes 리소스 상태 메트릭 수집
    - `node-exporter`: 노드(CPU, 메모리 등) 리소스 메트릭 수집
- Prometheus 수집용 서비스들을 **LoadBalancer 타입**으로 노출

### ✅ EC2 인스턴스 (Ubuntu)
- 외부에서 Prometheus + Grafana 설치 및 운영
- Prometheus는 EKS에서 노출된 메트릭 엔드포인트를 수집
- Grafana는 Prometheus를 데이터 소스로 연결해 시각화

---

## ⚙️ 구성 아키텍처

```
[EKS 클러스터]
 ├─ kube-state-metrics (LoadBalancer)
 └─ node-exporter      (LoadBalancer)
       ▲
       │ (메트릭 수집)
       ▼
[EC2: Prometheus]
       ▼
[EC2: Grafana]
```

---

## ⬇️ 설치 및 설정 요약

### 1. EKS 클러스터 구성
```bash
eksctl create cluster --name ce-20-myeks --region ap-northeast-2 ...
```

### 2. Helm 설치 및 kube-prometheus-stack 배포
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### 3. 서비스 외부 노출
```bash
kubectl patch svc kube-prom-stack-kube-state-metrics -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc kube-prom-stack-prometheus-node-exporter -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n monitoring
```

### 4. EC2에 Prometheus 설치
- `prometheus.yml` 설정 예시:
```yaml
scrape_configs:
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['<EXTERNAL-IP>:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['<EXTERNAL-IP>:9100']
```

- Prometheus 서비스 시작:
```bash
sudo systemctl restart prometheus
```

### 5. EC2에 Grafana 설치 및 연결
- 접속: `http://<EC2_PUBLIC_IP>:3000`
- 기본 ID/PW: `admin / admin`
- Prometheus 데이터 소스 추가
- 대시보드 Import:
  - Dashboard ID: `315` (Kubernetes Cluster)
  - Dashboard ID: `1860` (Node Exporter Full)

---

## ✅ 확인 항목

| 항목                    | 주소                          |
|-------------------------|-------------------------------|
| Prometheus Web UI       | `http://<EC2>:9090`           |
| Prometheus Targets      | `http://<EC2>:9090/targets`   |
| Grafana Web UI          | `http://<EC2>:3000`           |
| Grafana Dashboard       | 클러스터/노드 상태 시각화    |

---

## 📌 참고 사항
- EC2 보안 그룹에 포트 `9090`, `3000` 인바운드 허용 필요
- `kubectl`, `aws`, `helm` CLI 도구 설치 필요
- Prometheus는 외부에서 직접 수집, EKS 내부 Prometheus는 사용하지 않음

---

## ✨ 향후 확장 아이디어
- Alertmanager를 통해 Slack/Email 알림 연동
- Grafana 알림 조건 설정
- Prometheus Remote Write를 통해 외부 저장소 연동

