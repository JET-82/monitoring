FROM ${java-base-image}

WORKDIR /app
# 수정 필요
ADD /local/path/jmx_prometheus_javaagent-0.20.0.jar /app/jmx_exporter.jar
# 수정 필요
ADD /local/path/config.yaml /app/config.yaml

ARG JAR_FILE=${springboot_jar}
COPY ${JAR_FILE} .
# 수정 필요
ENTRYPOINT ["java","-javaagent:/app/jmx_exporter.jar=9090:/app/config.yaml","-jar","${springboot_jar}"]