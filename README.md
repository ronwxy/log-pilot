log-pilot-filebeat-7.3.1
=========

### 1.upgrade filebeat

Dockerfile.filebeat:

change 

```yaml
ENV FILEBEAT_VERSION=6.1.1-3
```

to 

```yaml
ENV FILEBEAT_VERSION=7.3.1
```

### 2.upgrade filebeat config

assets/filebeat/config.filebeat:

* remove `filebeat.registry_file: /var/lib/filebeat/registry`
* update `filebeat.config.prospectors:` to `filebeat.config.inputs:`

origin:
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



