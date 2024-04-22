# [🔥 k8s_kafka_guide]

# 1. Strimzi Operator란?

**Strimzi operator는 쿠버네티스 환경에서 카프카 설치와 운영을 단순화 했습니다.** 

쿠버네티스 crd를 이용하여 주키퍼, 카프카 클러스터를 설치할 수 있고 토픽 등도 crd로 관리합니다.

## 1-1. 아키텍처

![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/63e8348e-767d-4a0f-a7b9-930965d3fd2b)


- cluster operator: kafaka cluster, zookeeper cluster 등 컴퍼넌트를 배포하고 관리
- entity operator: user operator와 topic operator를 관리
    - topic operator: topic생성, 삭제 등 topic관리
    - user operator: kafka 유저 관리
- zookeeper cluster
    - 카프카의 메타데이터 관리 및 브로커의 정상 상태 점검
- kafka cluster
    - 카프카 클러스터(여러 대 브로커 구성) 구성

# 2. Kafka 설치

OKD 환경에서 설치하였으며, Worker Node 3개를 사용하였습니다.

- `oc login <Cluster API address> -u <id> -p <pw> --insecure-skip-tls-verify`

## 2-1. Cluster Operator 설치

**카프카, 주키퍼 설치는 cluster operator를 설치하는 것으로 시작**합니다.

![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/b8e7013d-0606-4a40-82be-880c2f2d39d6)


---

- 1) 카프카가 설치될 namespace를 생성합니다.
    - `oc new-project kafka-system --display-name 'kafka-system'`
- 2) Kafka Cluster, Zookeeper을 Node 1-3에 각 하나씩 설치할 것이므로 Node-Selector 설정을 해줍니다.
    - `kubectl get node`
        
        ```java
        root@edu3:~/icis-tr/kafka/strimzi# k get node
        NAME                     STATUS   ROLES    AGE   VERSION
        edu-cloud.worker01       Ready    worker   27d   v1.23.5+9ce5071
        edu-cloud.worker02       Ready    worker   27d   v1.23.5+9ce5071
        edu-cloud.worker03       Ready    worker   27d   v1.23.5+9ce5071
        ```
        
    - `oc label nodes <노드1~3이름> kafka-system=true --overwrite`
        - ex) `oc label nodes edu-cloud.worker01 kafka-system=true --overwrite`
    - `kubectl edit ns kafka-system`에서 annotation을 수정합니다.
        
        ```java
        metadata:
          annotations:
            openshift.io/description: ""
            openshift.io/display-name: argocd
            openshift.io/node-selector: kafka-system=true #추가
            openshift.io/requester: root
            openshift.io/sa.scc.mcs: s0:c26,c20
            openshift.io/sa.scc.supplemental-groups: 1000690000/10000
            openshift.io/sa.scc.uid-range: 1000690000/10000
        ```
        
- 3) Strimzi Kafka Operator을 다운 받고 설치합니다.
    - `wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.40.0/strimzi-0.40.0.tar.gz`
    - `tar -zxvf strimzi-0.40.0.tar.gz` > `cd strimzi-0.40.0`
    - namespace를 kafka-system으로 변경합니다.
        - `sed -i 's/namespace: .*/namespace: kafka-system/' install/cluster-operator/**RoleBinding**.yaml`
    - Deployment 파일을 다음과 같이 수정합니다 : `vi install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml`
        
        ```yaml
         41           env:
         42             - name: STRIMZI_NAMESPACE
         43               value: kafka-system
         44               #valueFrom:
         45               #  fieldRef:
         46               #    fieldPath: metadata.namespace
        ```
        
    - `kubectl apply -f install/cluster-operator/`
- 4) Operator가 정상 작동 하는지 확인합니다.
    
    ```java
    root@edu3:~/icis-tr/kafka/strimzi# k get pod
    NAME                                             READY   STATUS    RESTARTS        AGE
    ...
    strimzi-cluster-operator-6b8688cf68-9hkbs        1/1     Running   2 (2d21h ago)   3d11h
    ```
    

## 2-2. (선택) PV, PVC 생성

strimzi의 pvc 생성 규칙에 맞춰서 다음과 같이 pv, pvc를 생성합니다.

**cluster, zookeeper를 각 3개 씩** 설치할 것이므로 아래와 같이 pv, pvc를 구성합니다. 

```java
root@edu3:~/icis-tr/kafka# k get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                                                        STORAGECLASS   REASON   AGE
kafka-cluster-pv-0                         5Gi        RWX            Retain           Bound         kafka-system/data-0-kafka-cluster-kafka-0                                            13d
kafka-cluster-pv-1                         5Gi        RWX            Retain           Bound         kafka-system/data-0-kafka-cluster-kafka-1                                            13d
kafka-cluster-pv-2                         5Gi        RWX            Retain           Bound         kafka-system/data-0-kafka-cluster-kafka-2                                            13d
kafka-pv-0                                 5Gi        RWX            Retain           Bound         kafka-system/data-kafka-cluster-zookeeper-0                                          20d
kafka-pv-1                                 5Gi        RWX            Retain           Bound         kafka-system/data-kafka-cluster-zookeeper-1                                          20d
kafka-pv-2                                 5Gi        RWX            Retain           Bound         kafka-system/data-kafka-cluster-zookeeper-2                                          20d
```

```yaml
root@edu3:~/icis-tr/kafka# k get pvc
NAME                             STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-0-kafka-cluster-kafka-0     Bound    kafka-cluster-pv-0   5Gi        RWX                           120m
data-0-kafka-cluster-kafka-1     Bound    kafka-cluster-pv-1   5Gi        RWX                           119m
data-0-kafka-cluster-kafka-2     Bound    kafka-cluster-pv-2   5Gi        RWX                           118m
data-kafka-cluster-zookeeper-0   Bound    kafka-pv-0           5Gi        RWX                           160m
data-kafka-cluster-zookeeper-1   Bound    kafka-pv-1           5Gi        RWX                           159m
data-kafka-cluster-zookeeper-2   Bound    kafka-pv-2           5Gi        RWX                           159m
```

---

- (예시)`vi kafka-pv.yaml`
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
       # 위의 pv명 참고해서 6개 생성할 것
      name: kafka-cluster-pv-0
    spec:
      accessModes:
      - ReadWriteMany
      capacity:
        storage: 5Gi
      nfs:
        path: /<nfs경로>/data-0-kafka-cluster-kafka-0
        server: <nfs server ip>
      persistentVolumeReclaimPolicy: Retain
    ```
    
- (예시)`vi kafka-pvc.yaml`
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      # pvc 이름 바꾸지 말고 그대로
      # data-0-kafka-cluster-kafka-1, data-0-kafka-cluster-kafka-2 등 위의 pvc정보 참고해서 만들 것!!!
      name: data-0-kafka-cluster-kafka-0 
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 5Gi
      volumeName: kafka-cluster-pv-0 #pv이름
    ```
    

## 2-3. Cluster 생성

**Strimzi CRD로 주키퍼 클러스터와 카프카 클러스터를 설치합니다.** 

Kafka CRD를 이용하여 쉽게 설치할 수 있습니다. (2-1에서 CRD 설치 했었음)

cluster operator가 **zookeeper 클러스터와 kafka 클러스터를 관리하는** 것을 알 수 있습니다.

![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/689dc382-d006-460c-8179-2fe67570efe1)


---

- 1) kafka cluster을 생성하는 yaml파일을 작성합니다. 파일 정보는 다음과 같습니다.
    - 1-1) **`runAsUser`을 변경합니다.**
        - runAsUser은 보안 컨텍스트에 허용되는 값을 지시하는 값입니다. 
        `oc describe project <ns>` 를 통해 확인할 수 있습니다.
            
            ```java
            root@edu3:~/icis-tr/kafka/strimzi# oc describe project kafka-system
            Name:			kafka-system
            Created:		3 weeks ago
            Labels:			kubernetes.io/metadata.name=kafka-system
            Annotations:		openshift.io/description=
            			openshift.io/display-name=kafka-system
            			openshift.io/node-selector=kafka-system=true
            			openshift.io/requester=root
            			openshift.io/sa.scc.mcs=s0:c27,c19
            			openshift.io/sa.scc.supplemental-groups=1000740000/10000
            ```
            위의 예시에서는 upplemental-groups가 `1000740000/10000` 이므로 `[10740000,10749999]` 사이의 값을 설정해주어야 합니다.
            
        - 다음의 값을 반영한 `cluster.yaml` 파일입니다.
            
            ```java
             47         securityContext:
             48           runAsUser: 1000740001 #수정
             49           fsGroup: 1000
             ...
              73       pod:
             74         securityContext:
             75           runAsUser: 1000740001 #수정
             76           fsGroup: 1000
            ```
            
        - 만약 runAsUser을 알맞은 값으로 설정하지 않는다면 `kubectl get events` 에서 다음과 같은 오류가 발생할 수 있습니다.
            
            ```java
             Warning  FailedCreate      19s (x14 over 40s)  statefulset-controller  create Pod test-0 in StatefulSet test failed error: pods "test-0" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 65000: must be in the ranges: [1000740000, 100749999]]
            ```
            
    - 1-2)`spec.kafka.storage` 와 `spec.zookeeper.storage`에 2-2에서 생성한 pv, pvc를 사용하도록 설정했습니다.
        
        ```java
         35     storage:
         36       type: jbod
         37       volumes:
         38         - id: 0
         39           type: persistent-claim
         40           size: 5Gi
         41           deleteClaim: true
         ...
         68     storage:
         69       type: persistent-claim
         70       size: 5Gi
         71       deleteClaim: false
        ```
        
        - 만약 **임시 스토리지를 사용할 것이라면, 아래 링크를 참고하여 `spec.kafka.storage` 와 `spec.zookeeper.storage`를 ephemeral로 변경**합니다.
        - https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/kafka/kafka-ephemeral.yaml
    - 1-3) `route` listener
        
        ```java
         11     listeners:
        ...
         24         type: route
         25         tls: true
         26         authentication:
         27           type: tls
        ```
        
        - strimzi 일정 버전 이상부터 **listener type이 route라면 tls가 무조건 true여야** 합니다.
        - 따라서 외부에서 kafka bootstrap server에 접속할 때엔 인증서가 필요합니다. (2-4에서 생성 예정)
    - 1-4) 이전에 **broker, zookeeper를 각 3개씩** 설치한다고 했습니다.
    `podAntiAffinity` 를 통해 Node1~3에 각각 하나씩 배치되도록 설정하였습니다.
        
        ```java
         50         affinity:
         51           podAntiAffinity:
         52             requiredDuringSchedulingIgnoredDuringExecution:
         53               - labelSelector:
         54                   matchExpressions:
         55                     - key: app.kubernetes.io/name
         56                       operator: In
         57                       values:
         58                         - kafka
         59                 topologyKey: "kubernetes.io/hostname"
        ```
        
        ```java
        root@edu3:~/icis-tr/kafka# k get pod -o wide
        NAME                                    NODE                 NOMINATED NODE   READINESS GATES
        kafka-cluster-kafka-0                   edu-cloud.worker02   <none>           <none>
        kafka-cluster-kafka-1                   edu-cloud.worker03   <none>           <none>
        kafka-cluster-kafka-2                   edu-cloud.worker01   <none>           <none>
        kafka-cluster-zookeeper-0               edu-cloud.worker02   <none>           <none>
        kafka-cluster-zookeeper-1               edu-cloud.worker03   <none>           <none>
        kafka-cluster-zookeeper-2               edu-cloud.worker01   <none>           <none>
        ```
        
    - 1-5) monitoring을 위해 prometheus Exporter 을 추가했습니다.
        
        ```java
          ...
            metricsConfig:
              type: jmxPrometheusExporter
              valueFrom:
                configMapKeyRef:
                  name: kafka-metrics
                  key: kafka-metrics-config.yml
          ...
          kafkaExporter:
            topicRegex: ".*"
            groupRegex: ".*"
        ```
        

위의 설정을 모두 적용한 yaml파일은 다음과 같습니다.

- 2) `vi cluster.yaml`
    
    ```yaml
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    metadata:
      name: kafka-cluster
    spec:
      kafka:
        version: 3.7.0
        replicas: 3
        listeners:
          - name: plain
            port: 9092
            type: internal
            tls: false        
          - name: tls
            port: 9093
            type: internal
            tls: true
          - name: route
            port: 9094
            type: route
            tls: true
            authentication:
              type: tls
        config:
          offsets.topic.replication.factor: 3
          transaction.state.log.replication.factor: 3
          transaction.state.log.min.isr: 2
          default.replication.factor: 3
          min.insync.replicas: 2
          inter.broker.protocol.version: "3.7"
        # pvc info
        storage:
          type: jbod
          volumes:
            - id: 0
              type: persistent-claim
              size: 5Gi
              deleteClaim: true
        template:
          pod:
            metadata:
              labels:
                app: kafka-cluster
            securityContext:
              runAsUser: 1000740001 
              fsGroup: 1000
            # 3개의 cluster가 각각 workernode1,2,3에 배치되도록 한다.
            affinity:
              podAntiAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  - labelSelector:
                      matchExpressions:
                        - key: app.kubernetes.io/name
                          operator: In
                          values:
                            - kafka
                    topologyKey: "kubernetes.io/hostname"
        # prometheus monitoring을 위한 exporter
        metricsConfig:
          type: jmxPrometheusExporter
          valueFrom:
            configMapKeyRef:
              name: kafka-metrics
              key: kafka-metrics-config.yml
      zookeeper:
        replicas: 3
        storage:
          type: persistent-claim
          size: 5Gi
          deleteClaim: false
        template:
          pod:
            securityContext:
              runAsUser: 1000740001
              fsGroup: 1000
            affinity:
              podAntiAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  - labelSelector:
                      matchExpressions:
                        - key: app.kubernetes.io/name
                          operator: In
                          values:
                            - zookeeper
                    topologyKey: "kubernetes.io/hostname"
        metricsConfig:
          type: jmxPrometheusExporter
          valueFrom:
            configMapKeyRef:
              name: kafka-metrics
              key: zookeeper-metrics-config.yml
      entityOperator:
        topicOperator: {}
        userOperator: {}
      kafkaExporter:
        topicRegex: ".*"
        groupRegex: ".*"
    ---
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: kafka-metrics
      labels:
        app: strimzi
    data:
      kafka-metrics-config.yml: |
        # See https://github.com/prometheus/jmx_exporter for more info about JMX Prometheus Exporter metrics
        lowercaseOutputName: true
        rules:
        # Special cases and very specific rules
        - pattern: kafka.server<type=(.+), name=(.+), clientId=(.+), topic=(.+), partition=(.*)><>Value
          name: kafka_server_$1_$2
          type: GAUGE
          labels:
           clientId: "$3"
           topic: "$4"
           partition: "$5"
        - pattern: kafka.server<type=(.+), name=(.+), clientId=(.+), brokerHost=(.+), brokerPort=(.+)><>Value
          name: kafka_server_$1_$2
          type: GAUGE
          labels:
           clientId: "$3"
           broker: "$4:$5"
        - pattern: kafka.server<type=(.+), cipher=(.+), protocol=(.+), listener=(.+), networkProcessor=(.+)><>connections
          name: kafka_server_$1_connections_tls_info
          type: GAUGE
          labels:
            cipher: "$2"
            protocol: "$3"
            listener: "$4"
            networkProcessor: "$5"
        - pattern: kafka.server<type=(.+), clientSoftwareName=(.+), clientSoftwareVersion=(.+), listener=(.+), networkProcessor=(.+)><>connections
          name: kafka_server_$1_connections_software
          type: GAUGE
          labels:
            clientSoftwareName: "$2"
            clientSoftwareVersion: "$3"
            listener: "$4"
            networkProcessor: "$5"
        - pattern: "kafka.server<type=(.+), listener=(.+), networkProcessor=(.+)><>(.+):"
          name: kafka_server_$1_$4
          type: GAUGE
          labels:
           listener: "$2"
           networkProcessor: "$3"
        - pattern: kafka.server<type=(.+), listener=(.+), networkProcessor=(.+)><>(.+)
          name: kafka_server_$1_$4
          type: GAUGE
          labels:
           listener: "$2"
           networkProcessor: "$3"
        # Some percent metrics use MeanRate attribute
        # Ex) kafka.server<type=(KafkaRequestHandlerPool), name=(RequestHandlerAvgIdlePercent)><>MeanRate
        - pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*><>MeanRate
          name: kafka_$1_$2_$3_percent
          type: GAUGE
        # Generic gauges for percents
        - pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*><>Value
          name: kafka_$1_$2_$3_percent
          type: GAUGE
        - pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*, (.+)=(.+)><>Value
          name: kafka_$1_$2_$3_percent
          type: GAUGE
          labels:
            "$4": "$5"
        # Generic per-second counters with 0-2 key/value pairs
        - pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*, (.+)=(.+), (.+)=(.+)><>Count
          name: kafka_$1_$2_$3_total
          type: COUNTER
          labels:
            "$4": "$5"
            "$6": "$7"
        - pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*, (.+)=(.+)><>Count
          name: kafka_$1_$2_$3_total
          type: COUNTER
          labels:
            "$4": "$5"
        - pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*><>Count
          name: kafka_$1_$2_$3_total
          type: COUNTER
        # Generic gauges with 0-2 key/value pairs
        - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+), (.+)=(.+)><>Value
          name: kafka_$1_$2_$3
          type: GAUGE
          labels:
            "$4": "$5"
            "$6": "$7"
        - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+)><>Value
          name: kafka_$1_$2_$3
          type: GAUGE
          labels:
            "$4": "$5"
        - pattern: kafka.(\w+)<type=(.+), name=(.+)><>Value
          name: kafka_$1_$2_$3
          type: GAUGE
        # Emulate Prometheus 'Summary' metrics for the exported 'Histogram's.
        # Note that these are missing the '_sum' metric!
        - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+), (.+)=(.+)><>Count
          name: kafka_$1_$2_$3_count
          type: COUNTER
          labels:
            "$4": "$5"
            "$6": "$7"
        - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.*), (.+)=(.+)><>(\d+)thPercentile
          name: kafka_$1_$2_$3
          type: GAUGE
          labels:
            "$4": "$5"
            "$6": "$7"
            quantile: "0.$8"
        - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+)><>Count
          name: kafka_$1_$2_$3_count
          type: COUNTER
          labels:
            "$4": "$5"
        - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.*)><>(\d+)thPercentile
          name: kafka_$1_$2_$3
          type: GAUGE
          labels:
            "$4": "$5"
            quantile: "0.$6"
        - pattern: kafka.(\w+)<type=(.+), name=(.+)><>Count
          name: kafka_$1_$2_$3_count
          type: COUNTER
        - pattern: kafka.(\w+)<type=(.+), name=(.+)><>(\d+)thPercentile
          name: kafka_$1_$2_$3
          type: GAUGE
          labels:
            quantile: "0.$4"
      zookeeper-metrics-config.yml: |
        # See https://github.com/prometheus/jmx_exporter for more info about JMX Prometheus Exporter metrics
        lowercaseOutputName: true
        rules:
        # replicated Zookeeper
        - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+)><>(\\w+)"
          name: "zookeeper_$2"
          type: GAUGE
        - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+), name1=replica.(\\d+)><>(\\w+)"
          name: "zookeeper_$3"
          type: GAUGE
          labels:
            replicaId: "$2"
        - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+), name1=replica.(\\d+), name2=(\\w+)><>(Packets\\w+)"
          name: "zookeeper_$4"
          type: COUNTER
          labels:
            replicaId: "$2"
            memberType: "$3"
        - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+), name1=replica.(\\d+), name2=(\\w+)><>(\\w+)"
          name: "zookeeper_$4"
          type: GAUGE
          labels:
            replicaId: "$2"
            memberType: "$3"
        - pattern: "org.apache.ZooKeeperService<name0=ReplicatedServer_id(\\d+), name1=replica.(\\d+), name2=(\\w+), name3=(\\w+)><>(\\w+)"
          name: "zookeeper_$4_$5"
          type: GAUGE
          labels:
            replicaId: "$2"
            memberType: "$3"
    
    ```
    
- 3) `kubectl apply -f cluster.yaml -n kafka-system` 으로 파일을 적용합니다.
    - 만약 권한 에러가 발생한다면 security account에 대한 보안 컨텍스트 제약 조건을 변경해줍니다.
        
        
        | anyuid | restricted SCC의 모든 기능을 제공하지만 사용자는 모든 UID 및 GID로 실행할 수 있습니다. |
        | --- | --- |
        | privileged | 모든 권한 및 호스트 기능에 대한 액세스를 허용하고 모든 사용자, 그룹, FSGroup 및 모든 SELinux 컨텍스트로 실행할 수 있습니다. |
        
        ```java
        oc adm policy add-scc-to-user anyuid -z default -n kafka-system
        oc adm policy add-scc-to-user privileged -z default -n kafka-system
        
        oc adm policy add-scc-to-user anyuid -z strimzi-cluster-operator-n kafka-system
        oc adm policy add-scc-to-user privileged -z strimzi-cluster-operator -n kafka-system
         
        oc adm policy add-scc-to-user anyuid -z kafka-cluster-zookeeper -n kafka-system
        oc adm policy add-scc-to-user privileged -z kafka-cluster-zookeeper -n kafka-system
        
        oc adm policy add-scc-to-user anyuid -z kafka-cluster-kafka   -n kafka-system
        oc adm policy add-scc-to-user privileged -z kafka-cluster-kafka   -n kafka-system
        
        oc adm policy add-scc-to-user anyuid -z kafka-ui   -n kafka-system
        oc adm policy add-scc-to-user privileged -z kafka-ui   -n kafka-system
        
        oc adm policy add-scc-to-user anyuid -z kafka-cluster-entity-operator    -n kafka-system
        oc adm policy add-scc-to-user privileged -z kafka-cluster-entity-operator  -n kafka-system
        
        oc adm policy add-scc-to-user anyuid -z prometheus-operator  -n kafka-system
        oc adm policy add-scc-to-user privileged -z prometheus-operator -n kafka-system
        
        ```
        
- 4) `kubectl get pod`, `kubectl get svc` 로 설치를 확인합니다.
    
    ```java
    root@edu3:~/icis-tr/kafka/strimzi# k get pod -o wide
    NAME                                             READY   STATUS    RESTARTS        AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
    kafka-cluster-entity-operator-68c69b8788-6nx64   2/2     Running   1 (7m40s ago)   9m49s   10.130.4.93   edu-cloud.worker03   <none>           <none>
    kafka-cluster-kafka-0                            1/1     Running   1 (10m ago)     10m     10.131.4.52   edu-cloud.worker02   <none>           <none>
    kafka-cluster-kafka-1                            1/1     Running   0               10m     10.130.4.92   edu-cloud.worker03   <none>           <none>
    kafka-cluster-kafka-2                            1/1     Running   0               10m     10.131.2.42   edu-cloud.worker01   <none>           <none>
    kafka-cluster-kafka-exporter-54c4dfc96-wrcsl     1/1     Running   0               7m17s   10.131.4.53   edu-cloud.worker02   <none>           <none>
    kafka-cluster-zookeeper-0                        1/1     Running   0               11m     10.131.4.51   edu-cloud.worker02   <none>           <none>
    kafka-cluster-zookeeper-1                        1/1     Running   0               11m     10.130.4.91   edu-cloud.worker03   <none>           <none>
    kafka-cluster-zookeeper-2                        1/1     Running   0               11m     10.131.2.41   edu-cloud.worker01   <none>           <none>
    kafka-ui-7f59cdfc7c-hctlj                        1/1     Running   0               2d9h    10.130.4.25   edu-cloud.worker03   <none>           <none>
    strimzi-cluster-operator-6b8688cf68-bkgd9        1/1     Running   0               15m     10.131.4.50   edu-cloud.worker02   <none>           <none>
    ```
    
    ```java
    root@edu3:~/icis-tr/kafka/strimzi# k get svc
    NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE
    kafka-cluster-kafka-bootstrap         ClusterIP   172.30.26.163    <none>        9091/TCP,9092/TCP,9093/TCP                     4d6h
    kafka-cluster-kafka-brokers           ClusterIP   None             <none>        9090/TCP,9091/TCP,8443/TCP,9092/TCP,9093/TCP   4d6h
    kafka-cluster-kafka-route-0           ClusterIP   172.30.227.123   <none>        9094/TCP                                       4d6h
    kafka-cluster-kafka-route-1           ClusterIP   172.30.134.137   <none>        9094/TCP                                       4d6h
    kafka-cluster-kafka-route-2           ClusterIP   172.30.246.147   <none>        9094/TCP                                       4d6h
    kafka-cluster-kafka-route-bootstrap   ClusterIP   172.30.83.30     <none>        9094/TCP                                       4d6h
    kafka-cluster-zookeeper-client        ClusterIP   172.30.226.98    <none>        2181/TCP                                       4d6h
    kafka-cluster-zookeeper-nodes         ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP                     4d6h
    ```
    
- 5) listener 정보를 추출해 확인합니다. : `kubectl get kafka -n kafka-system kafka-cluster -o jsonpath={.status} | jq -r ".listeners"`
    
    ```yaml
    [
      {
        "addresses": [
          {
            "host": "kafka-cluster-kafka-bootstrap.kafka-system.svc",
            "port": 9092
          }
        ],
        "bootstrapServers": "kafka-cluster-kafka-bootstrap.kafka-system.svc:9092",
        "name": "plain"
      },
      {
        "addresses": [
          {
            "host": "kafka-cluster-kafka-bootstrap.kafka-system.svc",
            "port": 9093
          }
        ],
        "bootstrapServers": "kafka-cluster-kafka-bootstrap.kafka-system.svc:9093",
        "certificates": [
          "-----BEGIN CERTIFICATE-----\nMIIFLTCCAxWgAwIBAgIUU8bPKlNTgUNPo5ZocouoDWmOjYQwDQYJKoZIhvcNAQEN\nBQAwLTETMBEGA1UECgwKaW8uc3RyaW16aTEWMBQGA1UEAwwNY2x1c3Rlci1jYSB2\nMDAeFw0yNDA0MTcwNTMwMjZaFw0yNTA0MTcwNTMwMjZaMC0xEzARBgNVBAoMCmlv\nLnN0cmltemkxFjAUBgNVBAMMDWNsdXN0ZXItY2EgdjAwggIiMA0GCSqGSIb3DQEB\nAQUAA4ICDwAwggIKAoICAQC63a+aoxpMwqnnUfAhtGJai9HCylihiRJ0SUqNJnB3\nRCBPh3/THMeIGeLD6KperlnSwhFZ5gJLjsVlvL2zY6715sYxwvIyGTEAzawU33sL\nlecsYi2a9nIQtBfU8kZor/Pjt/z9N0Ze6F1VxcPZmupLz7pLQ77yvR3Q2y3Clgxt\n+FRypencQRwt0h0ZZGwztP0WUE6czgix/UoF/MAaDu8UfDSQmUwSdZy03r8uxTZt\nozdLreNWvLms7FoQKZKqilU/BCDTZyf/rE87nzX9Jr+OuBiQgp1yDimogmVp1Ph+\n/kSgpBWAdsrHNmkLOL4IXyX7xOGrbEr5AFkc0yQHT5fNgB88l7a4zqJzKJ8UX8Iy\noVTtiS7bVjGq78JbXh5Pm1IOUnA4/pzP4qAJJPB5QSEnVQE/qKbHpyAxwcsAv9Ud\nacHG8lQzJOTK8G4Z2Nw3QZ0zEj1vZem0m5DRQOZsww7ZNDd2URuWePnfTTHf0W4K\nGRXfOETJxnF9SHa055QhDTdAOWaFc4fvYxy7bCXzpcn1mED/dsIgh4YsWeXW5N20\nRk2rk0alk6LUR9wRbL+v4GbUGuIGk/GD+WIEzdM+xrsLZB8RCkKpF6RIjR+Np7C8\ntII2tisGnNX0fUUyiUmKFBFpcxIlJz8CaWdpXkAn5fN+G69a2+VZqjdZ2H9P35Ad\nxQIDAQABo0UwQzAdBgNVHQ4EFgQUlUDZNpe+Mn6eT7WPWS4ujbJYM5swEgYDVR0T\nAQH/BAgwBgEB/wIBADAOBgNVHQ8BAf8EBAMCAQYwDQYJKoZIhvcNAQENBQADggIB\nAJiYu95DBKy0sAMTCGijBnv8TrmEtOFvOzZYYNQ9L7Y3LtBZ85k8wkKK6JnBhMNY\nZafPGChcZbV4ckd0pOBuKUkg2Lff2W+93nE+pOd2/0TkbGuvxuBCdm6xoK1tt7Ny\niVUVdbzz5BFGYUb88kHPqTCp3GilgQ+Pe3o5CyZ8GpoX9QvIntYjO8bh4UtzsG6M\nIeBBH/bZZYEvIZHK/NS5oisJ4+NnJunem0CjfUG4ROdewxzj+Dsm47tFl6Bl4V36\nHKxs8eZwQZRUFfkKEBhXzlwne9C5caEGsdNIlORhvxCAvJM8ipHYQJiran1P60Vx\niW6V8wqTbMn5p1cuegHhYVkNFftmIeR9AFEOBTYj8qbIPulyHabfj+uXbfdo4cXw\nppsZzBV9BtI5p6AU8Gg18ygFIUPAvGLJHihjK+N4+ebxVT3eRwfuVjdt6rW9HDq+\nCRmE5wJqinBDkX5gBGFllRpziouusOvayeedlwqkvfjmLLpSXYRMlPA+yk0JPBIN\n1CFJvoe1JQBNYHJ86qEWLKYXwLv5R9p0hiJ/FHsGyVEj/h30NIdqGsb2Uq01uyTK\nkazBL5CZx4CR3K0Hd2DKuuoadUx9scbMP0JEvoOPMzxuUX/YEMbgoZbzeQvKf9Ir\ncY1CmIe48jml9icHhNG4ozYxy1L8UMGR1MB8XqwybNxh\n-----END CERTIFICATE-----\n"
        ],
        "name": "tls"
      },
      {
        "addresses": [
          {
            "host": "kafka-cluster-kafka-route-bootstrap-kafka-system.apps.211-34-231-93.nip.io",
            "port": 443
          }
        ],
        "bootstrapServers": "kafka-cluster-kafka-route-bootstrap-kafka-system.apps.211-34-231-93.nip.io:443",
        "certificates": [
          "-----BEGIN CERTIFICATE-----\nMIIFLTCCAxWgAwIBAgIUU8bPKlNTgUNPo5ZocouoDWmOjYQwDQYJKoZIhvcNAQEN\nBQAwLTETMBEGA1UECgwKaW8uc3RyaW16aTEWMBQGA1UEAwwNY2x1c3Rlci1jYSB2\nMDAeFw0yNDA0MTcwNTMwMjZaFw0yNTA0MTcwNTMwMjZaMC0xEzARBgNVBAoMCmlv\nLnN0cmltemkxFjAUBgNVBAMMDWNsdXN0ZXItY2EgdjAwggIiMA0GCSqGSIb3DQEB\nAQUAA4ICDwAwggIKAoICAQC63a+aoxpMwqnnUfAhtGJai9HCylihiRJ0SUqNJnB3\nRCBPh3/THMeIGeLD6KperlnSwhFZ5gJLjsVlvL2zY6715sYxwvIyGTEAzawU33sL\nlecsYi2a9nIQtBfU8kZor/Pjt/z9N0Ze6F1VxcPZmupLz7pLQ77yvR3Q2y3Clgxt\n+FRypencQRwt0h0ZZGwztP0WUE6czgix/UoF/MAaDu8UfDSQmUwSdZy03r8uxTZt\nozdLreNWvLms7FoQKZKqilU/BCDTZyf/rE87nzX9Jr+OuBiQgp1yDimogmVp1Ph+\n/kSgpBWAdsrHNmkLOL4IXyX7xOGrbEr5AFkc0yQHT5fNgB88l7a4zqJzKJ8UX8Iy\noVTtiS7bVjGq78JbXh5Pm1IOUnA4/pzP4qAJJPB5QSEnVQE/qKbHpyAxwcsAv9Ud\nacHG8lQzJOTK8G4Z2Nw3QZ0zEj1vZem0m5DRQOZsww7ZNDd2URuWePnfTTHf0W4K\nGRXfOETJxnF9SHa055QhDTdAOWaFc4fvYxy7bCXzpcn1mED/dsIgh4YsWeXW5N20\nRk2rk0alk6LUR9wRbL+v4GbUGuIGk/GD+WIEzdM+xrsLZB8RCkKpF6RIjR+Np7C8\ntII2tisGnNX0fUUyiUmKFBFpcxIlJz8CaWdpXkAn5fN+G69a2+VZqjdZ2H9P35Ad\nxQIDAQABo0UwQzAdBgNVHQ4EFgQUlUDZNpe+Mn6eT7WPWS4ujbJYM5swEgYDVR0T\nAQH/BAgwBgEB/wIBADAOBgNVHQ8BAf8EBAMCAQYwDQYJKoZIhvcNAQENBQADggIB\nAJiYu95DBKy0sAMTCGijBnv8TrmEtOFvOzZYYNQ9L7Y3LtBZ85k8wkKK6JnBhMNY\nZafPGChcZbV4ckd0pOBuKUkg2Lff2W+93nE+pOd2/0TkbGuvxuBCdm6xoK1tt7Ny\niVUVdbzz5BFGYUb88kHPqTCp3GilgQ+Pe3o5CyZ8GpoX9QvIntYjO8bh4UtzsG6M\nIeBBH/bZZYEvIZHK/NS5oisJ4+NnJunem0CjfUG4ROdewxzj+Dsm47tFl6Bl4V36\nHKxs8eZwQZRUFfkKEBhXzlwne9C5caEGsdNIlORhvxCAvJM8ipHYQJiran1P60Vx\niW6V8wqTbMn5p1cuegHhYVkNFftmIeR9AFEOBTYj8qbIPulyHabfj+uXbfdo4cXw\nppsZzBV9BtI5p6AU8Gg18ygFIUPAvGLJHihjK+N4+ebxVT3eRwfuVjdt6rW9HDq+\nCRmE5wJqinBDkX5gBGFllRpziouusOvayeedlwqkvfjmLLpSXYRMlPA+yk0JPBIN\n1CFJvoe1JQBNYHJ86qEWLKYXwLv5R9p0hiJ/FHsGyVEj/h30NIdqGsb2Uq01uyTK\nkazBL5CZx4CR3K0Hd2DKuuoadUx9scbMP0JEvoOPMzxuUX/YEMbgoZbzeQvKf9Ir\ncY1CmIe48jml9icHhNG4ozYxy1L8UMGR1MB8XqwybNxh\n-----END CERTIFICATE-----\n"
        ],
        "name": "route"
      }
    ]
    
    ```
    

## 2-4. Kafka 외부 접속을 위한 인증서 저장

- 1) Kafka Topic 생성 : `vi topic.yaml` > `kubectl apply -f topic.yaml`
    
    ```yaml
    apiVersion: kafka.strimzi.io/v1beta1
    kind: KafkaTopic
    metadata:
      name: ppon #topic명
      namespace: kafka-system
      labels:
        strimzi.io/cluster: "kafka-cluster"
    spec:
      partitions: 3 #파티션 수
      replicas: 1
    ```
    
    - `kubectl get kafkatopic` 을 통해 생성 확인
        
        ```java
        NAME         CLUSTER         PARTITIONS   REPLICATION FACTOR   READY
        ppon         kafka-cluster   3            1                    True
        ```
        
- 2) Kafka User 생성 : `vi user.yaml` > `kubectl apply -f user.yaml`
    
    ```yaml
    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaUser
    metadata:
      name: order-user #내가 원하는 user이름으로 변경
      labels:
        strimzi.io/cluster: kafka-cluster
    spec:
      authentication:
        type: tls
    ```
    
    - `kubectl get kafkauser`을 통해 `READY=True`상태가 되었는지 확인
        
        ```java
        NAME         CLUSTER         AUTHENTICATION   AUTHORIZATION   READY
        order-user   kafka-cluster   tls                              True
        ```
        
- 3) 인증서 저장 및 비밀번호 확인
    
    ```java
    kubectl get secret kafka-cluster-cluster-ca-cert -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
    kubectl get secret kafka-cluster-cluster-ca-cert -o jsonpath='{.data.ca\.p12}' | base64 -d > ca.p12
    kubectl get secret kafka-cluster-cluster-ca-cert -o jsonpath='{.data.ca\.password}' | base64 -d
    ---- OUTPUT ----
    VGQYKZGFncZR
    
    kubectl get secret <kafkaUser이름> -o jsonpath='{.data.user\.p12}' | base64 -d > user.p12
    kubectl get secret <kafkaUser이름> -o jsonpath='{.data.user\.password}' | base64 -d
    ---- OUTPUT ----
    XdAXpqlDwgwgzS8kC9yNRmPs7zRfxbZ2
    ```
    
- 4) 외부에서 접속하기 위해 `ca.p12` 와 `user.p12` 를 `src/resources` 경로에 저장합니다
    
    <img width="339" alt="image" src="https://github.com/heewon00/k8s_kafka_guide/assets/55778040/b60d229e-9ef2-430a-9767-4b44e0e91aa2">

    
- 5) `application.yaml` 을 아래와 같이 설정합니다.
    
    ```java
      kafka:
        bootstrap-servers: kafka-cluster-kafka-route-bootstrap-kafka-system.apps.211-34-231-93.nip.io:443 #<본인 host주소>:443으로 변경
        properties:
          security:
            protocol: SSL
        ssl:
          trust-store-location: classpath:/ca.p12 
          trust-store-type: PKCS12
          trust-store-password: VGQYKZGFncZR #ca.p12 비밀번호로 변경
          key-store-location: classpath:/user.p12
          key-store-type: PKCS12
          key-store-password: XdAXpqlDwgwgzS8kC9yNRmPs7zRfxbZ2 #user.p12 비밀번호로 변경
        listener:
          ack-mode: MANUAL_IMMEDIATE
        consumer:
          group-id: ppon #원하는 group-id 설정
        topics: ppon #내 topic이름으로 변경
    ```
    
    - (참고) SASL, TSL 사용 여부에 따른 프로토콜 값
        
        `sasl`및 `tls`값을 사용하여 수신기에 매핑할 프로토콜을 결정하는 프로토콜 맵이 생성됩니다 
        
        - SASL = 참, TLS = 참 → `SASL_SSL`
        - SASL = 거짓, TLS = 참 → `SSL`
        - SASL = 참, TLS = 거짓 → `SASL_PLAINTEXT`
        - SASL = 거짓, TLS = 거짓 → `PLAINTEXT`
- application을 실행했을 때 아래와 같이 log가 뜨면 성공
    
    ```java
     KAFKA  15:19:10.014 INFO  KafkaConsumerApplication : - Started KafkaConsumerApplication in 1.065 seconds (process running for 1.434) 
     KAFKA  15:19:27.542 INFO  KafkaMessageListenerContainer : - ppon: partitions assigned: [ppon-0, ppon-1, ppon-2] 
    ```
    

# 3. UI for Apache kafka 설치

Apache Kafka용 UI는 Apache Kafka 클러스터를 모니터링하고 관리하기 위한 무료 오픈 소스 웹 UI입니다.

대시보드를 사용하면 브로커, 주제, 파티션, consumer와 같은 Kafka 클러스터의 메트릭을 추적할 수 있습니다.

---

helm을 통해 설치합니다.  (helm이 설치되어 있지 않다면 [🔗helm설치](https://github.com/kt-cloudnative/education/blob/master/chapter5.md#helm-설치--httpshelmshkodocsintroinstall-) 참고하여 설치)

- `helm repo add kafka-ui https://provectus.github.io/kafka-ui-charts`
- `helm repo update`
- `helm search repo kafka-ui`
    
    ```java
    root@edu3:~/icis-tr/kafka/strimzi# helm search repo kafka-ui
    NAME             	CHART VERSION	APP VERSION	DESCRIPTION              
    kafka-ui/kafka-ui	0.7.5        	v0.7.1     	A Helm chart for kafka-U
    ```
    
- `helm show values kafka-ui/kafka-ui > values.yaml`
- `vi values.yaml` 을 통해 파일을 수정합니다.
    - svc 정보를 확인하고 수정합니다.
    
    ```java
    root@edu3:~/icis-tr/kafka/strimzi# k get svc
    NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE
    kafka-cluster-kafka-bootstrap         ClusterIP   172.30.26.163    <none>        9091/TCP,9092/TCP,9093/TCP                     4d6h
    ```
    
    ```yaml
     26   kafka:
     27     clusters:
     28       - name: order
     29         bootstrapServers: kafka-cluster-kafka-bootstrap.kafka-system.svc:9092 #svc입력
    ```
    
- 설치 : `helm install kafka-ui -f values.yaml kafka-ui/kafka-ui`
    
    ```java
    root@edu3:~/icis-tr/kafka/strimzi# k get svc
    NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE                   4d6h
    kafka-ui                              ClusterIP   172.30.27.103    <none>        80/TCP                                         20d
    ```
    
- route 생성 : `route.yaml` > `kubectl apply -f route.yaml -n kafka-system`
    - targetPort 이름 확인
        
        ```java
        root@edu3:~/icis-tr/kafka/strimzi# k edit svc kafka-ui
          ports:
          - name: http
            port: 80
            protocol: TCP
            targetPort: http
        ```
        
        혹은 okd UI에서 확인
        
        ![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/8056125d-7bcc-4ffc-88fa-a15dcf97cafd)

        
    
    ```yaml
    apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: kafka-ui
      namespace: kafka-system
    spec:
      host: kafka-ui.apps.211-34-231-93.nip.io **#본인 host로 변경!!**
      to:
        kind: Service
        name: kafka-ui #svc 이름
        weight: 100
      port:
        targetPort: http #위의 설명 참고하여 port name이 다르다면 변경
      tls:
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
      wildcardPolicy: None
    ```
    
- route에 접속해 확인합니다.
    - [https://kafka-ui.apps.211-34-231-93.nip.io](https://kafka-ui.apps.211-34-231-93.nip.io/)
    
   ![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/2ee71e9d-41fc-433d-bc58-566d5f3428a2)


# 4. Kafka Grafana Dashboard

2-3에서 Cluster을 설치하면서 Prometheus Exporter을 추가해줬습니다.

Grafana에서 Dashboard형태로 볼 수 있도록 Prometheus를 설치하겠습니다. 

( 이 문서에서는 Grafana설치에 대해서는 다루고 있지 않습니다. 

Grafana helm으로 설치하기 : https://peterica.tistory.com/301)

---

## 4-1. Prometheus 설치

- 1) Prometheus Operator Crd 및 Prometheus Operator 설치  
    - `curl -s https://raw.githubusercontent.com/AmarendraSingh88/kafka-on-kubernetes/main/kafka-demo/demo3-monitoring/prometheus-operator-deployment.yaml | sed -e 's/namespace: monitoring/namespace: kafka-system/' > prometheus-operator-deploy.yaml`
    - `kubectl apply -f prometheus-operator-deploy.yaml`
- 2) Pod Monitor 설치  
    - `cat <strimzi 4.0 파일경로>/examples/metrics/prometheus-install/strimzi-pod-monitor.yaml | sed -e 's/- myproject/- kafka-system/' > strimzi-pod-monitor.yaml`
    - `kubectl apply -f strimzi-pod-monitor.yaml`
- 3) Prometheus Rule 설치   
    - `cat <strimzi 4.0 파일경로>/examples/metrics/prometheus-install/prometheus-rules.yaml > prometheus-rules.yaml`
    - `kubectl apply -f prometheus-rules.yaml`
- 4) prometheus 설치 : `vi prometheus.yaml` > `kubectl apply -f prometheus.yaml`
    
    ```yaml
    #prometheus.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: prometheus-server
      labels:
        app: strimzi
    rules:
      - apiGroups: [""]
        resources:
          - nodes
          - nodes/proxy
          - services
          - endpoints
          - pods
        verbs: ["get", "list", "watch"]
      - apiGroups:
          - extensions
        resources:
          - ingresses
        verbs: ["get", "list", "watch"]
      - nonResourceURLs: ["/metrics"]
        verbs: ["get"]
    
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: prometheus-server
      labels:
        app: strimzi
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: prometheus-server
      labels:
        app: strimzi
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: prometheus-server
    subjects:
      - kind: ServiceAccount
        name: prometheus-server
        namespace: kafka-system
    ---
    apiVersion: monitoring.coreos.com/v1
    kind: Prometheus
    metadata:
      name: prometheus
      labels:
        app: strimzi
    spec:
      replicas: 1
      serviceAccountName: prometheus-server
      podMonitorSelector:
        matchLabels:
          app: strimzi
      serviceMonitorSelector: {}
      enableAdminAPI: false
      image: quay.io/prometheus/prometheus:v2.33.4
      podMonitorNamespaceSelector: {}
      podMonitorSelector: {}
    ```
    
- 설치 확인
    
    ```java
    root@edu3:~/icis-tr/kafka/strimzi# k get all
    NAME                                                 READY   STATUS    RESTARTS        AGE
    pod/kafka-cluster-entity-operator-68c69b8788-sp6xx   2/2     Running   1 (4d7h ago)    4d7h
    pod/kafka-cluster-kafka-0                            1/1     Running   0               4d7h
    pod/kafka-cluster-kafka-1                            1/1     Running   0               4d7h
    pod/kafka-cluster-kafka-2                            1/1     Running   0               4d7h
    pod/kafka-cluster-kafka-exporter-54c4dfc96-xqst8     1/1     Running   0               4d7h
    pod/kafka-cluster-zookeeper-0                        1/1     Running   0               4d7h
    pod/kafka-cluster-zookeeper-1                        1/1     Running   0               4d7h
    pod/kafka-cluster-zookeeper-2                        1/1     Running   0               4d7h
    pod/kafka-ui-7c8d46d5cf-9c9l4                        1/1     Running   0               4d6h
    pod/kafka-ui-7c8d46d5cf-pd49p                        1/1     Running   0               4d6h
    pod/prometheus-operator-5878fc6567-qt48r             1/1     Running   0               13d
    pod/prometheus-prometheus-0                          2/2     Running   0               13d
    pod/strimzi-cluster-operator-6b8688cf68-9hkbs        1/1     Running   2 (3d16h ago)   4d7h
    
    NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE
    service/kafka-cluster-kafka-bootstrap         ClusterIP   172.30.26.163    <none>        9091/TCP,9092/TCP,9093/TCP                     4d7h
    service/kafka-cluster-kafka-brokers           ClusterIP   None             <none>        9090/TCP,9091/TCP,8443/TCP,9092/TCP,9093/TCP   4d7h
    service/kafka-cluster-kafka-route-0           ClusterIP   172.30.227.123   <none>        9094/TCP                                       4d7h
    service/kafka-cluster-kafka-route-1           ClusterIP   172.30.134.137   <none>        9094/TCP                                       4d7h
    service/kafka-cluster-kafka-route-2           ClusterIP   172.30.246.147   <none>        9094/TCP                                       4d7h
    service/kafka-cluster-kafka-route-bootstrap   ClusterIP   172.30.83.30     <none>        9094/TCP                                       4d7h
    service/kafka-cluster-zookeeper-client        ClusterIP   172.30.226.98    <none>        2181/TCP                                       4d7h
    service/kafka-cluster-zookeeper-nodes         ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP                     4d7h
    service/kafka-ui                              ClusterIP   172.30.27.103    <none>        80/TCP                                         20d
    service/prometheus-operated                   ClusterIP   None             <none>        9090/TCP                                       13d
    service/prometheus-operator                   ClusterIP   None             <none>        8080/TCP                                       13d
    
    NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/kafka-cluster-entity-operator   1/1     1            1           4d7h
    deployment.apps/kafka-cluster-kafka-exporter    1/1     1            1           4d7h
    deployment.apps/kafka-ui                        2/2     2            2           20d
    deployment.apps/prometheus-operator             1/1     1            1           13d
    deployment.apps/strimzi-cluster-operator        1/1     1            1           4d7h
    
    ...
    ```
    

## 4-2. Grafana 접속 및 Monitoring Dashboard 생성

### 4-2-1. Data Source 추가

- Grafana에 접속 > Connections > Data Sources > Add new data source 를 클릭합니다.
    
    ![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/1385f18e-8b16-4080-b1d3-6f0ed3b61453)

    
    - Prometheus server URL 은 `kubectl get svc` 를 통해 확인
        
        ```java
        root@edu3:~/icis-tr/kafka/strimzi# k get svc
        NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE
        ...           
        prometheus-operated                   ClusterIP   None             <none>        9090/TCP    
        ```
        
        예시) [http://prometheus-operated.kafka-system.svc.cluster.local:9090](http://prometheus-operated.kafka-system.svc.cluster.local:9090/)
        
    - 위와 같이 작성하고 save&test 클릭

### 4-2-2. Dashboard 생성

- Dashboard > New > import 클릭
    
    ![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/f5e7da86-a9b5-4a3d-b0e3-2157aa64604e)

    
- https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/grafana-dashboards/strimzi-kafka.json 에 있는 값을 붙여넣기 하고 Load를 클릭합니다.
    
    ![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/fcfb755d-46fa-41f6-997c-2dff1b125b82)

    
- import 하면 다음과 같은 화면이 뜨는 것을 확인할 수 있습니다.
    
   ![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/043cbbc1-2fb7-4ed2-b684-2c48b3211c6d)

    

## 4-3. (선택) Prometheus Route 생성

- route 생성 : `route.yaml` > `kubectl apply -f route.yaml`
    
    ```yaml
    apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: kafka-prometheus
      namespace: kafka-system
    spec:
      host: kafka-prometheus.apps.211-34-231-93.nip.io #본인 host 이름으로 변경
      to:
        kind: Service
        name: prometheus-operated # Prometheus svc 이름
        weight: 100
      port:
        targetPort: web
      tls:
        termination: edge
        insecureEdgeTerminationPolicy: Redirect
      wildcardPolicy: None
    ```
    
- Prometheus Route 접속 > 상단바 Status > Targets에서 podMonitor정보를 확인할 수 있습니다.
    
    ![image](https://github.com/heewon00/k8s_kafka_guide/assets/55778040/fd369f89-c2fc-49a1-955f-98be40b3c496)

    

# 5. 참고 URL

- strimzi operator란? : https://malwareanalysis.tistory.com/346
- kafka strimzi 설치
    - https://sotl.tistory.com/292
    - https://blog.valensas.com/monitoring-strimzi-kafka-clusters-with-prometheus-3e0c43d04db5
    - https://www.entechlog.com/blog/kafka/create-kafka-cluster-on-kubernetes-using-strimzi/
    - https://github.com/strimzi/strimzi-kafka-operator/tree/main/examples/metrics
- TLS : https://medium.com/@lbroudoux/deploying-strimzi-kafka-and-java-clients-with-security-part-1-authentication-1b4e10e6ab16
- UI : https://pastime2532.tistory.com/257
- 테스트 : https://mokpolar.tistory.com/25
