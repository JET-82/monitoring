## Prometheus, Grafana Monitoring in Docker

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [데모용 프로젝트](#데모용-프로젝트)
3. [필요 파일](#필요-파일)
    1. [Springboot](#1-springboot)
    2. [Prometheus, Grafana](#2-prometheus-grafana)

<br>

### 들어가기 전에
1. **Kubernetes의 metric을 수집하기 위한 Prometheus 환경이 아닙니다. Springboot 프로젝트를 모니터링 합니다.** Docker container에서 Prometheus와 Grafana를 실행합니다.
2. 마찬가지로 Springboot 프로젝트 또한 Docker container에서 실행됩니다.

### 실행 환경
#### GCP VM
> 해당 모니터링 테스트는 GCP VM에서 진행했습니다.

|VM 구분|실행 구분|VM 유형|비고|
|:--|:--|:--|:--:|
|Docker1|Springboot proj|e2-medium (2 vCPU, 1 Core, 4 Mem)|고정 IP 주소 사용|
|Docker2|Prometheus, Grafana|e2-medium (2 vCPU, 1 Core, 4 Mem)|고정 IP 주소 사용|

#### 데모용 프로젝트
> 🐳 [ftest5916/team5-deal2:v1.5](https://hub.docker.com/r/ftest5916/team5-deal2/tags) (업데이트 시 갱신 예정) <br>
> ⚠️ 실행할 Springboot 프로젝트가 있는 경우에는 무시하세요.

- docker image pull
```shell
docker pull ftest5916/team5-deal2:v1.1
```
- docker run
```shell
docker run -p 8080:8080 --name springboot ftest5916/team5-deal2:v1.1
```
- 
```
http://${vm-public-ip}:8080/actuator/
```


### 필요 파일
#### 1. Springboot
- prometheus가 metric 정보를 가져올 Springboot 프로젝트
- [application.yml](/prometheus-grafana-in-docker/application.yml)

#### 2. Prometheus, Grafana
- [docker-compose.yml](/prometheus-grafana-in-docker/docker-compose.yml)
- [prometheus.yml](/prometheus-grafana-in-docker/prometheus.yml)