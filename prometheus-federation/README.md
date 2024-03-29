# Prometheus Federation

**Prometheus Federation**을 사용하여 아래 이미지와 같은 중앙 집중식 메트릭 수집을 목표로 합니다. <br>

![architecture-federate](/prometheus-federation/img/architecture-federate.png)

- 한 곳에서 모든 클러스터의 메트릭을 관측하기 때문에, 모든 메트릭을 수집하도록 하면, 수집 클러스터(여기서는 모니터링 클러스터)에 부하가 올 수 있습니다. 많은 수의 클러스터를 관측한다면, 한정된 메트릭만을 수집하도록 설정 파일을 수정하는 것을 권장합니다.
- 더 나은 중앙 집중 메트릭 수집 솔루션으로는 [Thanos](https://thanos.io/) 등이 있습니다.


## 공식 문서
Prometheus Federation에 대한 문서는 [여기](https://prometheus.io/docs/prometheus/latest/federation/)를 확인하세요.<br>

## 참고 문서
- [SeSAC-AWS-Final-Team-2/dev-eks](https://github.com/SeSAC-AWS-Final-Team-2/dev-eks/blob/main/third-party/monitoring.md)
- [choisungwook/portfolio/kubernetes/helm/prometheus-charts](https://github.com/choisungwook/portfolio/tree/master/kubernetes/helm/prometheus-charts)
- https://github.com/prometheus-operator/prometheus-operator/issues/2877

<br>

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [Amazon EKS](#amazon-eks)
    3. [helm-chart/kube-prometheus-stack](#helm-chartkube-prometheus-stack)
3. [필요 파일](#필요-파일)
    1. [Monitoring 클러스터의 경우](#monitoring-클러스터의-경우)
    2. [Service 클러스터의 경우](#service-클러스터의-경우)
4. [실행 방법](#실행-방법)
    1. [클러스터에 메트릭 서버 추가](#클러스터에-메트릭-서버-추가)
    2. [helm 설치 및 repo 추가](#helm-설치-및-repo-추가)
    3. [Prometheus를 배포할 EKS 연결](#prometheus를-배포할-eks-연결)
    4. [helm operator를 통해 Prometheus 배포](#helm-operator를-통해-prometheus-배포)
    5. [배포된 Prometheus 및 Grafana의 hostname 확인](#배포된-prometheus-및-grafana의-hostname-확인)
5. [수집 metric 확인](#수집-metric-확인)
6. [그 외](#그-외)
    1. [monitor와 service 클러스터 파일 분리 이유](#monitor와-service-클러스터-파일-분리-이유)
    2. [helm upgrade](#helm-upgrade)

<br>

## 들어가기 전에
1. helm operator와 `kube-prometheus-stack` chart를 사용해 Prometheus와 Grafana를 Amazon EKS(k8s) 환경에서 실행하고 메트릭을 수집합니다.
2. [Prometheus federation](https://prometheus.io/docs/prometheus/latest/federation/) 기능을 사용하여 여러 클러스터로부터 메트릭을 수집합니다.
3. 프로젝트 배포 및 EKS(k8s) 환경 설정에 대한 것은 다루지 않습니다.

<br>

## 실행 환경
### GCP VM
> 해당 모니터링 테스트는 **GCP VM**을 bastion host처럼 사용했습니다.<br>
> `helm`과 `kubectl`은 local bash에서도 실행 가능합니다.

|VM 구분|실행 구분|VM 유형|OS|비고|
|:--|:--|:--|:--:|:--:|
|operator|helm|e2-medium (2 vCPU, 1 Core, 4GiB Mem)|CentOS7|고정 IP 주소 사용|


### Amazon EKS
> 프로젝트 배포 및 metric 수집에는 Amazon의 관리형 쿠버네티스 서비스인 EKS를 사용합니다. (프로젝트 배포에 대한 것은 기술하지 않음) <br>

|VM 구분|실행 구분|VM 유형|OS|비고|
|:--|:--|:--|:--:|:--:|
|eks managed node group|eks|t3.medium (2 vCPU, 1 Core, 4GiB Mem)|Amazon Linux 2 AMI|-|

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
### Monitoring 클러스터의 경우
- **[monitoir-values.yml](/prometheus-federation/monitor-values.yaml):** 모니터링에 사용할 클러스터에서 helm chart 설치 시 사용할 manifest 파일

```shell
helm install -n monitoring kube-prometheus-stack . \
    -f monitoir-values.yaml
```
- node-exporter 비활성화 (필요에 따라 활성화해도 무관함)
- alertmanager 비활성화 (필요에 따라 활성화해도 무관함)
- **Prometheus Federation 사용 설정:** `prometheus.prometheusSpec.additionalScrapeConfigs:` 부분 참고


### Service 클러스터의 경우
- **[service-values.yml](/prometheus-federation/service-values.yaml):** 서비스가 배포된 클러스터에서 helm chart 설치 시 사용할 manifest 파일

```shell
helm install -n monitoring kube-prometheus-stack . \
    -f service-values.yaml
```

<br>

## 실행 방법
### 클러스터에 메트릭 서버 추가
- **Deploy Metrics Server**
```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
- **확인**
```shell
kubectl get deployment metrics-server -n kube-system
```

### helm 설치 및 repo 추가
> [helm 설치 가이드 - helm docs](https://helm.sh/ko/docs/intro/install/)
1. **helm 설치 (CentOS7)**
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
```
```shell
chmod 700 get_helm.sh
./get_helm.sh
```
<details>
    <summary>설치 중 오류가 발생한다면:</summary>

<br>

방법 1. openssl 패키지 설치
```shell
yum install -y openssl
```

방법 2. 환경 변수에서 `VERIFY_CHECKSUM`을 `false`로 설정
```shell
export VERIFY_CHECKSUM=false
```

</details>

<br>

- **helm 버전 확인**
```shell
helm version
```
```shell
# 결과
version.BuildInfo{Version:"v3.14.2", GitCommit:"c309b6f0ff63856811846ce18f3bdc93d2b4d54b", GitTreeState:"clean", GoVersion:"go1.21.7"}
```

2. **helm repo 추가**
```shell
# [prometheus-community] repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
```shell
helm repo update
```
```shell
# 결과
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### Prometheus를 배포할 EKS 연결
```shell
aws eks --region ${eks-region} update-kubeconfig \
    --name ${eks-name}
```

### helm operator를 통해 Prometheus 배포
1. **namespace 추가**
```shell
kubectl create ns monitoring
```

2. **`.tgz` 형태로 helm chart 파일 가져오기**
```shell
helm pull prometheus-community/kube-prometheus-stack --version 45.7.1     
tar -xvzf kube-prometheus-stack-45.7.1.tgz 
cd kube-prometheus-stack         
```

3. **values 파일 작성**
- [monitor-values.yaml](/prometheus-federation/monitor-values.yaml)
    - **Prometheus Federation 사용 설정:** `prometheus.prometheusSpec.additionalScrapeConfigs:` 부분 참고
- [service-values.yaml](/prometheus-federation/service-values.yaml)

4. **values 파일과 namespace 포함하여 helm 배포**
```shell
helm install -n monitoring kube-prometheus-stack . \
    -f monitoir-values.yaml
```
혹은
```shell
helm install -n monitoring kube-prometheus-stack . \
    -f service-values.yaml
```

### 배포된 Prometheus 및 Grafana의 hostname 확인
```shell
# prometheus
kubectl get svc -n monitoring kube-prometheus-stack-prometheus -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# grafana
kubectl get svc -n monitoring kube-prometheus-stack-grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
```shell
# 전체 svc 확인
kubectl get svc -n monitoring
```
명령어 실행 결과로 나오는 `hostname` 값을 통해서 외부 브라우저에서 접근합니다. **(주소 뒤에 포트 번호 추가 필요)**

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
### monitor와 service 클러스터 파일 분리 이유
**오퍼레이터 실행 시 사용할 `.yaml` 파일 내용 중에서, metric을 수집 당하는 클러스터의 경우에는 target 설정이 필요가 없습니다.** 그래서 중앙에서 metric을 수집하는 클러스터와 수집 당하는 클러스터의 `.yaml` 파일은 서로 구분해서 저장해두는 것이 편합니다.

- 예를 들어서:
```shell
# 수집 당하는 eks에서
helm install kube-prometheus-stack -f service-values.yaml --namespace monitoring .
# 수집하는 eks에서
helm install kube-prometheus-stack -f monitor-values.yaml --namespace monitoring .
```

<br>

### helm upgrade
helm 설치 시 사용한 `.yaml` 파일을 수정하고 재적용시키려는 경우, `helm upgrade` 명령어를 사용할 수 있습니다. 이 때, `-f` 옵션을 사용하여 수정한 모든 `.yaml` 파일을 호출해주어야합니다. 그렇지 않으면 호출되지 않은 파일의 내용은 클러스터에서 제거됩니다. <br>

예를 들어, 위의 `monitor-values.yaml` 파일에서 prometheus, grafana 설정을 두 개의 파일로 분리했다고 가정합니다. 그렇다면 다음과 같이 명령어를 사용할 수 있습니다.

```shell
helm upgrade -n monitoring kube-prometheus-stack . \
    -f grafana.yaml \
    -f prometheus.yaml
```

파일의 우선 순위는 뒤로 갈 수록 높아집니다. 
