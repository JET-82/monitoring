# Prometheus Alertmanager - Slack Incoming Webhook

## 공식 문서
- [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/overview/)

## 참고 문서
- [Prometheus Alerting - 토리맘의 한글라이즈 프로젝트](https://godekdls.github.io/Prometheus/alerting/)
- [1 Kubernetes All-in-one Cluster Monitoring KR](https://grafana.com/grafana/dashboards/13770-1-kubernetes-all-in-one-cluster-monitoring-kr/)

<br>

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [Amazon EKS](#amazon-eks)
    3. [helm-chart/kube-prometheus-stack](#helm-chartkube-prometheus-stack)
3. [필요 파일](#필요-파일)
4. [실행 방법](#실행-방법)
    1. [Slack Webhook 설정 (Create an App)](#slack-webhook-설정-create-an-app)
    2. [설정 파일을 포함하여 Prometheus 배포](#설정-파일을-포함하여-prometheus-배포)
    3. [CPU 부하 테스트를 통한 메시지 수신 확인](#cpu-부하-테스트를-통한-메시지-수신-확인)

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

<br>

## 필요 파일
[helm-chart/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)를 사용하여 Prometheus를 배포하기 때문에, 설정 `.yaml` 파일은 helm chart 사용을 기준으로 작성되었습니다. `.yaml` 파일은 `-f` 옵션을 사용하여 아래처럼 호출합니다.

```shell
helm install -n monitoring kube-prometheus-stack . \
    -f prometheus-rules.yaml \
    -f alertmanager.yaml \
    -f values.yaml
```

1. **[prometheus-rules.yaml](/prometheus-slack-alert/prometheus-rules.yaml):** `additionalPrometheusRulesMap` 항목에 alert(경보) 규칙 설정
```yaml
additionalPrometheusRulesMap:
  cpuusagerules:
    groups:
      - name: cpu-usage-warning  # group name
        rules:  # 아래부터 규칙 설정
          - alert: cpu-usage_45  # alert 구분: alertmanager.yaml에서 이 이름을 참고합니다.
            # expr: 규칙 계산식
            expr: (sum by (instance,nodename) (irate(node_cpu_seconds_total{mode!~"guest.*|idle|iowait"}[1m])) + on(instance) group_left(nodename) node_uname_info - 1) > 0.45
            for: 1m
            labels:
              severity: warning
            annotations:  # 경보 메시지 정보
              # summary: 주제, description: 부가 설명
              summary: "{{ $labels.instance }} CPU 사용량 45% 초과"
              description: "CPU 사용량이 45%를 초과하였습니다."
      # - name: 
```
```yaml
# vCPU 사용률 계산
expr: (sum by (instance,nodename) (irate(node_cpu_seconds_total{mode!~"guest.*|idle|iowait"}[1m])) + on (instance) group_left(nodename) node_uname_info - 1) > 0.45
```
<details>
    <summary> 계산식 해설</summary>

<br>

- **`sum by (instance, nodename) (...)`:** 각 인스턴스와 노드 이름별로 CPU 사용률의 합을 구합니다.
- **`irate(...) [1m]`:** 선택된 시간 범위(1m) 동안의 증가율(irate=instant rate)을 계산합니다.
- **`node_cpu_seconds_total{mode!~"guest.*|idle|iowait"}`:** CPU 사용 모드 중 `guest`로 시작하는 모든 모드, `idle` 및 `iowait` 상태를 제외한 CPU 시간(초)을 측정합니다. (guest 모드는 가상화 환경에서 작동하는 가상 CPU(vCPU)의 사용을 나타냅니다.)

- **`on (instance) group_left(nodename)`:** `sum by (...)`로부터 얻은 결과와 `node_uname_info` 메트릭을 `instance`를 기준으로 left join하면서 `nodename`을 유지합니다. (`group_left`는 조인하는 두 쪽 중 왼쪽(첫 번째 메트릭)에 레이블이 없는 경우에 사용)
    - `node_uname_info` 메트릭은 시스템에 대한 정보(예: 운영 체제, 노드 이름 등)를 제공합니다. (이 정보는 레이블로 포함되어 있으며, 여기서는 주로 `nodename`을 사용.)

- **마지막에 `1`을 빼는 이유:** `node_uname_info`가 실제로 CPU 사용에 기여하지 않기 때문에, 이를 보정하기 위함입니다.
- **`( ... ) > 0.45`:** 최종적으로 계산된 값이 `0.45`를 초과하는 경우에 경고가 발생합니다. (0.45 = 45%)

</details>

<br>

```yaml
annotations:
  summary: "{{ $labels.instance }} CPU 사용량 45% 초과"
```
- **`{{ $labels.instance }}`:** 경고가 발생한 인스턴스의 이름을 의미합니다.

<br>

2. **[alertmanager.yaml](/prometheus-slack-alert/alertmanager.yaml):** alert(경보) 수신 설정
```yaml
receivers:
  - name: 'default-receiver'  # default alert에 대한 receiver 설정
    slack_configs:
      # Slack api 에서 발급받은 url
      - api_url: '${#alert-heartbeat-channel-api-url}' # https://hooks.slack.com/services/...
        # Slack Channel 명
        channel: '#alert-heartbeat'
        send_resolved: true  # 해소된(resolved) alert도 통보할지 여부, default = false
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
  - name: 'cpu-usage_45'  # custom alert에 대한 receiver 설정
    slack_configs:
      - api_url: '${#alert-warning-channel-api-url}' # https://hooks.slack.com/services/...
        channel: '#alert-warning'
        send_resolved: true
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
```
- **`title: '{{ .CommonAnnotations.summary }}'`:** `prometheus-rules.yaml` 파일에 작성된 `rules.annotations.summary` 정보를 가져와 Slack 경보 메시지로 출력합니다.
- **`text: '{{ .CommonAnnotations.description }}'`:** `prometheus-rules.yaml` 파일에 작성된 `rules.annotations.description` 정보를 가져와 Slack 경보 메시지로 출력합니다.

<br>

3. **[values.yaml](/prometheus-federation/service-values.yaml):** Prometheus, Grafana 배포 설정


<br>

## 실행 방법

### Slack Webhook 설정 (Create an App)
1. **https://api.slack.com/apps 접속**
2. **[Create an App] 버튼 클릭**

![slackapi1](/prometheus-slack-alert/img/slackapi1.png)

3. **[From scratch] 선택**

![slackapi2](/prometheus-slack-alert/img/slackapi2.png)

4. **App Name 작성, Slack 워크스페이스 선택**

![slackapi3](/prometheus-slack-alert/img/slackapi3.png)

5. **[Incoming Webhooks] → [App New Webhook to Workspace] 클릭**

![slackapi4](/prometheus-slack-alert/img/slackapi4.png)

6. **Incoming Webhook가 액세스할 Slack 채널 선택**

![slackapi5](/prometheus-slack-alert/img/slackapi5.png)

7. **부여된 Webhook URL을 사용하여 메시지 전송 테스트**

![slackapi6](/prometheus-slack-alert/img/slackapi6.png)
```shell
curl -X POST -H 'Content-type: application/json' --data '{"text":"hello world"}' https://hooks.slack.com/services/...
```
**(주의)** Webhook URL은 외부에 노출되지 않도록 주의합니다.

8. **채널에서 전송된 메시지 확인**

![slackapi7](/prometheus-slack-alert/img/slackapi7.png)

<br>

### 설정 파일을 포함하여 Prometheus 배포

```shell
helm install -n monitoring kube-prometheus-stack . \
    -f prometheus-rules.yaml \
    -f alertmanager.yaml \
    -f values.yaml
```

### CPU 부하 테스트를 통한 메시지 수신 확인
CPU에 임의로 부하를 주어 Alertmanager 설정이 잘 적용되었는지 확인합니다. 저는 [nGrinder](https://naver.github.io/ngrinder/)를 사용하였습니다.
- nGrinder 사용과 관련하여 작성한 블로그 글은 [여기](https://jungeun5-choi.github.io/categories/#ngrinder)를 참고해주세요.

![slack-message1](/prometheus-slack-alert/img/slack-message1.png)