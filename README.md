# Digit Helm Deployment Common Chart

To create a kubernetes helm chart for deployments, we need the key attributes [Replicas, image, init containers, etc..] 
and its values to be passed for each service mandatorily.  
The common templates has the required files to generate deployment manifest, service with common variables and values based on user input.

## Requirements

The values template file [values.yaml](https://github.com/egovernments/eGov-infraOps/blob/helm/helm/charts/common/values.yaml) has the common attributes and its values required for all manifest files.

The service template file [_service.yaml](https://github.com/egovernments/eGov-infraOps/blob/helm/helm/charts/common/templates/_service.yaml) has the attributes and values for generating service manifest.

The ingress template file [_ingress.yaml](https://github.com/egovernments/eGov-infraOps/blob/helm/helm/charts/common/templates/_ingress.yaml) has the attributes and values for generating ingress manifest.

The deployment template file [_deployment.yaml](https://github.com/egovernments/eGov-infraOps/blob/helm/helm/charts/common/templates/_deployment.yaml) has the attributes and values for generating deployment manifest.

## Values template

-------------------------------------------------------------------------------------------------------------------------------
```yaml
namespace: egov  #default namespace for the service/deployment
replicas: 1      # number of pods
httpPort: 8080   # default port for this service/deployment
appType: ""
```
Most of the services such as core services, business services, municipal services will be created under namespace “egov”.

By default the replicas count is “1”. This means one pod will be created for the service.

To increase the replica count we need to override the variables for that particular service.

Default port number of a service is “8080”

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
ingress:
  enabled: false
```
When ingress enabled is false, The ingress configuration will not be generated for this service. There will be no access to the service via ingress using context path [Eg: https://<domain-name>/<service-name>]. 

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
zuul: false
```
The Zuul configuration will not be generated for this service. The access for this service will not come via zuul when enabled false.

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/app-root: /citizen  
```
The annotation will claim Nginx as our default ingress controller and the application default context path is set to /citizen.

-------------------------------------------------------------------------------------------------------------------------------
  ```yaml
  waf: 
    enabled: true
    annotations:
      nginx.ingress.kubernetes.io/lua-resty-waf: "active"  
      nginx.ingress.kubernetes.io/lua-resty-waf-debug: "true"
      nginx.ingress.kubernetes.io/lua-resty-waf-score-threshold: "10"
      nginx.ingress.kubernetes.io/lua-resty-waf-allow-unknown-content-types: "true"
      nginx.ingress.kubernetes.io/lua-resty-waf-process-multipart-body: "false"              
```
Web application firewall will be enabled when waf.enabled is true. Enabling WAF helps to avoid cyber attacks on the application.

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
image:
  pullPolicy: IfNotPresent   
  tag: latest
```
To pull the latest image from the docker repository if the image is not present. 

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
affinity:
    preferSpreadAcrossAZ: true
```
To create the pods across multiple availability zones in the cloud infrastructure. Which is useful when more than 1 replicas created for the service.

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
initContainers:
  dbMigration:
    enabled: false
    image:
      pullPolicy: IfNotPresent  
      tag: latest
    env: |
        - name: "DB_URL"
          valueFrom: 
            configMapKeyRef: 
              name: egov-config
              key: db-url
        - name: "SCHEMA_TABLE"
          value: {{ .Values.initContainers.dbMigration.schemaTable | quote }}              
        - name: "FLYWAY_USER"
          valueFrom: 
            secretKeyRef: 
                name: db
                key: flyway-username
        - name: "FLYWAY_PASSWORD"
          valueFrom:
            secretKeyRef: 
                name: db
                key: flyway-password
        - name: "FLYWAY_LOCATIONS"
          valueFrom: 
            configMapKeyRef: 
                name: egov-config
                key: flyway-locations
```
When DB migration is set to false, the init container configuration to DB migration will not be created. When enabled true the DB migration scripts will run from the init container before starting the service pod. 
The environment variables [env] passed in this section will establish the Database connectivity and run the database scripts from the mentioned Flyway location.

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
  gitSync:
    enabled: false
    image:
      repository: k8s.gcr.io/git-sync
      tag: v3.1.1
      pullPolicy: IfNotPresent   
    env: |
        - name: "GIT_SYNC_REPO"
          value: "{{ .Values.initContainers.gitSync.repo }}"
        - name: "GIT_SYNC_BRANCH"
          value: "{{ .Values.initContainers.gitSync.branch }}"        
        - name: "GIT_SYNC_DEPTH"
          value: "1"            
        - name: "GIT_SYNC_ONE_TIME"
          value: "true"          
        - name: "GIT_SYNC_SSH"
          value: "true"      
        - name: "GIT_SYNC_ROOT"
          value: "/work-dir"
```
When gitSync.enabled is false, the git synchronization will be ignored. 
If the gitSync.enabled is true, The specified git docker image will be executed before starting the service pod. The mentioned environment variables are the git repository specifications which needs to be passed as cluster environment [dev, QA, etc.] specific variable.
GIT_SYNC_REPO -  Git Repository URL
GIT_SYNC_BRANCH - Repository Branch
GIT_SYNC_SSH - Enabled true for SSH based synchronization
GIT_SYNC_ROOT - Destination path to synchronize

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
healthChecks:
    enabled: false
    livenessProbe: |
        httpGet:
            path: "{{ .Values.healthChecks.livenessProbePath }}"
            port: {{ .Values.httpPort }}
        initialDelaySeconds: 60
        timeoutSeconds: 3
        periodSeconds: 60
        successThreshold: 1
        failureThreshold: 5
    readinessProbe: |
        httpGet:
            path: "{{ .Values.healthChecks.readinessProbePath }}"
            port: {{ .Values.httpPort }}
        initialDelaySeconds: 60
        timeoutSeconds: 3
        periodSeconds: 30
        successThreshold: 1
        failureThreshold: 5        
```
When healthchecks.enabled is false. The periodic health check of the pod availability is disabled. 
When enabled true the liveness and readiness of the pod will be checked periodically with the mentioned httpget path and mentioned port for every periodSeconds mentioned. 
When the pod is not reachable after the initialDelaySeconds [the time once the pod is launched] initially and not reachable for the mentioned failureThreshold it will restart the pod.

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
lifecycle:
    preStop:
        exec:
            command:
            - sh
            - -c
            - "sleep 10"
```
The lifecycle preStop will run the sleep command for 10 seconds to kill the container in the pod gracefully.

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
memory_limits: "512Mi"
```
To set the memory limit for the pod to 512 MB. 

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
resources: |
  {{- if eq .Values.appType "java-spring" -}}
  requests:
    memory: {{ .Values.memory_limits | quote }}
  limits:
    memory: {{ .Values.memory_limits | quote }}
  {{- end -}}
```
To set the memory limit for the service. The values for the memory request and memory limit are added from the environment variables [eg: dev, QA] for this service. 

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
## Allows specification of additional environment variables
extraEnv:
  java: |
      - name: SPRING_DATASOURCE_URL
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: db-url
      - name: FLYWAY_ENABLED
        value: "false"      
      - name: APP_TIMEZONE
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: timezone              
      - name: FLYWAY_URL
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: db-url
      - name: SPRING_DATASOURCE_USERNAME
        valueFrom:
          secretKeyRef:
            name: db
            key: username
      - name: SPRING_DATASOURCE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db
            key: password
      - name: SPRING_DATASOURCE_TOMCAT_INITIAL_SIZE
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: spring-datasource-tomcat-initialSize
      - name: SERVER_TOMCAT_MAX_THREADS
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: server-tomcat-max-threads
      - name: SERVER_TOMCAT_MAX_CONNECTIONS
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: server-tomcat-max-connections
      - name: SPRING_DATASOURCE_TOMCAT_MAX_ACTIVE
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: spring-datasource-tomcat-max-active  
      - name: KAFKA_CONFIG_BOOTSTRAP_SERVER_CONFIG
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: kafka-brokers
      - name: SPRING_KAFKA_BOOTSTRAP_SERVERS
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: kafka-brokers
      - name: SPRING_JPA_SHOW_SQL
        valueFrom:
          configMapKeyRef:
            name: egov-config
            key: spring-jpa-show-sql           
```
If the service requires environment variables like Flyway, DB Connection, Kafka connection, etc.. for the deployment. All the additional environment variables will override from the Configmaps and Secrets of the cluster specific [dev, QA, etc..] environment variables. 

-------------------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------------------
```yaml
    jaeger: |             
      - name: JAEGER_SERVICE_NAME
        value: {{ .Chart.Name }}
      - name: JAEGER_SAMPLER_TYPE
        value: remote
      - name: JAEGER_AGENT_HOST
        valueFrom:
          fieldRef:
            fieldPath: status.hostIP
      - name: JAEGER_AGENT_PORT
        value: "6831"
      - name: JAEGER_SAMPLER_MANAGER_HOST_PORT
        value: "$(JAEGER_AGENT_HOST):5778"
```        
Jaeger API tracing will be enabled for this service by enabling this block. The traces metrics will be sent to Jaeger agent to monitor the service.

-------------------------------------------------------------------------------------------------------------------------------
```yaml
## Additional init containers
extraInitContainers: |

## Additional sidecar containers
extraContainers: |

## Add additional volumes and mounts, e. g. for custom themes
extraVolumes: |
extraVolumeMounts: |

additionalLabels: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000           
```


