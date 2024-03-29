# Prometheus Alert Manager - Slack Incoming Webhook

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [Amazon EKS](#amazon-eks)
    3. [helm-chart/kube-prometheus-stack](#helm-chartkube-prometheus-stack)
3. [필요 파일](#필요-파일)
4. [실행 방법](#실행-방법)

<br>

## 들어가기 전에
1. **Slack의 Incoming Webhook(수신 웹후크)** app을 사용하여 alert(경보)를 전달하는 내용을 다룹니다.
2. helm operator와 `kube-prometheus-stack` chart를 사용해 Prometheus와 Grafana를 Amazon EKS(k8s) 환경에서 실행하고 메트릭을 수집합니다. (* 프로젝트 배포 및 EKS(k8s) 환경 설정에 대한 것은 다루지 않습니다.) 
    - 자세한 내용은 [prometheus-federation](/prometheus-federation/)에서 다루고 있습니다.

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

## 필요 파일
[helm-chart/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)를 사용하여 Prometheus를 배포하기 때문에, 설정 `.yaml` 파일은 helm chart 사용을 기준으로 작성되었습니다.

- **[prometheus-rules.yaml](/prometheus-slack-alert/prometheus-rules.yaml):** `additionalPrometheusRulesMap` 항목에 alert(경보) 규칙 설정
- **[alertmanager.yaml](/prometheus-slack-alert/alertmanager.yaml):** alert(경보) 수신 설정
- **[values.yaml](/prometheus-federation/service-values.yaml):** Prometheus, Grafana 배포 설정

```shell
helm install -n monitoring kube-prometheus-stack . \
    -f prometheus-rules.yaml \
    -f alertmanager.yaml \
    -f values.yaml
```

## 실행 방법

### 