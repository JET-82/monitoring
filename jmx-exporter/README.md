# JMX monitoring in Docker

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [Docker compose](#docker-compose)
    2. [JMX Exporter](#JMX-Exporter)
3. [필요 파일](#필요-파일)
    1. [Springboot](#1-springboot)
    2. [Prometheus, Grafana](#2-prometheus-grafana)
4. [실행 방법](#실행-방법)
    1. [Springboot 프로젝트 설정 파일 작성](#springboot-프로젝트-설정-파일-작성)
    2. [Docker compose 설정](#docker-compose-설정)
    3. [Prometheus 및 Grafana 실행](#prometheus-및-grafana-실행)
    4. [Springboot 프로젝트 실행](#springboot-프로젝트-실행)
    5. [수집 metric 확인](#수집-metric-확인)
5. [그 외](#그-외)    
    1. [데모용 프로젝트](#데모용-프로젝트)

<br>

## 들어가기 전에
1. **[Prometheus, Grafana Monitoring in Docker](/prometheus-grafana-in-docker/README.md)에서 JMX exporter 모니터링을 추가한 버전입니다.** 마찬가지로 Docker container에서 Prometheus와 Grafana를 실행합니다.
2. 현재 버전은 Docker container에서 Springboot 프로젝트를 실행하는 것을 기준으로 진행합니다. (추후 k8s 버전도 갱신 예정)


## 실행 환경
### GCP VM
> 해당 모니터링 테스트는 **GCP VM**에서 진행했습니다.

|VM 구분|실행 구분|VM 유형|OS|비고|
|:--|:--|:--|:--:|:--:|
|Docker1|Springboot proj|e2-medium (2 vCPU, 1 Core, 4 Mem)|CentOS7|고정 IP 주소 사용|
|Docker2|Prometheus, Grafana|e2-medium (2 vCPU, 1 Core, 4 Mem)|CentOS7|고정 IP 주소 사용|

### Docker compose
> 🐳 [v2.24.4 버전](https://github.com/docker/compose/releases/tag/v2.24.4) 사용

### JMX Exporter
> 🕵️ [jmx_prometheus_javaagent-0.20.0 버전](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar) (모니터링 테스트 당시 최신 버전)

<br>

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

<br>

## 실행 방법
### Springboot 프로젝트 설정 파일 작성
1. `build.gradle` 의존성 추가
2. `application.yml` 수정
3. `.jar` 파일 빌드
4. `Dockerfile` 사용하여 Docker 빌드 및 실행

### Docker compose 설정
- [Prometheus, Grafana Monitoring in Docker의 Grafana Docker compose 설정](/prometheus-grafana-in-docker/README.md#docker-compose-설정) 참고
- `docker-compose.yml`파일은 현재 디렉토리의 파일을 사용합니다.

### Prometheus 및 Grafana 실행
- [Prometheus, Grafana Monitoring in Docker의 Prometheus 및 Grafana 실행](/prometheus-grafana-in-docker/README.md#prometheus-및-grafana-실행) 참고
- `prometheus.yml`파일은 현재 디렉토리의 파일을 사용합니다.

### Springboot 프로젝트 실행
```shell
docker run -p 8080:8080 -p 9090:9090 --name ${container-name} ${image-name}
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


### 수집 metric 확인
#### jmx http 경로 진입 화면

![http](/jmx-exporter/img/http-jmx-exporter.png)

#### Prometheus에서 Target 확인

![prom](/jmx-exporter/img/prom-jmx-exporter.png)


#### Grafana에서 Dashboard 설정 후

![graf](/jmx-exporter/img/graf-jmx-dashboard.png)

<br>

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