## JMX monitoring in Docker

### 목차
1. 들어가기 전에
2. 실행 환경
    1. GCP VM
    2. JMX Exporter
    3. 데모용 프로젝트
3. 필요 파일
    1. Springboot
    2. Prometheus, Grafana


### 들어가기 전에
1. 현재 버전은 Docker container에서 Springboot 프로젝트를 실행하는 것을 기준으로 진행합니다. (추후 k8s 버전도 갱신 예정)
2. Prometheus 및 Grafana도 Docker container에서 실행됩니다.

### 실행 환경
#### GCP VM
> 해당 모니터링 테스트는 GCP VM에서 진행했습니다.

|VM 구분|실행 구분|VM 유형|
|:--|:--|:--|
|Docker1|Springboot proj|e2-medium (2 vCPU, 1 Core, 4 Mem)|
|Docker2|Prometheus, Grafana|e2-medium (2 vCPU, 1 Core, 4 Mem)|

#### JMX Exporter
> 🕵️ [jmx_prometheus_javaagent-0.20.0 버전](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar) (모니터링 테스트 당시 최신 버전)

#### 데모용 프로젝트
> 🐳 [ftest5916/team5-deal2:v1.5](https://hub.docker.com/r/ftest5916/team5-deal2/tags) (업데이트 시 갱신 예정) <br>
> ⚠️ 실행할 Springboot 프로젝트가 있는 경우에는 무시하세요.

- docker image pull
```shell
docker pull ftest5916/team5-deal2:v1.5
```
- docker run
```shell
docker run -p 8080:8080 -p 9090:9090 --name springboot ftest5916/team5-deal2:v1.5
```

- docker container 조회
```shell
docker container ls
```
```
# 조회 결과
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS          PORTS                                                                                  NAMES
ae8a40dbbfe2   ftest5916/team5-deal2:v1.5   "java -javaagent:/ap…"   59 minutes ago   Up 59 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:9090->9090/tcp, :::9090->9090/tcp   springboot
```

### 필요 파일
#### 1. Springboot
- jmx가 관찰할 Springboot 프로젝트
- [Dockerfile](/jmx-exporter/Dockerfile)
- [config.yaml](/jmx-exporter/config.yaml)

#### 2. Prometheus, Grafana
- [Prometheus, Grafana Monitoring in Docker](/prometheus-grafana-in-docker/README.md) 참고