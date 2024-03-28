<!-- 
https://grafana.com/grafana/dashboards/17900-1-kubernetes-all-in-one-cluster-monitoring-kr-v1-26-0/
    1. monitor-eks: 
        - monitor-values.yaml

    helm install kube-prometheus-stack -f service-values.yaml --namespace monitoring .

    2. service-eks:
        - service-values.yaml
        - alert-manager.yaml
        - prometheus-rules.yaml
    
    helm install -n monitoring kube-prometheus-stack . \
    -f prometheus-rules.yaml \
    -f alert-manager.yaml \
    -f service-values.yaml

-->

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
6. [그 외](#그-외)

<br>

## 들어가기 전에
1. helm operator를 사용해 Prometheus와 Grafana를 Amazon EKS(k8s) 환경에서 실행하고 메트릭을 수집합니다.
2. [Prometheus federation](https://prometheus.io/docs/prometheus/latest/federation/) 기능을 사용하여 여러 클러스터로부터 메트릭을 수집합니다.
3. 프로젝트 배포 및 EKS(k8s) 환경 설정에 대한 것은 다루지 않습니다.

<br>

## 실행 환경
### GCP VM
> 해당 모니터링 테스트는 **GCP VM**을 bastion host처럼 사용했습니다.<br>
> `helm`과 `kubectl`은 local bash에서도 실행 가능합니다.

|VM 구분|실행 구분|VM 유형|OS|비고|
|:--|:--|:--|:--:|:--:|
|operator|helm|e2-medium (2 vCPU, 1 Core, 4 Mem)|CentOS7|고정 IP 주소 사용|


### Amazon EKS
> 프로젝트 배포 및 metric 수집에는 Amazon의 관리형 쿠버네티스 서비스인 EKS를 사용합니다. (프로젝트 배포에 대한 것은 기술하지 않음) <br>

|VM 구분|실행 구분|VM 유형|OS|비고|
|:--|:--|:--|:--:|:--:|
|eks managed node group|eks|t3.medium|Amazon Linux 2 AMI|-|

- **EKS 생성 후 awscli에서 접근:**
```shell
aws eks --region ${eks-region} update-kubeconfig \
    --name ${eks-name}
```

### [helm-chart/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
> **v45.7.1** 버전 사용

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm pull prometheus-community/kube-prometheus-stack --version 45.7.1
```

<br>

## 필요 파일
### Prometheus helm 설치 파일 (monitoring cluster)
- **[monitoir-values.yml](/prometheus-federation/monitor-values.yaml):** Prometheus 설정 파일
```shell
helm install -n monitoring kube-prometheus-stack . \
    -f monitoir-values.yaml
```

### Prometheus helm 설치 파일 (service cluster)
- **[service-values.yml](/prometheus-federation/service-values.yaml):** 서비스가 배포된 cluster에서 helm 설치 시 사용할 manifest 파일

<br>

## 실행 방법


<br>

## 수집 metric 확인
### /federate http 경로 진입 화면

![http](/prometheus-federation/img/http-federate.png)

### Prometheus에서 Target 확인

#### case 1. 1개의 EKS만 타겟에 추가
![prom1](/prometheus-federation/img/prom-federate1.png)

#### case 2. 2개의 EKS와 1개의 JMX exporter를 타겟에 추가
![prom2](/prometheus-federation/img/prom-federate2.png)

### Grafana에서 Dashboard 설정 후
> metric 수집 클러스터의 노드 3개 + 관찰 클러스터 노드 3개 = 총 6개의 노드

#### case 1. 1개의 EKS만 타겟에 추가
![graf1](/prometheus-federation/img/graf-federate1.png)
![graf2](/prometheus-federation/img/graf-federate2.png)

#### case 2. JMX exporter metric 확인 (임시)

![jmx1](/prometheus-federation/img/jmx-example1.png)
![jmx2](/prometheus-federation/img/jmx-example2.png)

<br>

---

### ingress controller hostname 조회
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
- **svc 전체 조회**
```shell
kubectl get svc -n monitoring
```


<br>

## 그 외
**오퍼레이터 실행 시 사용할 `.yaml` 파일 내용 중에서, metric을 수집 당하는 클러스터의 경우에는 target 설정이 필요가 없습니다.** 그래서 중앙에서 metric을 수집하는 클러스터와 수집 당하는 클러스터의 `.yaml` 파일은 서로 구분해서 저장해두는 것이 편합니다.

- 예를 들어서:
```shell
# 수집 당하는 eks에서
helm install kube-prometheus-stack -f service-values.yaml --namespace monitoring .
# 수집하는 eks에서
helm install kube-prometheus-stack -f monitor-values.yaml --namespace monitoring .
```

<br>

helm 설치 시 사용한 `.yaml` 파일을 수정하고 재적용시키려는 경우, `helm upgrade` 명령어를 사용할 수 있습니다. 이 때, `-f` 옵션을 사용하여 수정한 모든 `.yaml` 파일을 호출해주어야합니다. 그렇지 않으면 호출되지 않은 파일의 내용은 클러스터에서 제거됩니다. <br>

예를 들어, 위의 `monitor-values.yaml` 파일에서 prometheus, grafana 설정을 두 개의 파일로 분리했다고 가정합니다. 그렇다면 다음과 같이 명령어를 사용할 수 있습니다.

```shell
helm upgrade -n monitoring kube-prometheus-stack . \
    -f grafana.yaml \
    -f prometheus.yaml
```

파일의 우선 순위는 뒤로 갈 수록 높아집니다.