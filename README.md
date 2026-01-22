Setup Fluentd on critical environments for log data collection and integrate with elasticsearch
Architecture Overview
Kubernetes Node Logs
   (/var/log/containers/*.log)
              │
              ▼
        Fluentd (DaemonSet)
              │
              ▼
      Elasticsearch (StatefulSet)
              │
              ▼
          Kibana (Ingress/UI)

Scope of This Document
This document covers only the Helm sub-charts under:
RepoDir:
├── elasticsearch/
├── kibana/
└── fluentd/
Elasticsearch Helm Chart
Directory Structure
elasticsearch/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── secret.yaml
    ├── service.yaml
    └── statefulset.yaml

File Purpose
Defines metadata for the Elasticsearch Helm chart.
Tested  elasticsearch/Chart.yaml

apiVersion: v2
name: elasticsearch
description: Elasticsearch subchart for EFK Stack
type: application
version: 0.1.0
appVersion: "7.17.15"

Explanation
Field
Meaning
apiVersion: v2
Helm 3 chart format
name
Chart name (used by parent chart)
type
Application workload
version
Helm chart version
appVersion
Elasticsearch version

File Purpose
Defines default configuration values for Elasticsearch.
Tested elasticsearch/values.yaml

image: docker.elastic.co/elasticsearch/elasticsearch:7.17.15
replicas: 1

security:
  elasticPassword: "Elastic@123"
  kibanaPassword: "kibana@123"
  fluentdPassword: "fluentd@123"

Explanation
Key
Description
image
Elasticsearch container image
replicas
Number of Elasticsearch nodes
security.*
Passwords for system users

Note: These values are injected into Kubernetes Secrets.


File Purpose
Stores Elasticsearch credentials securely.
Tested elasticsearch/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: elastic-credentials
  namespace: elastic
type: Opaque
stringData:
  elastic: {{ .Values.security.elasticPassword }}
  kibana_system: {{ .Values.security.kibanaPassword }}
  fluentd: {{ .Values.security.fluentdPassword }}




Explanation
Element
Purpose
Secret
Secure password storage
stringData
Plain-text input (auto-encoded)
Helm template
Pulls values from values.yaml

Prevents hardcoding credentials in Pods
Used by Elasticsearch, Kibana, Fluentd
File Purpose
Exposes Elasticsearch internally inside the cluster.
Tested elasticsearch/templates/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elastic
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  ports:
    - name: http
      protocol: TCP
      port: 9200
      targetPort: 9200
    - name: transport
      protocol: TCP
      port: 9300
      targetPort: 9300
  type: ClusterIP




Explanation
Port
Purpose
9200
    REST / HTTP API
9300
Node-to-node transport

Used by:
Fluentd → log ingestion
Kibana → search & visualization
File Purpose
Runs Elasticsearch as a stateful workload.
Tested elasticsearch/templates/statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elastic
spec:
  serviceName: elasticsearch
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: {{ .Values.image }}
        env:
        - name: discovery.type
          value: single-node
        - name: xpack.security.enabled
          value: "true"
        - name: ELASTIC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elastic-credentials
              key: elastic
        ports:
        - containerPort: 9200




kind: StatefulSet
replicas: {{ .Values.replicas }}
✔ Stable pod identity
✔ Supports persistent storage

Environment Variables
- name: discovery.type
  value: single-node
➡ Runs Elasticsearch in single-node mode
- name: xpack.security.enabled
  value: "true"
➡ Enables authentication & security

Password Injection
valueFrom:
  secretKeyRef:
    name: elastic-credentials
➡ Secure password loading at runtime
















Kibana Helm Chart
Directory Structure
charts/kibana/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
Purpose
Defines metadata for Kibana chart.
Tested kibana/Chart.yaml
apiVersion: v2
name: kibana
description: Kibana subchart for Elasticsearch visualization
type: application
version: 0.1.0
appVersion: "7.17.15"

Kibana version must match Elasticsearch version.
Tested  kibana/values.yaml

image: docker.elastic.co/kibana/kibana:7.17.15

elasticsearchHost: elasticsearch.elastic.svc.cluster.local
username: elastic
password: "Elastic@123"

ingress:
  enabled: true
  className: nginx
  host: efk-kibana.ssd-uat.opsmx.org
  path: /
  tls:
    enabled: false
    secretName: ""


Explanation
Field
Purpose
elasticsearchHost
ES service DNS
username/password
Authentication

Tested kibana/templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elastic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: {{ .Values.image }}
        env:
        - name: ELASTICSEARCH_HOSTS
          value: http://{{ .Values.elasticsearchHost }}:9200
        - name: ELASTICSEARCH_USERNAME
          value: {{ .Values.username }}
        - name: ELASTICSEARCH_PASSWORD
          value: {{ .Values.password }}
        ports:
        - containerPort: 5601






File Purpose
Runs Kibana as a stateless service.
Key Configurations
env:
- name: ELASTICSEARCH_HOSTS
➡ Tells Kibana where Elasticsearch is
- name: ELASTICSEARCH_USERNAME
- name: ELASTICSEARCH_PASSWORD
➡ Enables secure connection
2.4 templates/service.yaml
kind: Service
type: ClusterIP
port: 5601

Port
Purpose
5601
Kibana UI

Purpose
Exposes Kibana externally.
Tested kibana/templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: elastic
  annotations:
    kubernetes.io/ingress.class: {{ .Values.ingress.className | quote }}
spec:
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: Prefix
            backend:
              service:
                name: kibana
                port:
                  number: 5601
{{- end }}




host: kibana-efk.example.com
✔ Supports NGINX
✔ TLS can be enabled later
Fluentd Helm Chart
Directory Structure
charts/fluentd/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── serviceaccount.yaml
    ├── clusterrole.yaml
    ├── clusterrolebinding.yaml
    ├── configmap.yaml
    └── daemonset.yaml
Defines Fluentd logging agent metadata.
Tested fluentd/Chart.yaml
apiVersion: v2
name: fluentd
description: Fluentd subchart for Kubernetes log collection
type: application
version: 0.1.0
appVersion: "1.16"


Tested fluentd/values.yaml

fluentd:
  image: "fluent/fluentd-kubernetes-daemonset:v1.19.0-debian-elasticsearch7-1.1"
  elasticsearch:
    host: "elasticsearch.elastic.svc.cluster.local"
    port: 9200
    username: "elastic"
    password: "Elastic@123"
    indexPrefix: "ssd-k8s-logs"

Explanation
Defines:
ES destination
Index naming
Authentication
Tested fluentd/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: logging




Purpose
Provides identity for Fluentd Pods.
kind: ServiceAccount
Required for Kubernetes API access.
Purpose
Allows Fluentd to read Kubernetes metadata.
Tested fluentd/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
  - apiGroups: [""]
    resources:
      - namespaces
      - pods
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["apps"]
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch





resources:
- pods
- namespaces
➡ Used for log enrichment.
Purpose
Binds RBAC permissions to Fluentd service account.
Tested fluentd/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluentd
subjects:
  - kind: ServiceAccount
    name: fluentd
    namespace: logging


Tested fluentd/templates/configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      tag kubernetes.*
      read_from_head true
      pos_file /var/log/fluentd-containers.log.pos
      <parse>
        @type cri
      </parse>
    </source>

    <filter kubernetes.**>
      @type kubernetes_metadata
    </filter>

    <match kubernetes.**>
      @type elasticsearch
      host {{ .Values.fluentd.elasticsearch.host }}
      port {{ .Values.fluentd.elasticsearch.port }}
      user {{ .Values.fluentd.elasticsearch.username }}
      password {{ .Values.fluentd.elasticsearch.password }}
      logstash_format true
      logstash_prefix {{ .Values.fluentd.elasticsearch.indexPrefix }}
      include_tag_key true
      type_name _doc

      <buffer>
        @type file
        path /var/log/fluentd-buffers/es
        flush_interval 5s
      </buffer>
    </match>



Log Flow Explained
/var/log/containers/*.log
   ↓
Fluentd
   ↓
Elasticsearch
Key Sections
Section
Purpose
<source>
Reads container logs
<filter>
Adds Kubernetes metadata
<match>
Sends logs to Elasticsearch


Tested fluentd/templates/daemonset.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      priorityClassName: system-cluster-critical
      tolerations:
        - key: "node-role.kubernetes.io/master"
          operator: "Exists"
          effect: "NoSchedule"
        - key: "node.kubernetes.io/disk-pressure"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
      - name: fluentd
        image: {{ .Values.fluentd.image }}
        resources:
          requests:
            cpu: "100m"
            memory: "200Mi"
          limits:
            cpu: "500m"
            memory: "500Mi"
        volumeMounts:
          - name: varlog
            mountPath: /var/log
          - name: config
            mountPath: /fluentd/etc
          - name: fluentd-buffer
            mountPath: /var/log/fluentd-buffers
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
            type: Directory
        - name: config
          configMap:
            name: fluentd-config
        - name: fluentd-buffer
          hostPath:
            path: /mnt/fluentd-buffer
            type: DirectoryOrCreate




Why DaemonSet?
✔ One Fluentd pod per node
✔ Collects all node logs
Volume Mounts
/var/log
➡ Access to container logs on host.
End-to-End Data Flow Summary
Container Logs
   ↓
Fluentd DaemonSet
   ↓
Elasticsearch Indices
   ↓
Kibana Dashboards


Installation Steps (Step-by-Step)

Step 1:  Create Namespaces
kubectl create namespace elastic
kubectl create namespace logging

Step 2:  Update Helm Dependencies
cd efk-stack
helm dependency update

Step 3:  Install EFK Stack
helm install efk-stack . -n elastic

Verification Steps
Pod Status
kubectl get pods -n elastic
kubectl get pods -n logging
Expected:
Elasticsearch → Running
Kibana → Running
Fluentd → Running on all nodes
Test Elasticsearch
kubectl exec -it elasticsearch-0 -n elastic \
-- curl -u elastic:Elastic@123 http://localhost:9200/_cluster/health
Expected:
"status": "green"
Access Kibana
kubectl port-forward svc/kibana -n elastic 5601:5601
Open browser:
http://localhost:5601

13.4 Create Index Pattern
ssd-k8s-logs*
Common Operations
Upgrade
helm upgrade efk-stack . -n elastic


Uninstall
helm uninstall efk-stack -n elastic
Fluentd Logs
kubectl logs -n logging daemonset/fluentd


