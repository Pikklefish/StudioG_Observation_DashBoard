# Implementing Observability with Grafana Stack
원본 소스 코드를 보려면 [SaiUpadhyayula's GitHub Repository](https://github.com/SaiUpadhyayula/springboot3-observablity) 

> [!NOTE]
> 이 프로젝트는 [**스튜디오갈릴레이**](https://www.studiog.kr/) 
> 기업부설연구소 (Create G)의 용도로 변경되었습니다.

마지막 수정 (2024-08-07)


## 기본 개요
- Grafana Loki: 로그용 데이터베이스 (LOGQL)
- Prometheus: 메트릭용 데이터베이스 (PROMQL)
- Grafana Tempo: 트레이스용 데이터베이스 (TRACEQL)
- Grafana: 모든 정보를 보기 위한 대시보드

![StudioG_Observation_DashBoard _GitHub.png](images/StudioG_Observation_DashBoard%20_GitHub.png)
## 설정
이 설정은 **Maven** 환경에 특화되어 있으며, Gradle을 사용하는 경우 종속성을 재확인하십시오. 컨테이너에는 **Docker**가 사용되며, 프로덕션 수준에서는 
Kubernetes를 선호합니다.

### 의존성
> [!IMPORTANT] 
> 각 서비스의 `pom.xml` 파일에 다음 의존성을 추가하십시오.

**Loki 의존성** 
```
<dependency>
    <groupId>com.github.loki4j</groupId>
    <artifactId>loki-logback-appender</artifactId>
    <version>1.5.2</version>
</dependency>
```
최신 버전을 확인하려면 이 라이브러리를 확인하십시오: [Maven Repository](https://mvnrepository.com/artifact/com.github.loki4j/loki-logback-appender)

**Prometheus 의존성**

*Micrometer Prometheus*
```
<dependency>
 <groupId>io.micrometer</groupId>
 <artifactId>micrometer-registry-prometheus</artifactId>
 <scope>runtime</scope>
</dependency>
```

*Spring Actuator: SpringBoot Actuator는 실행 중인 애플리케이션에 대한 운영 정보를 노출합니다.*
```
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Tempo 의존성**

*Micrometer Tracing*
```
<dependency>
 <groupId>io.micrometer</groupId>
 <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<dependency>
 <groupId>io.zipkin.reporter2</groupId>
 <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```
`micrometer-tracing-bridge-brave` 는 distributed tracing을 위해 traceID를 자동으로 추가합니다. `zipkin-reporter-brave`
는 trace 정보를 Tempo로 내보냅니다.
>[!NOTE]
> `Opentelemetry-micrometer-tracing-bridge-otel`을 `zipkin-reporter-brave` 대신 사용할 수 있습니다.

데이터베이스 호출을 추적하려면 다음 의존성을 추가하십시오
```
<dependency>
    <groupId>net.ttddyy.observation</groupId>
    <artifactId>datasource-micrometer-spring-boot</artifactId>
    <version>1.0.5</version>
</dependency>
```
최신 버전을 확인하려면 이 라이브러리를 확인하십시오: [Maven Repository](https://mvnrepository.com/artifact/net.ttddyy.observation/datasource-micrometer-spring-boot)

통합의 용이성을 위해 AOP 의존성을 다운로드하십시오
```
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### LOKI 설정

먼저, `src/main/resources` 디렉토리 안에 `logback-spring.xml` 파일을 생성하십시오. 이 파일에는 로그를 구조화하는 방법과 로그를 어디로 
보낼지에 대한 필요한 정보가 포함됩니다 (LOKI URL 정보 포함).
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    <springProperty scope="context" name="appName" source="spring.application.name"/>
 
    <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
        <http>
            <url>http://localhost:3100/loki/api/v1/push</url>
        </http>
        <format>
            <label>
                <pattern>application=${appName},host=${HOSTNAME},level=%level</pattern>
            </label>
            <message>
                <pattern>${FILE_LOG_PATTERN}</pattern>
            </message>
            <sortByTime>true</sortByTime>
        </format>
    </appender>
 
    <root level="INFO">
        <appender-ref ref="LOKI"/>
    </root>
</configuration>
```
둘째, 두 서비스의 루트 디렉토리에 `docker-compose.yaml` 파일을 생성하십시오. 이 파일에는 다음 한 줄이 포함되어야 합니다: `services:`

셋째, `docker-compose.yaml` 파일의 `services:` 아래에 다음 Loki 구성을 추가하십시오
``` 
loki:
  image: grafana/loki:main
  command: [ "-config.file=/etc/loki/local-config.yaml" ] #Configuration file mapping
  ports:
    - "3100:3100"
```

마지막으로, 이것이 Loki의 구성 파일입니다. 이 파일은 로컬 컴퓨터 또는 Docker 컨테이너에 존재할 수 있습니다. `docker-compose.yaml`에서
Loki가 구성 파일의 위치에 올바르게 매핑되어 있는지 확인하십시오.

``` 
auth_enabled: false

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/usagestats/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
#analytics:
#  reporting_enabled: false
```

### PROMETHEUS 설정

> [!IMPORTANT]
>  각 서비스의 `application.properties` 파일에 다음 속성을 추가하십시오
``` 
management.endpoints.web.exposure.include=health, info, metrics, prometheus
management.metrics.distribution.percentiles-histogram.http.server.requests=true
management.observations.key-values.application= <your_service_name>
```

`web.exposure`는 health, info, metrics 및 Prometheus를 actuator 노출합니다. `metrics.distribution.percentiles-histogram`은 
히스토그램 형태로 메트릭을 수집하여 Prometheus로 보냅니다. Micrometer는 metric에 대한 추가 히스토그램 bucket을 기록하므로, 이는 평균이 오해를 
일으킬 수 있는 latency에 유용합니다.

`docker-compose.yaml`에 Prometheus 구성을 추가하십시오.
``` 
prometheus:
  image: prom/prometheus:v2.53.1
  command:
    - --enable-feature=exemplar-storage
    - --config.file=/etc/prometheus/prometheus.yml
  volumes:
    - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
  ports:
    - "9090:9090"
```
최신 버전을 확인하려면 이 라이브러리를 확인하십시오: [Prometheus GitHub](https://github.com/prometheus/prometheus/releases)

마지막으로, 서비스를 공유하는 루트 디렉토리에 있는 `docker` 디렉토리에 `prometheus.yml` 파일을 생성하여 Prometheus 구성을 설정하십시오.
``` 
global:
  scrape_interval: 2s
  evaluation_interval: 2s
 
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']
  - job_name: 'loan-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080'] ## only for demo purposes don't use host.docker.internal in production
  - job_name: 'fraud-detection'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8081'] ## only for demo purposes don't use host.docker.internal in production
```

### TEMPO 설정

**@Observed**
> [!TIP]
> 특정 호출을 수동으로 추적하려면 `Observation API`와 class 또는 method에 `@Observed` annotation을 사용할 수 있습니다.

예: JDBC 리포지토리 추적/호출을 추적하는 데 사용됨
``` 
@Repository
@RequiredArgsConstructor
@Observed
public class LoanRepository {
 
    private final JdbcClient jdbcClient;
 
    .....
    .....
    .....
}
```

다음, **ObservedAspect** 유형의 Bean을 정의하십시오. `src/main/java/.../config`에 `ObservationConfig.`java` 클래스를 생성하십시오.
``` 
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.aop.ObservedAspect;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
 
@Configuration
public class ObservationConfig {
    @Bean
    ObservedAspect observedAspect(ObservationRegistry registry) {
        return new ObservedAspect(registry);
    }
```

다음으로, 각 서비스의 `application.properties`에서 추적 속성을 변경하십시오.
``` 
management.tracing.sampling.probability=1.0
```

1.0은 100%의 trace 정보를 전송한다는 의미이며, 기본값은 0.1 (10%)입니다.

`docker-compose.yaml`에 Tempo를 추가하십시오.
``` 
tempo:
  image: grafana/tempo:2.5.0
  command: [ "-config.file=/etc/tempo.yaml" ]
  volumes:
    - ./docker/tempo/tempo.yml:/etc/tempo.yaml:ro
    - ./docker/tempo/tempo-data:/tmp/tempo
  ports:
    - "3200:3200" #Tempo changed from "3110:3100" 
    - "9411:9411" # zipkin
```

마지막으로, `docker/tempo` 디렉토리에 `tempo.yaml` 파일을 생성하십시오.
```
server:
  http_listen_port: 3200
 
distributor:
  receivers:
    zipkin:
 
storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/blocks
```

### Grafana 설정
> [!CAUTION]
> Production 수준에서는 다음 설정을 사용하지 마십시오.

`docker-compose.yaml`에 다음 Grafana 구성을 설정하십시오.
``` 
grafana:
  image: grafana/grafana:11.1.3
  volumes:
    - ./docker/grafana:/etc/grafana/provisioning/datasources:ro
  environment:
    - GF_AUTH_ANONYMOUS_ENABLED=true
    - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    - GF_AUTH_DISABLE_LOGIN_FORM=true
  ports:
    - "3000:3000"
```
최신 버전을 확인하려면 이 라이브러리를 확인하십시오: [Grafana GitHub](https://github.com/grafana/grafana/releases)

다음으로, `docker` 디렉토리 안에 있는 `grafana` 디렉토리에 `datasource.yaml` 파일을 생성하십시오. 다음 구성을 추가하십시오.
``` 
apiVersion: 1
 
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    editable: false
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo
  - name: Tempo
    type: tempo
    access: proxy
    orgId: 1
    url: http://tempo:3200
    basicAuth: false
    isDefault: true
    version: 1
    editable: false
    apiVersion: 1
    uid: tempo
    jsonData:
      httpMethod: GET
      tracesToLogs:
        datasourceUid: 'loki'
      nodeGraph:
        enabled: true
  - name: Loki
    type: loki
    uid: loki
    access: proxy
    orgId: 1
    url: http://loki:3100
    basicAuth: false
    isDefault: false
    version: 1
    editable: false
    apiVersion: 1
    jsonData:
      derivedFields:
        -   datasourceUid: tempo
            matcherRegex: \[.+,(.+?),
            name: TraceID
            url: $${__value.raw}
```

## Grafana Stack 실행하기
[Docker Desktop](https://www.docker.com/products/docker-desktop/)을 다운로드하고, Docker를 열어 엔진이 실행 중인지 확인하십시오.

![docker_engine.png](images/docker_engine.png)

터미널에 다음 명령을 실행하십시오: ```docker compose up -d```

다음이 나타나야 합니다

![docker_terminal.png](images/docker_terminal.png)

모든 컨테이너가 Docker Desktop에서 실행 중인지 다시 확인하십시오.

Loan Service 애플리케이션을 실행하십시오.

```cd loan-service```

```mvn spring-boot:run```

Fraud Detection Service  애플리케이션을 실행하십시오.

```cd fraud-detection-service```

```mvn spring-boot:run```


## 서비스에 접근하기
1. Grafana: http://localhost:3000
2. Prometheus: http://localhost:9090
3. Tempo: http://localhost:3110
4. Loki: http://localhost:3100

## 대시보드 구축하기
Some basic queries to build quick and easy dashboards

### Logs
![Grafana_Loki - Copy.png](images/Grafana_Loki%20-%20Copy.png)

로그 보기
1. 쿼리에 다음 코드를 붙여 넣으십시오: ``` {application="insert_your_service_name"} |= `$Filter` ```
2. 적절한 log visualization를 선택하십시오

오류 필터링
1. 다음 코드를 붙여넣고 visualization를 선택하십시오: ``` {application="loan-service"} |= `ERROR` ```

Time Series graph
1. 오류 수를 시간에 따라 보려면 다음 코드를 붙여넣으십시오: ``` count_over_time({application="loan-service"} |= "ERROR" [1h])```

### Prometheus
![Grafana_Prometheus - Copy.png](images/Grafana_Prometheus%20-%20Copy.png)
1. 미리 만들어진 대시보드를 가져올 수 있는지 확인하는 것이 좋습니다.
2. 필요에 맞는 대시보드를 확인하려면 [Grafana Dashboard website](https://grafana.com/grafana/dashboards/)를 확인하십시오.

### Tempo
![Grafana_Tempo_2 - Copy.png](images/Grafana_Tempo_2%20-%20Copy.png)

Tracing 그래프를 만들려면

![grafana_variable.png](images/grafana_variable.png)
1. 대시보드 설정에서 variable를 생성하십시오.
2. Visualization를 추가하십시오
3. TraceQL에 `$TraceID`를 추가하십시오
4. **Traces** visualization를 선택하십시오

Trace Logs
1. TraceQL에  ```{}``` 를 입력하여 모든 trace을 보십시오.
2. 표/그래프를 선택하고 다양한 시각화 중에서 선택하십시오.


EOF