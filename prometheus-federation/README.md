# Prometheus Federation
(완성된 문서가 아닙니다.)

## 공식 문서
Prometheus Federation에 대한 문서는 [여기](https://prometheus.io/docs/prometheus/latest/federation/)를 확인하세요.<br>

## 참고 문서
- [SeSAC-AWS-Final-Team-2/dev-eks](https://github.com/SeSAC-AWS-Final-Team-2/dev-eks/blob/main/third-party/monitoring.md)
- https://github.com/prometheus-operator/prometheus-operator/issues/2877

<br>

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [helm-chart/kube-prometheus-stack](#helm-chartkube-prometheus-stack)
    3. [Amazon EKS](#amazon-eks)
3. [필요 파일](#필요-파일)
4. [실행 방법](#실행-방법)
5. [수집 metric 확인](#수집-metric-확인)
6. [주의](#주의)

<br>

## 들어가기 전에

## 실행 환경
### GCP VM
> 해당 모니터링 테스트는 **GCP VM**을 bastion host처럼 사용했습니다.<br>
> `helm`과 `kubectl`은 local bash에서도 실행 가능합니다.

|VM 구분|실행 구분|VM 유형|OS|비고|
|:--|:--|:--|:--:|:--:|
|operator|helm|e2-medium (2 vCPU, 1 Core, 4 Mem)|CentOS7|고정 IP 주소 사용|

### [helm-chart/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
> **v45.7.1** 버전 사용

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm pull prometheus-community/kube-prometheus-stack --version 45.7.1
```


### Amazon EKS
> 프로젝트 배포 및 metric 수집에는 Amazon의 관리형 쿠버네티스 서비스인 EKS를 사용합니다. (프로젝트 배포에 대한 것은 기술하지 않음) <br>

- **EKS 생성 후 awscli에서 접근:**
```shell
aws eks --region ${eks-region} update-kubeconfig \
        --name ${eks-name}
```

<br>

## 수집 metric 확인
### `/federate` http 경로 진입 화면

![http](/prometheus-federation/img/http-federate.png)

### Prometheus에서 Target 확인

![prom](/prometheus-federation/img/prom-federate.png)

### Grafana에서 Dashboard 설정 후
> metric 수집 클러스터의 노드 3개 + 관찰 클러스터 노드 3개 = 총 6개의 노드

![graf1](/prometheus-federation/img/graf-federate1.png)
![graf2](/prometheus-federation/img/graf-federate2.png)

<br>

---

### ingress controller hostname
- **Prometheus**
```shell
kubectl get svc -n monitoring kube-prometheus-stack-prometheus -o \
                   jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
- **Grafana**
```shell
kubectl get svc -n monitoring kube-prometheus-stack-grafana -o \
                   jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

<br>

## 주의
**오퍼레이터 실행 시 사용할 `.yaml` 파일 내용 중에서, metric을 수집 당하는 클러스터의 경우에는 target 설정이 필요가 없습니다.** 그래서 중앙에서 metric을 수집하는 클러스터와 수집 당하는 클러스터의 `.yaml` 파일은 서로 구분해서 저장해두는 것이 편합니다.

- 예를 들어서:
```shell
# 수집 당하는 eks에서
helm install kube-prometheus-stack -f service-values.yaml --namespace monitoring .
# 수집하는 eks에서
helm install kube-prometheus-stack -f monitor-values.yaml --namespace monitoring .
```
