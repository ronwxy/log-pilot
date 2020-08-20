log-pilot-filebeat-7.3.1
=========

### 1. upgrade filebeat version

Dockerfile.filebeat:

original: 

```yaml
ENV FILEBEAT_VERSION=6.1.1-3
COPY assets/glibc/glibc-2.26-r0.apk /tmp/
RUN apk update && \ 
    apk add python && \
    apk add ca-certificates && \
    apk add wget && \
    update-ca-certificates && \
    wget http://acs-logging.oss-cn-hangzhou.aliyuncs.com/beats/filebeat/filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz -P /tmp/ && \
    mkdir -p /etc/filebeat /var/lib/filebeat /var/log/filebeat && \
```

to  (download filebeat package and put in the log-pilot folder)

```yaml
ENV FILEBEAT_VERSION=7.3.1
COPY assets/glibc/glibc-2.26-r0.apk /tmp/
COPY filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz /tmp/
RUN apk update && \ 
    apk add python && \
    apk add ca-certificates && \
    apk add wget && \
    update-ca-certificates && \
    mkdir -p /etc/filebeat /var/lib/filebeat /var/log/filebeat && \
```

### 2. upgrade filebeat config

assets/filebeat/config.filebeat:

* remove `filebeat.registry_file: /var/lib/filebeat/registry`
* update `filebeat.config.prospectors:` to `filebeat.config.inputs:`

original:
```$xslt
base() {
cat >> $FILEBEAT_CONFIG << EOF
path.config: /etc/filebeat
path.logs: /var/log/filebeat
path.data: /var/lib/filebeat/data
filebeat.registry_file: /var/lib/filebeat/registry
filebeat.shutdown_timeout: ${FILEBEAT_SHUTDOWN_TIMEOUT:-0}
logging.level: ${FILEBEAT_LOG_LEVEL:-info}
logging.metrics.enabled: ${FILEBEAT_METRICS_ENABLED:-false}
logging.files.rotateeverybytes: ${FILEBEAT_LOG_MAX_SIZE:-104857600}
logging.files.keepfiles: ${FILEBEAT_LOG_MAX_FILE:-10}
logging.files.permissions: ${FILEBEAT_LOG_PERMISSION:-0600}
${FILEBEAT_MAX_PROCS:+max_procs: ${FILEBEAT_MAX_PROCS}}
setup.template.name: "${FILEBEAT_INDEX:-filebeat}"
setup.template.pattern: "${FILEBEAT_INDEX:-filebeat}-*"
filebeat.config:
    prospectors:
        enabled: true
        path: \${path.config}/prospectors.d/*.yml
        reload.enabled: true
        reload.period: 10s
EOF
}
```
updated:
```$xslt
base() {
cat >> $FILEBEAT_CONFIG << EOF
path.config: /etc/filebeat
path.logs: /var/log/filebeat
path.data: /var/lib/filebeat/data
filebeat.shutdown_timeout: ${FILEBEAT_SHUTDOWN_TIMEOUT:-0}
logging.level: ${FILEBEAT_LOG_LEVEL:-info}
logging.metrics.enabled: ${FILEBEAT_METRICS_ENABLED:-false}
logging.files.rotateeverybytes: ${FILEBEAT_LOG_MAX_SIZE:-104857600}
logging.files.keepfiles: ${FILEBEAT_LOG_MAX_FILE:-10}
logging.files.permissions: ${FILEBEAT_LOG_PERMISSION:-0600}
${FILEBEAT_MAX_PROCS:+max_procs: ${FILEBEAT_MAX_PROCS}}
setup.template.name: "${FILEBEAT_INDEX:-filebeat}"
setup.template.pattern: "${FILEBEAT_INDEX:-filebeat}-*"
filebeat.config.inputs:
        enabled: true
        path: \${path.config}/prospectors.d/*.yml
        reload.enabled: true
        reload.period: 10s
EOF
}
```

### 3. get docker image

1.build from source code

```shell script
[root@kmaster]# git clone https://github.com/ronwxy/log-pilot.git
[root@kmaster]# cd log-pilot/ 
[root@kmaster]# git checkout filebeat-7.3.1
[root@kmaster]# ./build-image.sh
```

2.pull from registry

```shell script
[root@kmaster]# docker pull registry.cn-hangzhou.aliyuncs.com/jboost/log-pilot:filebeat-7.3.1
```

### 4. deploy log-pilot as DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-pilot-filebeat
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-pilot-filebeat
  template:
    metadata:
      labels:
        app: log-pilot-filebeat
    spec:
      containers:
      - name: log-pilot-filebeat
        #image: registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat
        image: registry.cn-hangzhou.aliyuncs.com/jboost/log-pilot:filebeat-7.3.1
        env:
        - name: "NODE_NAME"
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: "PILOT_LOG_PREFIX"
          value: "k8s"
        - name: "LOGGING_OUTPUT"
          value: "logstash"
        - name: "LOGSTASH_HOST"
          value: "{your-logstash-host}"
        - name: "LOGSTASH_PORT"
          value: "5044"
        volumeMounts:
        - name: sock
          mountPath: /var/run/docker.sock
        - name: root
          mountPath: /host
          readOnly: true
        - name: varlib
          mountPath: /var/lib/filebeat
        - name: varlog
          mountPath: /var/log/filebeat
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
        livenessProbe:
          failureThreshold: 3
          exec:
            command:
            - /pilot/healthz
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      volumes:
      - name: sock
        hostPath:
          path: /var/run/docker.sock
      - name: root
        hostPath:
          path: /
      - name: varlib
        hostPath:
          path: /var/lib/filebeat
          type: DirectoryOrCreate
      - name: varlog
        hostPath:
          path: /var/log/filebeat
          type: DirectoryOrCreate
      - name: localtime
        hostPath:
          path: /etc/localtime
```

### 5. application deployment config (only show part related with log collecting)

```yaml
spec:
      containers:
      - env:
        - name: k8s_logs_frameworktest
          value: /mnt/logs/app*.log

        volumeMounts:
        - mountPath: /mnt/logs
          name: app-log

      volumes:
      - emptyDir: {}
        name: app-log
```