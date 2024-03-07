# JMX monitoring in Docker

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [JMX Exporter](#JMX-Exporter)
3. [필요 파일](#필요-파일)
    1. [Springboot](#1-springboot)
    2. [Prometheus, Grafana](#2-prometheus-grafana)
4. [그 외](#그-외)    
    1. [데모용 프로젝트](#데모용-프로젝트)

<br>

## 들어가기 전에
1. **[Prometheus, Grafana Monitoring in Docker](/prometheus-grafana-in-docker/README.md)에서 JMX exporter 모니터링을 추가한 버전입니다.** 마찬가지로 Docker container에서 Prometheus와 Grafana를 실행합니다.
2. 현재 버전은 Docker container에서 Springboot 프로젝트를 실행하는 것을 기준으로 진행합니다. (추후 k8s 버전도 갱신 예정)


## 실행 환경
### GCP VM
> 해당 모니터링 테스트는 **GCP VM**에서 진행했습니다.

|VM 구분|실행 구분|VM 유형|비고|
|:--|:--|:--|:--:|
|Docker1|Springboot proj|e2-medium (2 vCPU, 1 Core, 4 Mem)|고정 IP 주소 사용|
|Docker2|Prometheus, Grafana|e2-medium (2 vCPU, 1 Core, 4 Mem)|고정 IP 주소 사용|

### JMX Exporter
> 🕵️ [jmx_prometheus_javaagent-0.20.0 버전](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar) (모니터링 테스트 당시 최신 버전)

## 필요 파일
### 1. Springboot
- prometheus가 metric 정보를 가져올 **Springboot 프로젝트**
- **[application.yml](/jmx-exporter/application.yml):** metric 수집 허용 정보
- **`build.gradle` 의존성** 추가
```gradle
dependencies {
    // ... 생략
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
}
```
- **[Dockerfile](/jmx-exporter/Dockerfile):** `docker build` 시 필요
- **[config.yaml](/jmx-exporter/config.yaml):** `docker build` 시 필요

### 2. Prometheus, Grafana
- **[prometheus.yml](/jmx-exporter/prometheus.yml):** Prometheus 설정 파일
- **[docker-compose.yml](/prometheus-grafana-in-docker/docker-compose.yml):** docker-compose 실행 파일


## 그 외
### 데모용 프로젝트
> 🐳 [ftest5916/team5-deal2:v1.5](https://hub.docker.com/r/ftest5916/team5-deal2/tags) (업데이트 시 갱신 예정) <br>
> ⚠️ 실행할 Springboot 프로젝트가 있는 경우에는 무시하세요.

- **docker image pull**
```shell
docker pull ftest5916/team5-deal2:v1.5
```
- **docker run**
```shell
docker run -p 8080:8080 -p 9090:9090 --name springboot ftest5916/team5-deal2:v1.5
```

- **docker container 조회하여 포트 번호 확인**
```shell
docker container ls
```
```
# 조회 결과
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS          PORTS                                                                                  NAMES
ae8a40dbbfe2   ftest5916/team5-deal2:v1.5   "java -javaagent:/ap…"   59 minutes ago   Up 59 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:9090->9090/tcp, :::9090->9090/tcp   springboot
```