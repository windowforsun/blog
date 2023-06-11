--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Data Flow(SCDF) MySQL 연동"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'SCDF 의 데이터를 외부 저장소 MySQL 과 연동해 저장하고 관리해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Cloud Data Flow
    - SCDF
    - MySQL
toc: true
use_math: true
---  

## Spring Cloud Data Flow with MySQL
[SCDF Kubernetes 환경 구성]({{site.baseurl}}{% link _posts/spring/2023-05-11-spring-practice-spring-cloud-data-flow-installation.md %})
에서는 `SCDF` 를 `Kubernetes` 환경에 `kubectl` 를 사용해 구성하는 방법에 대해 알아보았다. 
`SCDF` 는 `Spring Batch` 와 동일하게 `Batch`, `Stream` 의 구성과 수행 결과 및 관련 메타데이터와 히스토리를 
저장소를 통해 `Persistence` 를 제공한다. 
앞선 예제에서는 이러한 데이터를 `In-Memory` 인 `H2` 에 저장하는 방식이였다.  

하지만 위와 같은 방식은 만약 `Kubernetes` 환경에서 노드 재시작이나, 재배치 등의 상황에서 
지금까지 구성한 정보의 지속성을 보장할 수 없다. 
지속성을 보장하는 방법은 안정적으로 데이터를 저장할 수 있는 외부 저장소(`MySQL`, `PostgreSQL`)와 
연동하는 것이다.  

우선 `SCDF` 의 데이터를 연동할 수 있는 외부 저장소의 종류는 [여기](https://github.com/spring-cloud/spring-cloud-dataflow/tree/2.10.x/spring-cloud-dataflow-server-core/src/main/resources/schemas)
를 참조하면 아래와 같다.  

- `db2`
- `mariadb`
- `mysql`
- `oracle`
- `postgresql`
- `sqlserver`

본 포스트는 위 저장소 리스트 중 `MySQL` 연동 방법에 대해 알아보기 위해, 
앞선 예제인 [SCDF Kubernetes 환경 구성]({{site.baseurl}}{% link _posts/spring/2023-05-25-spring-practice-spring-cloud-data-flow-develop-deploy-application.md %})
에서 일정 설정과 `MySQL` 구성내용만 추가하는 방식으로 진행한다.  

테스트 구성에 사용할 버전 정보는 아래와 같다.  

- `Kubernetes` : v1.20.15
- `SCDF Server` : 2.10.3
- `SCDF Skipper` : 2.9.3
- `MySQL` : 8

### MySQL 구성
`MySQL` 은 `MySQL 8` 을 사용하고 테스트를 위해서는 `Kubernetes` 환경에 올리는 방식으로 진행한다. 
`MySQL` 을 `Kubernetes` 에 구성할 수 있는 템플릿 내용은 아래와 같다.  

```yaml
# mysql-deploy.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  labels:
    app: mysql

data:
  mysql.cnf: |-
    [mysqld]
    server-id = 1
    log_bin = mysql-bin
    binlog_format = ROW
    log_slave_updates = ON
    binlog_row_image = FULL
    expire_logs_days = 0
  init.sql: |-
    create database scdf;
    create database skipper;

---
apiVersion: v1
kind: Secret
metadata:
  name: mysql
  labels:
    app: mysql
data:
  # root
  mysql-root-password: cm9vdA==
  mysql-root-username: cm9vdA==

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: mysql-root-password
                  name: mysql
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-conf
              mountPath: /etc/mysql/conf.d/mysql.cnf
              subPath: mysql.cnf
            - name: init-sql
              mountPath: /docker-entrypoint-initdb.d/
      volumes:
        - name: mysql-conf
          configMap:
            name: mysql-configmap
            items:
              - key: mysql.cnf
                path: mysql.cnf
        - name: init-sql
          configMap:
            name: mysql-configmap
            items:
              - key: init.sql
                path: init.sql

---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  # Kubernetes 외부에서 접근이 필요한 경우
#  type: LoadBalancer
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```  

파일을 구성하는 `Kubernetes` 의 오브젝트의 구성은 아래와 같다. 

- `ConfigMap` : `MySQL` 설정 파일및 초기화 `SQL` 스크립트
- `Secret` : `MySQL` 계정 정보를 다른 `Kubernetes` 에서도 접근할 수 있도록 등록
- `Deploymenet` : `MySQL` 컨테이너 구성 및 위 구성정보 연동
- `Service` : 다른 `Kubernetes` 리소스에서 `MySQL` 포트 접근이 가능하도록 구성

위 템플릿을 아래 명령으로 `Kubernetes Cluster` 에 적용한다.  

```bash
$ kubectl apply -f mysql-deploy.yaml
configmap/mysql-configmap created
secret/mysql created
deployment.apps/mysql created
service/mysql created

$ kubectl get all -l app=mysql
NAME                        READY   STATUS    RESTARTS   AGE
pod/mysql-b64b5b87d-rvzzs   1/1     Running   0          35s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/mysql   ClusterIP   10.105.45.156   <none>        3306/TCP   35s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql   1/1     1            1           35s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql-b64b5b87d   1         1         1       35s
```  

구성이 됐다면 `MySQL` 접속해서 생성된 데이터베이스를 확인하는 방식으로 정상 구성여부를 확인 한다.  

```bash
$ kubectl exec -it mysql-b64b5b87d-rvzzs -- mysql -uroot -proot

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| scdf               |
| skipper            |
| sys                |
+--------------------+
7 rows in set (0.01 sec)

mysql> use scdf;
Database changed
mysql> show tables;
Empty set (0.01 sec)
mysql> use skipper;
Database changed
mysql> show tables;
Empty set (0.00 sec)
```  

`scdf`, `skipper` 데이터베이스가 정상정으로 생성됐고, 
아직 `SCDF Server`, `SCDF Skipper` 구동하지 않았기 때문에 해당 데이터베이스의 
테이블은 비어있는 상태임을 확인 할 수 있다.  

### SCDF Server, Skipper 구성
`SCDF Server 2.10.3` 와 `SCDF Skipper 2.9.3` 의 설정 정보만 잘 생성해 `MySQL` 과 연동하면 되는데, 
[여기](https://github.com/spring-cloud/spring-cloud-dataflow/blob/98ec6dc7ac0f693e49206fb904ced52cc6a35b6b/spring-cloud-dataflow-server-core/pom.xml#L155)
를 보면 `SCDF` 에서는 별도로 `MySQL` 드라이버를 사용하지 않고 `MariaDB` 드라이버만 사용할 수 있다.  

그리고 현재 사용하는 `MaraiDB` 라이브러리의 버전이 `3.x` 이므로 아래 내용을 참고해서 설정이 필요하다. [참고](https://mariadb.com/kb/en/about-mariadb-connector-j/#jdbcmysql-scheme-compatibility)

```
Connector/J still allows jdbc:mysql: as the protocol in connection strings when the permitMysqlScheme option is set. For example:

jdbc:mysql://HOST/DATABASE?permitMysqlScheme

(2.x version did permit connection URLs beginning with both jdbc:mariadb and jdbc:mysql)
```  

이전 포스트의 [SCDF](https://windowforsun.github.io/blog/spring/spring-practice-spring-cloud-data-flow-installation/#scdf-skipper-and-server)
구성 부분에 보면 `skipper/skipper-config-kafka.yaml` 과 `server/server-config.yaml` 파일이 있다. 
`Server` 와 `Skipper` 가 구동할 떄 참조하는 `Spring Boot Properties` 는 위 파일에 작성하게 되는데, 
`MySQL` 연동을 위해 두 파일을 각각 아래와 같이 `datasource` 내용을 추가한다.  

```yaml
# skipper/skipper-config-kafka.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: skipper
  labels:
    app: skipper
data:
  application.yaml: |-
    spring:
      cloud:
        skipper:
          server:
            platform:
              kubernetes:
                accounts:
                  default:
                    environmentVariables: 'SPRING_CLOUD_STREAM_KAFKA_BINDER_BROKERS=kafka-service:9092,SPRING_CLOUD_STREAM_KAFKA_BINDER_ZK_NODES=${KAFKA_ZK_SERVICE_HOST}:${KAFKA_ZK_SERVICE_PORT}'
                    limits:
                      memory: 1g
                      cpu: 1.0                    
                    readinessProbeDelay: 1
                    readinessProbeTimeout: 5
                    livenessProbeDelay: 1
                    livenessProbeTimeout: 2
                    startupProbeDelay: 20
                    startupProbeTimeout: 5
                    startupProbeFailure: 50
      datasource:
        url: jdbc:mysql://${MYSQL_SERVICE_HOST}:${MYSQL_SERVICE_PORT}/skipper?permitMysqlScheme
        username: root
        password: ${mysql-root-password}
        driverClassName: org.mariadb.jdbc.Driver
        testOnBorrow: true
        validationQuery: "SELECT 1"
```  

```yaml
# server/server-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: scdf-server
  labels:
    app: scdf-server
data:
  application.yaml: |-
    spring:
      cloud:
        dataflow:
          task:
            platform:
              kubernetes:
                accounts:                  
                  default:
                    limits:
                      memory: 1000Mi
      datasource:
        url: jdbc:mysql://${MYSQL_SERVICE_HOST}:${MYSQL_SERVICE_PORT}/scdf?permitMysqlScheme
        username: root
        password: ${mysql-root-password}
        driverClassName: org.mariadb.jdbc.Driver
        testOnBorrow: true
        validationQuery: "SELECT 1"
```  

그리고 `skipper/skipper-deployment.yaml` 과 `server/server-deployment.yaml` 
파일에 `MySQL` 의 `Secret` 을 연동하는 부분과 `initContainer` 를 통해 `MySQL` 연결여부 확인 과정을 추가한다.  

```yaml
# skipper/skipper-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: skipper
  labels:
    app: skipper
spec:
  selector:
    matchLabels:
      app: skipper
  replicas: 1
  template:
    metadata:
      labels:
        app: skipper
    spec:
      containers:
      - name: skipper
        image: springcloud/spring-cloud-skipper-server:2.9.3
        imagePullPolicy: Always
        volumeMounts:
          - name: config
            mountPath: /workspace/config
            readOnly: true
          # mysql secret 마운트
          - name: database
            mountPath: /etc/secrets/database
            readOnly: true
        ports:
        - containerPort: 7577
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 7577
          initialDelaySeconds: 1
          periodSeconds: 60
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /actuator/info
            port: 7577
          initialDelaySeconds: 1
          periodSeconds: 10
          timeoutSeconds: 2
        startupProbe:
          tcpSocket:
            port: 7577
          initialDelaySeconds: 15
          periodSeconds: 3
          failureThreshold: 120
          timeoutSeconds: 3
        resources:
          requests:
            cpu: 1.0
            memory: 1024Mi
        env:
        - name: LANG
          value: 'en_US.utf8'
        - name: LC_ALL
          value: 'en_US.utf8'
        - name: JDK_JAVA_OPTIONS
          value: '-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8'
        - name: SERVER_PORT
          value: '7577'
        - name: SPRING_CLOUD_CONFIG_ENABLED
          value: 'false'
        - name: SPRING_CLOUD_KUBERNETES_CONFIG_ENABLE_API
          value: 'false'
        - name: SPRING_CLOUD_KUBERNETES_SECRETS_ENABLE_API
          value: 'false'
        - name: SPRING_CLOUD_KUBERNETES_SECRETS_PATHS
          value: /etc/secrets
      # mysql 접근 여부 확인
      initContainers:
      - name: init-mysql-wait
        image: busybox:1
        imagePullPolicy: IfNotPresent
        command: ['sh', '-c', 'until nc -w3 -z mysql 3306; do echo waiting for mysql; sleep 3; done;']
      - name: init-mysql-database
        image: mysql:8
        env:
        - name: MYSQL_PWD
          valueFrom:
            secretKeyRef:
              name: mysql
              key: mysql-root-password
        command: ['sh', '-c', 'mysql -h mysql -u root --password=$MYSQL_PWD -e "CREATE DATABASE IF NOT EXISTS skipper;"']
      serviceAccountName: scdf-sa
      volumes:
        - name: config
          configMap:
            name: skipper
            items:
            - key: application.yaml
              path: application.yaml
        # mysql secret 마운트
        - name: database
          secret:
            secretName: mysql
```  

```yaml
# server/server-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: scdf-server
  labels:
    app: scdf-server
spec:
  selector:
    matchLabels:
      app: scdf-server
  replicas: 1
  template:
    metadata:
      labels:
        app: scdf-server
    spec:
      containers:
        - name: scdf-server
          image: springcloud/spring-cloud-dataflow-server:2.10.3
          imagePullPolicy: Always
          volumeMounts:
            - name: config
              mountPath: /workspace/config
              readOnly: true
            # mysql secret 마운트
            - name: database
              mountPath: /etc/secrets/database
              readOnly: true
          ports:
            - containerPort: 9393
          livenessProbe:
            httpGet:
              path: /management/health
              port: 9393
            initialDelaySeconds: 45
          readinessProbe:
            httpGet:
              path: /management/info
              port: 9393
            initialDelaySeconds: 45
          startupProbe:
            tcpSocket:
              port: 9393
            initialDelaySeconds: 15
            periodSeconds: 3
            failureThreshold: 120
            timeoutSeconds: 3
          resources:
            requests:
              cpu: 1.0
              memory: 2048Mi
          env:
            - name: LANG
              value: 'en_US.utf8'
            - name: LC_ALL
              value: 'en_US.utf8'
            - name: JDK_JAVA_OPTIONS
              value: '-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8'
            - name: KUBERNETES_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: "metadata.namespace"
            - name: SERVER_PORT
              value: '9393'
            - name: SPRING_CLOUD_CONFIG_ENABLED
              value: 'false'
            - name: SPRING_CLOUD_DATAFLOW_FEATURES_ANALYTICS_ENABLED
              value: 'true'
            - name: SPRING_CLOUD_DATAFLOW_FEATURES_SCHEDULES_ENABLED
              value: 'true'
            - name: SPRING_CLOUD_DATAFLOW_TASK_COMPOSEDTASKRUNNER_URI
              value: 'docker://springcloud/spring-cloud-dataflow-composed-task-runner:2.9.1'
            - name: SPRING_CLOUD_KUBERNETES_CONFIG_ENABLE_API
              value: 'false'
            - name: SPRING_CLOUD_KUBERNETES_SECRETS_ENABLE_API
              value: 'false'
            - name: SPRING_CLOUD_KUBERNETES_SECRETS_PATHS
              value: /etc/secrets
            - name: SPRING_CLOUD_DATAFLOW_SERVER_URI
              value: 'http://${SCDF_SERVER_SERVICE_HOST}:${SCDF_SERVER_SERVICE_PORT}'
              # Provide the Skipper service location
            - name: SPRING_CLOUD_SKIPPER_CLIENT_SERVER_URI
              value: 'http://${SKIPPER_SERVICE_HOST}:${SKIPPER_SERVICE_PORT}/api'
              # Add Maven repo for metadata artifact resolution for all stream apps
            - name: SPRING_APPLICATION_JSON
              value: "{ \"maven\": { \"local-repository\": null, \"remote-repositories\": { \"repo1\": { \"url\": \"https://repo.spring.io/libs-snapshot\"} } } }"
      # mysql 접근 여부 확인
      initContainers:
        - name: init-mysql-wait
          image: busybox:1
          imagePullPolicy: IfNotPresent
          command: ['sh', '-c', 'until nc -w3 -z mysql 3306; do echo waiting for mysql; sleep 3; done;']
      serviceAccountName: scdf-sa
      volumes:
        - name: config
          configMap:
            name: scdf-server
            items:
              - key: application.yaml
                path: application.yaml
        # mysql secret 마운트
        - name: database
          secret:
            secretName: mysql
```  

이제 `kubectl` 명령으로 `Kubernetes` 환경에 반영한다.  

```bash
$ kubectl apply -f skipper 
configmap/skipper created
deployment.apps/skipper created
service/skipper created
$ kubectl apply -f server 
configmap/scdf-server created
deployment.apps/scdf-server created
rolebinding.rbac.authorization.k8s.io/scdf-rb created
role.rbac.authorization.k8s.io/scdf-role created
service/scdf-server created
serviceaccount/scdf-sa created

$ kubectl get all -l app=scdf-server
NAME                               READY   STATUS    RESTARTS   AGE
pod/scdf-server-79584d68c4-zqmpm   1/1     Running   0          115s

NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/scdf-server   LoadBalancer   10.98.163.214   <pending>     80:30571/TCP   115s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scdf-server   1/1     1            1           115s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/scdf-server-79584d68c4   1         1         1       115s

$ kubectl get all -l app=skipper    
NAME                           READY   STATUS    RESTARTS   AGE
pod/skipper-685b5959d9-b5fx2   0/1     Running   0          2m4s

NAME              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/skipper   LoadBalancer   10.109.199.20   <pending>     80:31477/TCP   2m11s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/skipper   0/1     1            0           2m11s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/skipper-685b5959d9   1         1         0       2m11s
```  

`Server`, `Skipper` 의 `Pod` 상태가 `Running` 인것을 확인 후 로그를 보면 아래와 같이 
`MySQL` 에 정상적으로 접근해 스키마를 생성하고 애플리케이션 또한 에러없이 정상인 것을 확인 할 수 있다.  

```bash
$ kubectl logs -f skipper-685b5959d9-b5fx2
Setting Active Processor Count to 8
Calculating JVM memory based on 11550872K available memory
For more information on this calculation, see https://paketo.io/docs/reference/java-reference/#memory-calculator
Calculated JVM Memory Configuration: -XX:MaxDirectMemorySize=10M -Xmx10843083K -XX:MaxMetaspaceSize=195788K -XX:ReservedCodeCacheSize=240M -Xss1M (Total Memory: 11550872K, Thread Count: 250, Loaded Class Count: 32153, Headroom: 0%)
Enabling Java Native Memory Tracking
Adding 124 container CA certificates to JVM truststore
Spring Cloud Bindings Enabled

.. 생략 ..


2023-06-04 08:49:40.994  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2023-06-04 08:49:42.206  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2023-06-04 08:49:42.298  INFO 1 --- [           main] o.s.c.d.c.flyway.DatabaseDriverUtils     : Using MariaDB driver against MySQL database - will use MySQL
2023-06-04 08:49:42.298  INFO 1 --- [           main] d.m.SkipperFlywayConfigurationCustomizer : Adding vendor specific Flyway callback for MYSQL
2023-06-04 08:49:43.227  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    : Flyway Community Edition 8.5.13 by Redgate
2023-06-04 08:49:43.229  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    : See what's new here: https://flywaydb.org/documentation/learnmore/releaseNotes#8.5.13
2023-06-04 08:49:43.230  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    :
2023-06-04 08:49:43.469  INFO 1 --- [           main] o.f.c.i.database.base.BaseDatabaseType   : Database: jdbc:mariadb://10.105.45.156/skipper (MySQL 8.0)
2023-06-04 08:49:43.616  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    : Flyway Community Edition 8.5.13 by Redgate
2023-06-04 08:49:43.616  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    : See what's new here: https://flywaydb.org/documentation/learnmore/releaseNotes#8.5.13
2023-06-04 08:49:43.617  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    :
2023-06-04 08:49:43.676  INFO 1 --- [           main] o.f.core.internal.command.DbValidate     : Successfully validated 1 migration (execution time 00:00.014s)
2023-06-04 08:49:43.739  INFO 1 --- [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table `skipper`.`flyway_schema_history` ...
2023-06-04 08:49:43.993  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Current version of schema `skipper`: << Empty Schema >>
2023-06-04 08:49:44.011  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema `skipper` to version "1 - Initial Setup"
2023-06-04 08:49:46.349  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 1 migration to schema `skipper`, now at version v1 (execution time 00:02.367s)

.. 생략 ..
```  

```
$ kubectl logs -f scdf-server-79584d68c4-zqmpm
Setting Active Processor Count to 8
Calculating JVM memory based on 11950880K available memory
For more information on this calculation, see https://paketo.io/docs/reference/java-reference/#memory-calculator
Calculated JVM Memory Configuration: -XX:MaxDirectMemorySize=10M -Xmx11238679K -XX:MaxMetaspaceSize=200200K -XX:ReservedCodeCacheSize=240M -Xss1M (Total Memory: 11950880K, Thread Count: 250, Loaded Class Count: 32932, Headroom: 0%)
Enabling Java Native Memory Tracking
Adding 124 container CA certificates to JVM truststore
Spring Cloud Bindings Enabled
NOTE: Picked up JDK_JAVA_OPTIONS: -Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8
Picked up JAVA_TOOL_OPTIONS: -Djava.security.properties=/layers/paketo-buildpacks_bellsoft-liberica/java-security-properties/java-security.properties -XX:+ExitOnOutOfMemoryError -XX:ActiveProcessorCount=8 -XX:MaxDirectMemorySize=10M -Xmx11238679K -XX:MaxMetaspaceSize=200200K -XX:ReservedCodeCacheSize=240M -Xss1M -XX:+UnlockDiagnosticVMOptions -XX:NativeMemoryTracking=summary -XX:+PrintNMTStatistics -Dorg.springframework.cloud.bindings.boot.enable=true
2023-06-04 08:48:58.004  INFO 1 --- [kground-preinit] o.h.validator.internal.util.Version      : HV000001: Hibernate Validator 6.2.5.Final
  ____                              ____ _                __
 / ___| _ __  _ __(_)_ __   __ _   / ___| | ___  _   _  __| |
 \___ \| '_ \| '__| | '_ \ / _` | | |   | |/ _ \| | | |/ _` |
  ___) | |_) | |  | | | | | (_| | | |___| | (_) | |_| | (_| |
 |____/| .__/|_|  |_|_| |_|\__, |  \____|_|\___/ \__,_|\__,_|
  ____ |_|    _          __|___/                 __________
 |  _ \  __ _| |_ __ _  |  ___| | _____      __  \ \ \ \ \ \
 | | | |/ _` | __/ _` | | |_  | |/ _ \ \ /\ / /   \ \ \ \ \ \
 | |_| | (_| | || (_| | |  _| | | (_) \ V  V /    / / / / / /
 |____/ \__,_|\__\__,_| |_|   |_|\___/ \_/\_/    /_/_/_/_/_/

Spring Cloud Data Flow Server  (v2.10.3)

.. 생략 ..

2023-06-04 08:49:28.321  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2023-06-04 08:49:30.101  INFO 1 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2023-06-04 08:49:30.180  INFO 1 --- [           main] o.s.c.d.c.flyway.DatabaseDriverUtils     : Using MariaDB driver against MySQL database - will use MySQL
2023-06-04 08:49:30.181  INFO 1 --- [           main] .m.DataFlowFlywayConfigurationCustomizer : Adding vendor specific Flyway callback for MYSQL
2023-06-04 08:49:31.122  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    : Flyway Community Edition 8.5.13 by Redgate
2023-06-04 08:49:31.126  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    : See what's new here: https://flywaydb.org/documentation/learnmore/releaseNotes#8.5.13
2023-06-04 08:49:31.128  INFO 1 --- [           main] o.f.c.internal.license.VersionPrinter    :
2023-06-04 08:49:31.389  INFO 1 --- [           main] o.f.c.i.database.base.BaseDatabaseType   : Database: jdbc:mariadb://10.105.45.156/scdf (MySQL 8.0)
2023-06-04 08:49:31.574  INFO 1 --- [           main] o.f.core.internal.command.DbValidate     : Successfully validated 5 migrations (execution time 00:00.066s)
2023-06-04 08:49:31.625  INFO 1 --- [           main] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table `scdf`.`flyway_schema_history_dataflow` ...
2023-06-04 08:49:32.052  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Current version of schema `scdf`: << Empty Schema >>
2023-06-04 08:49:32.080  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema `scdf` to version "1 - Initial Setup"
2023-06-04 08:49:33.364  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema `scdf` to version "2 - Add Descriptions And OriginalDefinition"
2023-06-04 08:49:33.509  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema `scdf` to version "3 - Add Platform To AuditRecords"
2023-06-04 08:49:33.603  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema `scdf` to version "4 - Add Step Name Indexes"
2023-06-04 08:49:33.664  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Migrating schema `scdf` to version "5 - Add Task Execution Params Indexes"
2023-06-04 08:49:33.748  INFO 1 --- [           main] o.f.core.internal.command.DbMigrate      : Successfully applied 5 migrations to schema `scdf`, now at version v5 (execution time 00:01.718s)
2023-06-04 08:49:34.390  INFO 1 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2023-06-04 08:49:34.890  INFO 1 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 5.6.15.Final
2023-06-04 08:49:36.205  INFO 1 --- [           main] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.1.2.Final}
2023-06-04 08:49:36.941  INFO 1 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.MariaDB53Dialect
2023-06-04 08:49:41.690  INFO 1 --- [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2023-06-04 08:49:41.762  INFO 1 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2023-06-04 08:49:52.095  INFO 1 --- [           main] o.s.b.c.r.s.JobRepositoryFactoryBean     : No database type set, using meta data indicating: MYSQL
2023-06-04 08:49:52.268  INFO 1 --- [           main] o.s.c.d.s.b.SimpleJobServiceFactoryBean  : No database type set, using meta data indicating: MYSQL

.. 생략 ..
```  

이제 마지막으로 `MySQL` 에 다시 접속해 실제로 `scdf`, `skipper` 데이터베이스에 
필요한 테이블들이 모두 정상적으로 생성됐는지 확인 한다.  

```bash
$ kubectl exec -it mysql-b64b5b87d-rvzzs -- mysql -uroot -proot

mysql> use scdf;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+--------------------------------+
| Tables_in_scdf                 |
+--------------------------------+
| BATCH_JOB_EXECUTION            |
| BATCH_JOB_EXECUTION_CONTEXT    |
| BATCH_JOB_EXECUTION_PARAMS     |
| BATCH_JOB_EXECUTION_SEQ        |
| BATCH_JOB_INSTANCE             |
| BATCH_JOB_SEQ                  |
| BATCH_STEP_EXECUTION           |
| BATCH_STEP_EXECUTION_CONTEXT   |
| BATCH_STEP_EXECUTION_SEQ       |
| TASK_EXECUTION                 |
| TASK_EXECUTION_PARAMS          |
| TASK_LOCK                      |
| TASK_SEQ                       |
| TASK_TASK_BATCH                |
| app_registration               |
| audit_records                  |
| flyway_schema_history_dataflow |
| hibernate_sequence             |
| stream_definitions             |
| task_definitions               |
| task_deployment                |
| task_execution_metadata        |
| task_execution_metadata_seq    |
+--------------------------------+
23 rows in set (0.00 sec)

mysql> use skipper;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------------------+
| Tables_in_skipper         |
+---------------------------+
| action                    |
| deferred_events           |
| flyway_schema_history     |
| guard                     |
| hibernate_sequence        |
| skipper_app_deployer_data |
| skipper_info              |
| skipper_manifest          |
| skipper_package_file      |
| skipper_package_metadata  |
| skipper_release           |
| skipper_repository        |
| skipper_status            |
| state                     |
| state_entry_actions       |
| state_exit_actions        |
| state_machine             |
| state_state_actions       |
| transition                |
| transition_actions        |
+---------------------------+
20 rows in set (0.00 sec)
```  

위 구성까지 마쳤다면, 
`SCDF` 를 사용해서 `Bath`, `Stream` 을 구성하거나 애플리케이션을 관리할때 필요한 정보들이 
연동된 `MySQL` 에 저장되고 관리되기 때문에 데이터의 지속성을 보장 할 수 있다.  


---  
## Reference
[Spring Cloud Data Flow Deploying with kubectl](https://dataflow.spring.io/docs/installation/kubernetes/kubectl/)
[Stream Processing using Spring Cloud Data Flow](https://dataflow.spring.io/docs/stream-developer-guides/streams/data-flow-stream/)
