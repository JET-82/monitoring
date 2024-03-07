# Prometheus, Grafana Monitoring in Docker

### 목차
1. [들어가기 전에](#들어가기-전에)
2. [실행 환경](#실행-환경)
    1. [GCP VM](#gcp-vm)
    2. [docker compose 버전](#docker-compose-버전)
3. [필요 파일](#필요-파일)
    1. [Springboot](#springboot)
    2. [Prometheus, Grafana](#prometheus-grafana)
4. [실행 방법](#실행-방법)
    1. [Docker compose](#docker-compose)
    2. [Prometheus 및 Grafana 실행](#prometheus-및-grafana-실행)
    3. [수집 metric 확인](#수집-metric-확인)

5. [그 외](#그-외)
    1. [데모용 프로젝트](#데모용-프로젝트)

<br>

## 들어가기 전에
1. **Kubernetes의 metric을 수집하기 위한 Prometheus 환경이 아닙니다. Springboot 프로젝트를 모니터링 합니다.** Docker container에서 Prometheus와 Grafana를 실행합니다.
2. 마찬가지로 Springboot 프로젝트 또한 Docker container에서 실행됩니다.

<br>

## 실행 환경
### GCP VM
> 해당 모니터링 테스트는 **GCP VM**에서 진행했습니다.

|VM 구분|실행 구분|VM 유형|OS|비고|
|:--|:--|:--|:--:|:--:|
|Docker1|Springboot proj|e2-medium (2 vCPU, 1 Core, 4 Mem)|CentOS7|고정 IP 주소 사용|
|Docker2|Prometheus, Grafana|e2-medium (2 vCPU, 1 Core, 4 Mem)|CentOS7|고정 IP 주소 사용|

### docker compose 버전
> 🐳 [v2.24.4 버전](https://github.com/docker/compose/releases/tag/v2.24.4) 사용

<br>

## 필요 파일
### Springboot
- prometheus가 metric 정보를 가져올 **Springboot 프로젝트**
- Dockerfile: `docker build` 시 필요 (특별한 옵션 필요 X)
- **[application.yml](/prometheus-grafana-in-docker/application.yml):** metric 수집 허용 정보
- **`build.gradle` 의존성** 추가
    ```gradle
    dependencies {
        // ... 생략
        implementation 'org.springframework.boot:spring-boot-starter-actuator'
        runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
    }
    ```

### Prometheus, Grafana
- **[prometheus.yml](/prometheus-grafana-in-docker/prometheus.yml):** Prometheus 설정 파일
- **[docker-compose.yml](/prometheus-grafana-in-docker/docker-compose.yml):** docker-compose 실행 파일

<br>

## 실행 방법
### Docker compose
1. **Docker compose 설치**
    ```shell
    curl -SL https://github.com/docker/compose/releases/download/v2.24.4/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose

    chmod +x /usr/local/bin/docker-compose
    ```
    ```shell
    docker-compose -v  # 버전 확인
    ```
2. **`docker-compose.yml` 파일 작성**
    ```shell
    vi docker-compose.yml
    ```

### Prometheus 및 Grafana 실행
1. **`prometheus.yml` 작성**
    ```shell
    vi prometheus.yml
    ```
2. **`docker-compose.yml` 파일에서 지정한 경로에 따라 `prometheus.yml` 파일 이동**
    ```shell
    mv prometheus.yml /local/path/prometheus.yml
    ```
3. **docker-compose 실행**
    ```shell
    docker-compose up -d
    ```
4. **실행 후, prometheus 컨테이너 내부 경로에서 파일 확인하기**
    ```shell
    docker exec -it ${container} /bin/sh
    ```
    ```shell
    # container 내부 진입 후
    ls /etc/prometheus
    ```
    > 여기에서 `prometheus.yml` 파일이 없거나 or 있어도 내용이 비어있으면, `docker-compose.yml`에서 파일 경로를 다시 확인할 것


### 수집 metric 확인
#### actuator http 경로 진입 화면
```
http://${vm-public-ip}:8080/actuator/prometheus
```
![http](/prometheus-grafana-in-docker/img/http-actuator-prometheus.png)

#### **Prometheus에서 Target 확인**

![prom](/prometheus-grafana-in-docker/img/prom-actuator.png)


#### **Grafana에서 Dashboard 설정 후**

![graf](/prometheus-grafana-in-docker/img/graf-actuator.png)

<br>

## 그 외
### 데모용 프로젝트
> 🐳 [ftest5916/team5-deal2:v1.1](https://hub.docker.com/r/ftest5916/team5-deal2/tags) (업데이트 시 갱신 예정) <br>
> ⚠️ 실행할 Springboot 프로젝트가 있는 경우에는 무시하세요.

- **docker image pull**
```shell
docker pull ftest5916/team5-deal2:v1.1
```
- **docker run**
```shell
docker run -p 8080:8080 --name springboot ftest5916/team5-deal2:v1.1
```