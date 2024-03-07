## JMX monitoring in Docker

### 들어가기 전에
1. 현재 버전은 Docker container에서 Springboot 프로젝트를 실행하는 것을 기준으로 진행합니다. (추후 k8s 버전도 갱신 예정)
2. Prometheus 및 Grafana도 Docker container에서 실행합니다.

### 실행 환경
#### GCP VM
> 해당 모니터링 테스트는 GCP VM에서 진행했습니다.

|VM 구분|실행 구분|VM 유형|
|:--|:--|:--|
|Docker1|Springboot proj|e2-medium (2 vCPU, 1 Core, 4 Mem)|
|Docker2|Prometheus, Grafana|e2-medium (2 vCPU, 1 Core, 4 Mem)|

#### JMX Exporter
- [0.20.0 버전](https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar) 사용 (모니터링 테스트 당시 최신 버전)

#### 데모용 프로젝트
- [ftest5916/team5-deal2:v1.5](https://hub.docker.com/r/ftest5916/team5-deal2/tags) 사용 (업데이트 시 갱신)
- 사용할 springboot 프로젝트가 있다면 직접 작성해보길 권장합니다.

```shell
# docker image pull
docker pull ftest5916/team5-deal2:v1.5
```
```shell
# docker run
docker run -p 8080:8080 -p 9090:9090 --name ${container-name} ${docker-image-name}
docker run -p 8080:8080 -p 9090:9090 --name springboot ftest5916/team5-deal2:v1.5
```
```shell
# docker container 조회
docker container ls
# 조회 결과
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS          PORTS                                                                                  NAMES
ae8a40dbbfe2   ftest5916/team5-deal2:v1.5   "java -javaagent:/ap…"   59 minutes ago   Up 59 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:9090->9090/tcp, :::9090->9090/tcp   springboot
```