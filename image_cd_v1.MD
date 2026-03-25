# Container Continuous Delivery (CD) Design Document

## Enterprise Multi-Tenant Container Deployment Platform

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Toolchain & Component Responsibilities](#3-toolchain--component-responsibilities)
4. [Common Helm Template Design](#4-common-helm-template-design)
5. [Values File & External Config Management](#5-values-file--external-config-management)
6. [Terraform Enterprise Wrapper Design](#6-terraform-enterprise-wrapper-design)
7. [XL Release Template Design](#7-xl-release-template-design)
8. [Jenkins Pipeline Design](#8-jenkins-pipeline-design)
9. [CD Scenario 1 — New App Image Deployment](#9-cd-scenario-1--new-app-image-deployment)
10. [CD Scenario 2 — App Image Update Deployment](#10-cd-scenario-2--app-image-update-deployment)
11. [CD Scenario 3 — Repave (Base Image Update)](#11-cd-scenario-3--repave-base-image-update)
12. [CD Scenario 4 — CD-Only Repave (Redeploy Existing Production Image)](#12-cd-scenario-4--cd-only-repave-redeploy-existing-production-image)
13. [Promotion Model & Gating Strategy](#13-promotion-model--gating-strategy)
14. [Rollback Strategy](#14-rollback-strategy)
15. [Security & Governance](#15-security--governance)
16. [Observability & Audit](#16-observability--audit)
17. [Appendices](#17-appendices)

---

## 1. Executive Summary

This document defines the Continuous Delivery architecture and operational procedures for deploying containerized applications across a multi-tenant enterprise Kubernetes platform. It covers four distinct deployment scenarios — new deployments, image updates, base-image repaves, and production redeployments — all unified under a single toolchain and common Helm template strategy.

### Design Principles

| Principle | Description |
|-----------|-------------|
| **Uniformity** | Every app uses a common Helm chart; variance is expressed only through values |
| **Separation of Concerns** | CI produces artifacts; CD consumes them. This document covers CD only |
| **Config Externalization** | No hardcoded config; all values sourced from an external KV config manager |
| **Auditability** | Every deployment is traceable from Jenkins trigger through to cluster state |
| **Repeatability** | Identical inputs always produce identical deployments (idempotent) |
| **Least Privilege** | Each tenant workspace scoped to its own namespace and secrets |

---

## 2. Architecture Overview

### 2.1 End-to-End Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CD PLATFORM ARCHITECTURE                          │
└─────────────────────────────────────────────────────────────────────────────┘

 ┌──────────┐       ┌──────────────┐      ┌─────────────────┐     ┌────────────┐
 │          │       │              │      │   Terraform     │     │            │
 │ Jenkins  │──────▶│  XL Release  │─────▶│   Enterprise    │────▶│ Kubernetes │
 │ Pipeline │ REST  │  (XLR)       │ API  │   Workspace     │Helm │  Cluster   │
 │          │  API  │              │      │                 │Prov.│            │
 └──────────┘       └──────┬───────┘      └───────┬─────────┘     └────────────┘
      │                    │                      │                      │
      │                    │                      │                      │
      ▼                    ▼                      ▼                      ▼
 ┌──────────┐       ┌──────────────┐      ┌─────────────────┐     ┌────────────┐
 │ Artifact │       │   Gate /     │      │  External KV    │     │  Container │
 │ Registry │       │   Approval   │      │  Config Manager │     │  Registry  │
 │ (params) │       │   Engine     │      │  (Consul/Vault) │     │  (Images)  │
 └──────────┘       └──────────────┘      └─────────────────┘     └────────────┘
```

### 2.2 Component Interaction Sequence

```
┌─────────┐    ┌─────┐    ┌──────┐    ┌──────────┐    ┌──────┐    ┌──────────┐
│ Trigger │    │Jenkins│   │ XLR  │    │Terraform │    │ Helm │    │   K8s    │
│ Source  │    │      │    │      │    │Enterprise│    │Provider   │  Cluster │
└────┬────┘    └──┬───┘    └──┬───┘    └────┬─────┘    └──┬───┘    └────┬─────┘
     │            │           │             │             │             │
     │  Event     │           │             │             │             │
     │───────────▶│           │             │             │             │
     │            │  Start    │             │             │             │
     │            │  Release  │             │             │             │
     │            │──────────▶│             │             │             │
     │            │           │  Trigger    │             │             │
     │            │           │  TFE Run    │             │             │
     │            │           │────────────▶│             │             │
     │            │           │             │  Fetch KV   │             │
     │            │           │             │  Configs    │             │
     │            │           │             │─────┐       │             │
     │            │           │             │◀────┘       │             │
     │            │           │             │  Render     │             │
     │            │           │             │  Values     │             │
     │            │           │             │─────┐       │             │
     │            │           │             │◀────┘       │             │
     │            │           │             │  helm_release             │
     │            │           │             │────────────▶│             │
     │            │           │             │             │  Apply      │
     │            │           │             │             │────────────▶│
     │            │           │             │             │  Status     │
     │            │           │             │             │◀────────────│
     │            │           │             │◀────────────│             │
     │            │           │◀────────────│             │             │
     │            │◀──────────│             │             │             │
     │◀───────────│           │             │             │             │
```

### 2.3 Environment Topology

```
┌─────────────────────────────────────────────────────┐
│                   Environment Ladder                 │
│                                                     │
│  ┌─────────┐    ┌─────────┐    ┌─────────────────┐ │
│  │   DEV   │───▶│   LLE   │───▶│      PROD       │ │
│  │         │    │         │    │                 │ │
│  │ Auto    │    │ Auto +  │    │ Manual Approval │ │
│  │ Deploy  │    │ Gate    │    │ + Change Ticket │ │
│  └─────────┘    └─────────┘    └─────────────────┘ │
│                                                     │
│  Each environment maps to:                          │
│  • Dedicated TFE Workspace                          │
│  • Dedicated K8s Namespace                          │
│  • Dedicated KV Config Path                         │
│  • Dedicated XLR Phase                              │
└─────────────────────────────────────────────────────┘
```

---

## 3. Toolchain & Component Responsibilities

### 3.1 Responsibility Matrix

| Component | Role | Owns | Does NOT Own |
|-----------|------|------|-------------|
| **Jenkins Pipeline** | Orchestration trigger & parameter collection | Triggering XLR, passing deployment parameters, CI/CD boundary | Deployment logic, environment config |
| **XL Release (XLR)** | Release orchestration, gating, approvals | Deployment order, environment promotion, audit trail, rollback decisions | Infrastructure provisioning, config storage |
| **Terraform Enterprise (TFE)** | Infrastructure-as-Code execution | Helm provider invocation, values rendering, state management | Application code, image building |
| **Helm (via TF Provider)** | Kubernetes manifest templating & deployment | K8s resource lifecycle, release management | Environment logic, approvals |
| **External KV Config Manager** | Externalized configuration storage | App config, secrets references, feature flags | Deployment orchestration |
| **Common Helm Template** | Standardized K8s manifest generation | Deployment, Service, HPA, ConfigMap, Ingress templates | App-specific business logic |
| **Container Registry** | Image storage and distribution | Image tags, vulnerability scan metadata | Image building (CI responsibility) |

### 3.2 Artifact & Data Flow Ownership

```
┌───────────────────────────────────────────────────────────────────┐
│                      Data Flow & Ownership                        │
│                                                                   │
│  CI Pipeline (out of scope)                                       │
│  └──▶ Produces: image tag, image digest, build metadata          │
│       Publishes to: Container Registry + Artifact Metadata Store  │
│                                                                   │
│  Jenkins CD Pipeline                                              │
│  └──▶ Consumes: image tag, app name, deploy scenario type        │
│       Passes to: XLR via REST API                                │
│                                                                   │
│  XLR                                                              │
│  └──▶ Consumes: deployment parameters from Jenkins               │
│       Enriches: environment-specific gate conditions              │
│       Passes to: TFE workspace via API trigger                   │
│                                                                   │
│  TFE Workspace                                                    │
│  └──▶ Consumes: TF variables (image, env, app)                  │
│       Fetches: KV config values at plan/apply time               │
│       Passes to: Helm provider with rendered values              │
│                                                                   │
│  Helm Provider                                                    │
│  └──▶ Consumes: common chart + rendered values.yaml              │
│       Applies to: Target K8s cluster/namespace                   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 4. Common Helm Template Design

### 4.1 Chart Structure

```
common-helm-chart/
├── Chart.yaml
├── values.yaml                    # Default/reference values
├── templates/
│   ├── _helpers.tpl               # Named templates & labels
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret-external.yaml       # ExternalSecret or CSI references
│   ├── serviceaccount.yaml
│   ├── networkpolicy.yaml
│   ├── poddisruptionbudget.yaml
│   ├── NOTES.txt
│   └── tests/
│       └── test-connection.yaml
└── README.md
```

### 4.2 Chart.yaml

```yaml
apiVersion: v2
name: common-app-chart
description: Enterprise common Helm chart for all containerized applications
type: application
version: 2.5.0          # Chart version - managed centrally
appVersion: \"0.0.0\"     # Overridden per deployment
maintainers:
  - name: Platform Engineering
    email: platform-eng@enterprise.com
```

### 4.3 Core Templates

#### templates/_helpers.tpl

```yaml
{{/*
Common labels applied to all resources
*/}}
{{- define \"common.labels\" -}}
app.kubernetes.io/name: {{ .Values.app.name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Values.app.imageTag | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/part-of: {{ .Values.app.portfolio | default \"unknown\" }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
enterprise.com/tenant: {{ .Values.tenant.name }}
enterprise.com/environment: {{ .Values.environment }}
enterprise.com/cost-center: {{ .Values.tenant.costCenter | default \"unassigned\" }}
enterprise.com/deploy-scenario: {{ .Values.deploy.scenario }}
enterprise.com/deploy-timestamp: {{ now | date \"2006-01-02T15-04-05\" | quote }}
{{- end }}

{{/*
Selector labels (immutable subset)
*/}}
{{- define \"common.selectorLabels\" -}}
app.kubernetes.io/name: {{ .Values.app.name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Full image reference
*/}}
{{- define \"common.image\" -}}
{{ .Values.app.imageRegistry }}/{{ .Values.app.imageRepository }}:{{ .Values.app.imageTag }}
{{- end }}

{{/*
Annotations for deployment tracking
*/}}
{{- define \"common.deployAnnotations\" -}}
enterprise.com/deployed-by: \"terraform-helm-provider\"
enterprise.com/deploy-scenario: {{ .Values.deploy.scenario | quote }}
enterprise.com/source-pipeline: {{ .Values.deploy.jenkinsBuildUrl | default \"manual\" | quote }}
enterprise.com/xlr-release-id: {{ .Values.deploy.xlrReleaseId | default \"unknown\" | quote }}
enterprise.com/tfe-run-id: {{ .Values.deploy.tfeRunId | default \"unknown\" | quote }}
{{- if eq .Values.deploy.scenario \"repave\" }}
enterprise.com/repave-reason: {{ .Values.deploy.repaveReason | default \"base-image-update\" | quote }}
enterprise.com/previous-image-tag: {{ .Values.deploy.previousImageTag | default \"unknown\" | quote }}
{{- end }}
{{- end }}
```

#### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include \"common.labels\" . | nindent 4 }}
  annotations:
    {{- include \"common.deployAnnotations\" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.app.replicas }}
  {{- end }}
  revisionHistoryLimit: {{ .Values.app.revisionHistoryLimit | default 5 }}
  selector:
    matchLabels:
      {{- include \"common.selectorLabels\" . | nindent 6 }}
  strategy:
    type: {{ .Values.app.deployStrategy | default \"RollingUpdate\" }}
    {{- if eq (.Values.app.deployStrategy | default \"RollingUpdate\") \"RollingUpdate\" }}
    rollingUpdate:
      maxUnavailable: {{ .Values.app.maxUnavailable | default \"25%\" }}
      maxSurge: {{ .Values.app.maxSurge | default \"25%\" }}
    {{- end }}
  template:
    metadata:
      labels:
        {{- include \"common.labels\" . | nindent 8 }}
      annotations:
        # Force rolling restart on config/secret change
        checksum/config: {{ include (print $.Template.BasePath \"/configmap.yaml\") . | sha256sum }}
        {{- include \"common.deployAnnotations\" . | nindent 8 }}
        {{- if .Values.deploy.forceRestart }}
        enterprise.com/restart-nonce: {{ randAlphaNum 10 | quote }}
        {{- end }}
    spec:
      serviceAccountName: {{ .Values.app.name }}
      terminationGracePeriodSeconds: {{ .Values.app.terminationGracePeriod | default 30 }}
      {{- if .Values.app.podSecurityContext }}
      securityContext:
        {{- toYaml .Values.app.podSecurityContext | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Values.app.name }}
          image: {{ include \"common.image\" . }}
          imagePullPolicy: {{ .Values.app.imagePullPolicy | default \"IfNotPresent\" }}
          ports:
            {{- range .Values.app.ports }}
            - name: {{ .name }}
              containerPort: {{ .containerPort }}
              protocol: {{ .protocol | default \"TCP\" }}
            {{- end }}
          {{- if .Values.app.healthCheck.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.app.healthCheck.livenessPath }}
              port: {{ .Values.app.healthCheck.port }}
            initialDelaySeconds: {{ .Values.app.healthCheck.livenessInitialDelay | default 30 }}
            periodSeconds: {{ .Values.app.healthCheck.livenessPeriod | default 10 }}
            failureThreshold: {{ .Values.app.healthCheck.livenessFailureThreshold | default 3 }}
          readinessProbe:
            httpGet:
              path: {{ .Values.app.healthCheck.readinessPath }}
              port: {{ .Values.app.healthCheck.port }}
            initialDelaySeconds: {{ .Values.app.healthCheck.readinessInitialDelay | default 5 }}
            periodSeconds: {{ .Values.app.healthCheck.readinessPeriod | default 5 }}
            failureThreshold: {{ .Values.app.healthCheck.readinessFailureThreshold | default 3 }}
          {{- if .Values.app.healthCheck.startupProbe }}
          startupProbe:
            httpGet:
              path: {{ .Values.app.healthCheck.startupPath | default .Values.app.healthCheck.readinessPath }}
              port: {{ .Values.app.healthCheck.port }}
            initialDelaySeconds: {{ .Values.app.healthCheck.startupInitialDelay | default 0 }}
            periodSeconds: {{ .Values.app.healthCheck.startupPeriod | default 10 }}
            failureThreshold: {{ .Values.app.healthCheck.startupFailureThreshold | default 30 }}
          {{- end }}
          {{- end }}
          resources:
            requests:
              cpu: {{ .Values.app.resources.requests.cpu }}
              memory: {{ .Values.app.resources.requests.memory }}
            limits:
              cpu: {{ .Values.app.resources.limits.cpu }}
              memory: {{ .Values.app.resources.limits.memory }}
          {{- if .Values.app.env }}
          env:
            {{- toYaml .Values.app.env | nindent 12 }}
          {{- end }}
          envFrom:
            - configMapRef:
                name: {{ .Values.app.name }}-config
          {{- if .Values.app.containerSecurityContext }}
          securityContext:
            {{- toYaml .Values.app.containerSecurityContext | nindent 12 }}
          {{- end }}
      {{- with .Values.app.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.app.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.app.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      imagePullSecrets:
        - name: {{ .Values.app.imagePullSecret | default \"enterprise-registry-cred\" }}
```

### 4.4 Default values.yaml (Reference Schema)

```yaml
# ============================================================
# COMMON HELM CHART - DEFAULT VALUES REFERENCE
# All values below are overridden by TFE-rendered values file
# ============================================================

# -- Deployment metadata
deploy:
  scenario: \"\"             # new-app | image-update | repave | cd-repave
  jenkinsBuildUrl: \"\"
  xlrReleaseId: \"\"
  tfeRunId: \"\"
  forceRestart: false      # true for cd-repave scenario
  previousImageTag: \"\"     # tracked for repave scenarios
  repaveReason: \"\"         # base-image-update | security-patch | compliance

# -- Environment
environment: \"\"            # dev | lle | prod
namespace: \"\"

# -- Tenant
tenant:
  name: \"\"
  costCenter: \"\"

# -- Application
app:
  name: \"\"
  portfolio: \"\"
  imageRegistry: \"enterprise-registry.com\"
  imageRepository: \"\"
  imageTag: \"\"
  imagePullPolicy: \"IfNotPresent\"
  imagePullSecret: \"enterprise-registry-cred\"
  replicas: 2
  revisionHistoryLimit: 5
  deployStrategy: \"RollingUpdate\"
  maxUnavailable: \"25%\"
  maxSurge: \"25%\"
  terminationGracePeriod: 30

  ports:
    - name: http
      containerPort: 8080

  healthCheck:
    enabled: true
    port: 8080
    livenessPath: \"/health/live\"
    readinessPath: \"/health/ready\"
    startupProbe: false
    livenessInitialDelay: 30
    readinessInitialDelay: 5
    livenessPeriod: 10
    readinessPeriod: 5

  resources:
    requests:
      cpu: \"250m\"
      memory: \"256Mi\"
    limits:
      cpu: \"1000m\"
      memory: \"1Gi\"

  env: []
  nodeSelector: {}
  tolerations: []
  affinity: {}

  podSecurityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000

  containerSecurityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop:
        - ALL

# -- Autoscaling
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# -- Service
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# -- Ingress
ingress:
  enabled: false
  className: \"nginx\"
  host: \"\"
  path: \"/\"
  pathType: Prefix
  tls:
    enabled: false
    secretName: \"\"

# -- Network Policy
networkPolicy:
  enabled: true
  ingressRules: []
  egressRules: []

# -- Pod Disruption Budget
pdb:
  enabled: true
  minAvailable: \"50%\"
```

---

## 5. Values File & External Config Management

### 5.1 KV Config Manager Structure

```
config-manager/
├── global/
│   ├── registry                     = \"enterprise-registry.com\"
│   ├── imagePullSecret              = \"enterprise-registry-cred\"
│   └── chartVersion                 = \"2.5.0\"
│
├── tenants/
│   └── {tenant-name}/
│       ├── costCenter               = \"CC-12345\"
│       └── portfolio                = \"digital-banking\"
│
├── apps/
│   └── {app-name}/
│       ├── common/
│       │   ├── imageRepository      = \"digital-banking/payment-svc\"
│       │   ├── healthCheckPath      = \"/actuator/health\"
│       │   ├── containerPort        = \"8080\"
│       │   └── terminationGracePeriod = \"45\"
│       │
│       ├── dev/
│       │   ├── imageTag             = \"1.5.0-rc.3\"
│       │   ├── replicas             = \"1\"
│       │   ├── cpu_request          = \"100m\"
│       │   ├── memory_request       = \"256Mi\"
│       │   ├── cpu_limit            = \"500m\"
│       │   ├── memory_limit         = \"512Mi\"
│       │   ├── ingressHost          = \"payment-svc.dev.enterprise.com\"
│       │   ├── hpa_enabled          = \"false\"
│       │   ├── LOG_LEVEL            = \"DEBUG\"
│       │   ├── DB_HOST              = \"dev-db.internal\"
│       │   └── FEATURE_FLAG_X       = \"true\"
│       │
│       ├── lle/
│       │   ├── imageTag             = \"1.4.0\"
│       │   ├── replicas             = \"2\"
│       │   ├── cpu_request          = \"250m\"
│       │   ├── memory_request       = \"512Mi\"
│       │   ├── cpu_limit            = \"1000m\"
│       │   ├── memory_limit         = \"1Gi\"
│       │   ├── ingressHost          = \"payment-svc.lle.enterprise.com\"
│       │   ├── hpa_enabled          = \"true\"
│       │   ├── hpa_min              = \"2\"
│       │   ├── hpa_max              = \"5\"
│       │   ├── LOG_LEVEL            = \"INFO\"
│       │   ├── DB_HOST              = \"lle-db.internal\"
│       │   └── FEATURE_FLAG_X       = \"false\"
│       │
│       └── prod/
│           ├── imageTag             = \"1.3.0\"
│           ├── replicas             = \"3\"
│           ├── cpu_request          = \"500m\"
│           ├── memory_request       = \"1Gi\"
│           ├── cpu_limit            = \"2000m\"
│           ├── memory_limit         = \"2Gi\"
│           ├── ingressHost          = \"payment-svc.enterprise.com\"
│           ├── hpa_enabled          = \"true\"
│           ├── hpa_min              = \"3\"
│           ├── hpa_max              = \"20\"
│           ├── LOG_LEVEL            = \"WARN\"
│           ├── DB_HOST              = \"prod-db.internal\"
│           └── FEATURE_FLAG_X       = \"false\"
```

### 5.2 Templated Values File

This template is stored in the TFE workspace repo and rendered at `terraform plan/apply` time using `templatefile()`.

```yaml
# ============================================================
# values-${environment}.yaml.tpl
# Rendered by Terraform templatefile() function
# Tokens resolved from External KV Config Manager
# ============================================================

deploy:
  scenario: \"${deploy_scenario}\"
  jenkinsBuildUrl: \"${jenkins_build_url}\"
  xlrReleaseId: \"${xlr_release_id}\"
  tfeRunId: \"${tfe_run_id}\"
  forceRestart: ${force_restart}
  previousImageTag: \"${previous_image_tag}\"
  repaveReason: \"${repave_reason}\"

environment: \"${environment}\"
namespace: \"${tenant_name}-${app_name}-${environment}\"

tenant:
  name: \"${tenant_name}\"
  costCenter: \"${cost_center}\"

app:
  name: \"${app_name}\"
  portfolio: \"${portfolio}\"
  imageRegistry: \"${image_registry}\"
  imageRepository: \"${image_repository}\"
  imageTag: \"${image_tag}\"
  imagePullPolicy: \"${image_pull_policy}\"
  replicas: ${replicas}

  ports:
    - name: http
      containerPort: ${container_port}

  healthCheck:
    enabled: true
    port: ${container_port}
    livenessPath: \"${health_check_path}\"
    readinessPath: \"${health_check_path}\"
    startupProbe: ${startup_probe_enabled}

  resources:
    requests:
      cpu: \"${cpu_request}\"
      memory: \"${memory_request}\"
    limits:
      cpu: \"${cpu_limit}\"
      memory: \"${memory_limit}\"

  env:
    - name: LOG_LEVEL
      value: \"${log_level}\"
    - name: DB_HOST
      value: \"${db_host}\"
    - name: FEATURE_FLAG_X
      value: \"${feature_flag_x}\"
    - name: DEPLOY_SCENARIO
      value: \"${deploy_scenario}\"

autoscaling:
  enabled: ${hpa_enabled}
  minReplicas: ${hpa_min}
  maxReplicas: ${hpa_max}
  targetCPUUtilizationPercentage: ${hpa_cpu_target}

service:
  type: ClusterIP
  port: 80
  targetPort: ${container_port}

ingress:
  enabled: true
  className: \"nginx\"
  host: \"${ingress_host}\"
  path: \"/\"
  pathType: Prefix
  tls:
    enabled: ${tls_enabled}
    secretName: \"${app_name}-tls\"

networkPolicy:
  enabled: true

pdb:
  enabled: true
  minAvailable: \"${pdb_min_available}\"
```

### 5.3 Config Resolution Hierarchy

```
┌──────────────────────────────────────────────────────────────┐
│              Configuration Resolution Order                   │
│              (Highest to Lowest Priority)                     │
│                                                              │
│  1. Deploy-time override parameters (from Jenkins/XLR)       │
│     e.g., image_tag override for repave scenarios            │
│                                                              │
│  2. Environment-specific KV path                             │
│     apps/{app-name}/{env}/*                                  │
│                                                              │
│  3. Application common KV path                               │
│     apps/{app-name}/common/*                                 │
│                                                              │
│  4. Tenant-level KV path                                     │
│     tenants/{tenant-name}/*                                  │
│                                                              │
│  5. Global defaults KV path                                  │
│     global/*                                                 │
│                                                              │
│  6. Helm chart default values.yaml                           │
│     (Lowest priority - safety net only)                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Terraform Enterprise Wrapper Design

### 6.1 TFE Workspace Layout

```
Workspace Naming Convention:
  cd-{tenant}-{app-name}-{environment}

Examples:
  cd-digital-banking-payment-svc-dev
  cd-digital-banking-payment-svc-lle
  cd-digital-banking-payment-svc-prod
```

### 6.2 Terraform Code Structure

```
terraform-cd-wrapper/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── data.tf                  # KV config lookups
├── locals.tf                # Config merge logic
├── templates/
│   └── values.yaml.tpl     # Templated Helm values
├── versions.tf
└── README.md
```

### 6.3 providers.tf

```hcl
terraform {
  required_version = \">= 1.5.0\"

  required_providers {
    helm = {
      source  = \"hashicorp/helm\"
      version = \"~> 2.12\"
    }
    consul = {
      source  = \"hashicorp/consul\"
      version = \"~> 2.20\"
    }
    kubernetes = {
      source  = \"hashicorp/kubernetes\"
      version = \"~> 2.25\"
    }
  }

  cloud {
    organization = \"enterprise-org\"
    # Workspace set via TFE variable or CLI
  }
}

provider \"helm\" {
  kubernetes {
    host                   = var.k8s_cluster_endpoint
    cluster_ca_certificate = base64decode(var.k8s_cluster_ca)
    token                  = var.k8s_auth_token
  }
}

provider \"kubernetes\" {
  host                   = var.k8s_cluster_endpoint
  cluster_ca_certificate = base64decode(var.k8s_cluster_ca)
  token                  = var.k8s_auth_token
}

provider \"consul\" {
  address    = var.consul_address
  token      = var.consul_token
  datacenter = var.consul_datacenter
}
```

### 6.4 variables.tf

```hcl
# ============================================================
# Variables passed from XLR via TFE API
# ============================================================

variable \"app_name\" {
  description = \"Application name\"
  type        = string
}

variable \"tenant_name\" {
  description = \"Tenant/team name\"
  type        = string
}

variable \"environment\" {
  description = \"Target environment: dev, lle, prod\"
  type        = string
  validation {
    condition     = contains([\"dev\", \"lle\", \"prod\"], var.environment)
    error_message = \"Environment must be dev, lle, or prod.\"
  }
}

variable \"deploy_scenario\" {
  description = \"Deployment scenario type\"
  type        = string
  validation {
    condition     = contains([\"new-app\", \"image-update\", \"repave\", \"cd-repave\"], var.deploy_scenario)
    error_message = \"Deploy scenario must be: new-app, image-update, repave, or cd-repave.\"
  }
}

variable \"image_tag\" {
  description = \"Container image tag to deploy\"
  type        = string
}

variable \"image_tag_override\" {
  description = \"If set, overrides KV-stored image tag (used for repave scenarios)\"
  type        = string
  default     = \"\"
}

variable \"force_restart\" {
  description = \"Force pod restart even if image tag unchanged (cd-repave)\"
  type        = bool
  default     = false
}

variable \"previous_image_tag\" {
  description = \"Previous image tag for audit trail\"
  type        = string
  default     = \"\"
}

variable \"repave_reason\" {
  description = \"Reason for repave deployment\"
  type        = string
  default     = \"\"
}

variable \"jenkins_build_url\" {
  description = \"URL of triggering Jenkins build\"
  type        = string
  default     = \"\"
}

variable \"xlr_release_id\" {
  description = \"XL Release release ID\"
  type        = string
  default     = \"\"
}

variable \"chart_version\" {
  description = \"Override for common helm chart version\"
  type        = string
  default     = \"\"
}

# ============================================================
# Infrastructure / Connectivity (set as workspace variables)
# ============================================================

variable \"k8s_cluster_endpoint\" {
  type      = string
  sensitive = true
}

variable \"k8s_cluster_ca\" {
  type      = string
  sensitive = true
}

variable \"k8s_auth_token\" {
  type      = string
  sensitive = true
}

variable \"consul_address\" {
  type = string
}

variable \"consul_token\" {
  type      = string
  sensitive = true
}

variable \"consul_datacenter\" {
  type    = string
  default = \"dc1\"
}

variable \"helm_chart_repo\" {
  description = \"Helm chart repository URL\"
  type        = string
  default     = \"https://charts.enterprise.com\"
}
```

### 6.5 data.tf — External KV Lookups

```hcl
# ============================================================
# Fetch configuration from external KV store (Consul)
# Merge hierarchy: global < tenant < app-common < app-env
# ============================================================

# Global defaults
data \"consul_keys\" \"global\" {
  key {
    name    = \"registry\"
    path    = \"global/registry\"
    default = \"enterprise-registry.com\"
  }
  key {
    name    = \"image_pull_secret\"
    path    = \"global/imagePullSecret\"
    default = \"enterprise-registry-cred\"
  }
  key {
    name    = \"chart_version\"
    path    = \"global/chartVersion\"
    default = \"2.5.0\"
  }
}

# Tenant defaults
data \"consul_keys\" \"tenant\" {
  key {
    name    = \"cost_center\"
    path    = \"tenants/${var.tenant_name}/costCenter\"
    default = \"unassigned\"
  }
  key {
    name    = \"portfolio\"
    path    = \"tenants/${var.tenant_name}/portfolio\"
    default = \"unknown\"
  }
}

# App common config
data \"consul_keys\" \"app_common\" {
  key {
    name    = \"image_repository\"
    path    = \"apps/${var.app_name}/common/imageRepository\"
  }
  key {
    name    = \"container_port\"
    path    = \"apps/${var.app_name}/common/containerPort\"
    default = \"8080\"
  }
  key {
    name    = \"health_check_path\"
    path    = \"apps/${var.app_name}/common/healthCheckPath\"
    default = \"/health\"
  }
  key {
    name    = \"termination_grace_period\"
    path    = \"apps/${var.app_name}/common/terminationGracePeriod\"
    default = \"30\"
  }
}

# Environment-specific config
data \"consul_keys\" \"app_env\" {
  key {
    name    = \"kv_image_tag\"
    path    = \"apps/${var.app_name}/${var.environment}/imageTag\"
    default = \"\"
  }
  key {
    name    = \"replicas\"
    path    = \"apps/${var.app_name}/${var.environment}/replicas\"
    default = \"2\"
  }
  key {
    name    = \"cpu_request\"
    path    = \"apps/${var.app_name}/${var.environment}/cpu_request\"
    default = \"250m\"
  }
  key {
    name    = \"memory_request\"
    path    = \"apps/${var.app_name}/${var.environment}/memory_request\"
    default = \"256Mi\"
  }
  key {
    name    = \"cpu_limit\"
    path    = \"apps/${var.app_name}/${var.environment}/cpu_limit\"
    default = \"1000m\"
  }
  key {
    name    = \"memory_limit\"
    path    = \"apps/${var.app_name}/${var.environment}/memory_limit\"
    default = \"1Gi\"
  }
  key {
    name    = \"ingress_host\"
    path    = \"apps/${var.app_name}/${var.environment}/ingressHost\"
  }
  key {
    name    = \"hpa_enabled\"
    path    = \"apps/${var.app_name}/${var.environment}/hpa_enabled\"
    default = \"false\"
  }
  key {
    name    = \"hpa_min\"
    path    = \"apps/${var.app_name}/${var.environment}/hpa_min\"
    default = \"2\"
  }
  key {
    name    = \"hpa_max\"
    path    = \"apps/${var.app_name}/${var.environment}/hpa_max\"
    default = \"10\"
  }
  key {
    name    = \"log_level\"
    path    = \"apps/${var.app_name}/${var.environment}/LOG_LEVEL\"
    default = \"INFO\"
  }
  key {
    name    = \"db_host\"
    path    = \"apps/${var.app_name}/${var.environment}/DB_HOST\"
  }
  key {
    name    = \"feature_flag_x\"
    path    = \"apps/${var.app_name}/${var.environment}/FEATURE_FLAG_X\"
    default = \"false\"
  }
}
```

### 6.6 locals.tf — Config Merge & Image Resolution

```hcl
locals {
  # ============================================================
  # Image tag resolution logic per deploy scenario
  # ============================================================
  resolved_image_tag = coalesce(
    # Priority 1: Explicit override from pipeline (repave scenarios)
    var.image_tag_override,
    # Priority 2: Pipeline-passed image tag (new-app, image-update)
    var.image_tag,
    # Priority 3: KV-stored current tag (cd-repave fallback)
    data.consul_keys.app_env.var.kv_image_tag,
  )

  # Determine image pull policy per scenario
  image_pull_policy = var.deploy_scenario == \"cd-repave\" ? \"Always\" : \"IfNotPresent\"

  # Force restart flag - critical for cd-repave
  force_restart = var.deploy_scenario == \"cd-repave\" ? true : var.force_restart

  # Chart version resolution
  chart_version = coalesce(var.chart_version, data.consul_keys.global.var.chart_version)

  # Namespace construction
  namespace = \"${var.tenant_name}-${var.app_name}-${var.environment}\"

  # Rendered values file
  rendered_values = templatefile(\"${path.module}/templates/values.yaml.tpl\", {
    # Deploy metadata
    deploy_scenario    = var.deploy_scenario
    jenkins_build_url  = var.jenkins_build_url
    xlr_release_id     = var.xlr_release_id
    tfe_run_id         = terraform.workspace
    force_restart      = local.force_restart
    previous_image_tag = var.previous_image_tag
    repave_reason      = var.repave_reason

    # Environment
    environment = var.environment
    tenant_name = var.tenant_name

    # Tenant
    cost_center = data.consul_keys.tenant.var.cost_center
    portfolio   = data.consul_keys.tenant.var.portfolio

    # App image
    app_name         = var.app_name
    image_registry   = data.consul_keys.global.var.registry
    image_repository = data.consul_keys.app_common.var.image_repository
    image_tag        = local.resolved_image_tag
    image_pull_policy = local.image_pull_policy

    # App config
    container_port           = data.consul_keys.app_common.var.container_port
    health_check_path        = data.consul_keys.app_common.var.health_check_path
    startup_probe_enabled    = var.environment == \"prod\" ? \"true\" : \"false\"
    replicas                 = data.consul_keys.app_env.var.replicas

    # Resources
    cpu_request    = data.consul_keys.app_env.var.cpu_request
    memory_request = data.consul_keys.app_env.var.memory_request
    cpu_limit      = data.consul_keys.app_env.var.cpu_limit
    memory_limit   = data.consul_keys.app_env.var.memory_limit

    # Autoscaling
    hpa_enabled    = data.consul_keys.app_env.var.hpa_enabled
    hpa_min        = data.consul_keys.app_env.var.hpa_min
    hpa_max        = data.consul_keys.app_env.var.hpa_max
    hpa_cpu_target = var.environment == \"prod\" ? \"60\" : \"70\"

    # Networking
    ingress_host     = data.consul_keys.app_env.var.ingress_host
    tls_enabled      = var.environment == \"prod\" ? \"true\" : \"false\"
    pdb_min_available = var.environment == \"prod\" ? \"60%\" : \"50%\"

    # App env vars
    log_level      = data.consul_keys.app_env.var.log_level
    db_host        = data.consul_keys.app_env.var.db_host
    feature_flag_x = data.consul_keys.app_env.var.feature_flag_x
  })
}
```

### 6.7 main.tf — Helm Release

```hcl
# ============================================================
# Ensure namespace exists
# ============================================================
resource \"kubernetes_namespace\" \"app_namespace\" {
  count = var.deploy_scenario == \"new-app\" ? 1 : 0

  metadata {
    name = local.namespace
    labels = {
      \"app.kubernetes.io/managed-by\" = \"terraform\"
      \"enterprise.com/tenant\"        = var.tenant_name
      \"enterprise.com/environment\"   = var.environment
    }
  }
}

# ============================================================
# Helm Release - Core Deployment Resource
# ============================================================
resource \"helm_release\" \"app\" {
  name             = var.app_name
  namespace        = local.namespace
  create_namespace = false # Managed above or pre-provisioned

  repository = var.helm_chart_repo
  chart      = \"common-app-chart\"
  version    = local.chart_version

  values = [local.rendered_values]

  # Deployment behavior
  wait             = true
  wait_for_jobs    = true
  timeout          = var.environment == \"prod\" ? 600 : 300
  atomic           = true    # Auto-rollback on failure
  cleanup_on_fail  = true
  force_update     = local.force_restart
  recreate_pods    = var.deploy_scenario == \"cd-repave\" ? true : false
  replace          = false

  # Lifecycle
  lint = true

  # Dependency on namespace for new apps
  depends_on = [kubernetes_namespace.app_namespace]
}

# ============================================================
# Post-deploy: Update KV with deployed image tag
# ============================================================
resource \"consul_key_prefix\" \"deployed_state\" {
  path_prefix = \"apps/${var.app_name}/${var.environment}/deployed/\"

  subkeys = {
    \"imageTag\"        = local.resolved_image_tag
    \"deployScenario\"  = var.deploy_scenario
    \"deployTimestamp\"  = timestamp()
    \"helmRevision\"    = tostring(helm_release.app.metadata[0].revision)
    \"tfeRunId\"        = terraform.workspace
    \"xlrReleaseId\"    = var.xlr_release_id
    \"jenkinsBuildUrl\" = var.jenkins_build_url
  }
}
```

### 6.8 outputs.tf

```hcl
output \"release_name\" {
  value = helm_release.app.name
}

output \"release_namespace\" {
  value = helm_release.app.namespace
}

output \"release_revision\" {
  value = helm_release.app.metadata[0].revision
}

output \"release_status\" {
  value = helm_release.app.status
}

output \"deployed_image_tag\" {
  value = local.resolved_image_tag
}

output \"deploy_scenario\" {
  value = var.deploy_scenario
}

output \"chart_version\" {
  value = local.chart_version
}

output \"force_restart_applied\" {
  value = local.force_restart
}
```

---

## 7. XL Release Template Design

### 7.1 Template Hierarchy

```
XLR Templates/
├── Container-CD-Master/
│   ├── Dev-Deployment-Template
│   ├── LLE-Deployment-Template
│   └── Prod-Deployment-Template
│
├── Template per scenario type (parameterized):
│   ├── new-app
│   ├── image-update
│   ├── repave
│   └── cd-repave
```

### 7.2 Master Release Template (All Environments)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     XLR MASTER RELEASE TEMPLATE                             │
│                Container-CD-${app_name}-${deploy_scenario}                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PHASE 1: VALIDATION                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 1.1 Validate Input Parameters                                       │     │
│  │     - app_name, tenant_name, image_tag, deploy_scenario present     │     │
│  │     - image_tag format matches semver or digest pattern             │     │
│  │                                                                     │     │
│  │ 1.2 Image Validation (skip for cd-repave)                          │     │
│  │     - Verify image exists in container registry                     │     │
│  │     - Verify image vulnerability scan passed                        │     │
│  │     - Verify image is signed (cosign/notation)                      │     │
│  │                                                                     │     │
│  │ 1.3 Config Validation                                               │     │
│  │     - Verify KV config entries exist for target environments        │     │
│  │     - Dry-run template rendering                                    │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  PHASE 2: DEV DEPLOYMENT                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 2.1 [Gate] Pre-deploy check: DEV cluster health                     │     │
│  │                                                                     │     │
│  │ 2.2 [KV Update] Update DEV imageTag in KV store                    │     │
│  │     (for new-app and image-update scenarios only)                   │     │
│  │                                                                     │     │
│  │ 2.3 [TFE Trigger] Trigger TFE workspace: cd-{tenant}-{app}-dev    │     │
│  │     Variables:                                                      │     │
│  │       app_name         = ${app_name}                                │     │
│  │       tenant_name      = ${tenant_name}                             │     │
│  │       environment      = \"dev\"                                      │     │
│  │       deploy_scenario  = ${deploy_scenario}                         │     │
│  │       image_tag        = ${image_tag}                               │     │
│  │       force_restart    = ${force_restart}                           │     │
│  │       xlr_release_id   = ${release.id}                              │     │
│  │       jenkins_build_url = ${jenkins_build_url}                      │     │
│  │                                                                     │     │
│  │ 2.4 [Wait] Wait for TFE run to complete (poll status)              │     │
│  │                                                                     │     │
│  │ 2.5 [Verify] Post-deploy health check                              │     │
│  │     - HTTP readiness endpoint responds 200                          │     │
│  │     - Pod count matches desired replicas                            │     │
│  │     - No CrashLoopBackOff pods                                      │     │
│  │                                                                     │     │
│  │ 2.6 [Automated Test] Trigger DEV smoke tests                       │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  PHASE 3: LLE DEPLOYMENT                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 3.1 [Gate] Auto-gate: DEV deployment successful                     │     │
│  │     + DEV smoke tests passed                                        │     │
│  │     (cd-repave: skip test gate, proceed on deploy success only)     │     │
│  │                                                                     │     │
│  │ 3.2 [Gate] Pre-deploy check: LLE cluster health                    │     │
│  │                                                                     │     │
│  │ 3.3 [KV Update] Update LLE imageTag in KV store                   │     │
│  │     (for new-app and image-update scenarios only)                   │     │
│  │                                                                     │     │
│  │ 3.4 [TFE Trigger] Trigger TFE workspace: cd-{tenant}-{app}-lle    │     │
│  │     (Same variable pattern as DEV, environment = \"lle\")             │     │
│  │                                                                     │     │
│  │ 3.5 [Wait] Wait for TFE run completion                             │     │
│  │                                                                     │     │
│  │ 3.6 [Verify] Post-deploy health check                              │     │
│  │                                                                     │     │
│  │ 3.7 [Automated Test] Trigger LLE integration/regression tests      │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  PHASE 4: PROD DEPLOYMENT                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 4.1 [Gate] Auto-gate: LLE deployment + tests successful            │     │
│  │                                                                     │     │
│  │ 4.2 [Manual Gate] Change Advisory Board (CAB) approval             │     │
│  │     - Change ticket number required                                 │     │
│  │     - Approver: Release Manager + App Owner                         │     │
│  │     (cd-repave: Platform team approval only, expedited)             │     │
│  │                                                                     │     │
│  │ 4.3 [Gate] Pre-deploy check: PROD cluster health                   │     │
│  │     + Deployment window validation (business hours check)           │     │
│  │                                                                     │     │
│  │ 4.4 [KV Update] Update PROD imageTag in KV store                  │     │
│  │     (for new-app and image-update scenarios only)                   │     │
│  │                                                                     │     │
│  │ 4.5 [Snapshot] Capture current PROD state                          │     │
│  │     - Current image tag, replica count, helm revision               │     │
│  │     - Store in KV: apps/{app}/prod/rollback/*                       │     │
│  │                                                                     │     │
│  │ 4.6 [TFE Trigger] Trigger TFE workspace: cd-{tenant}-{app}-prod   │     │
│  │     (Same variable pattern, environment = \"prod\")                   │     │
│  │                                                                     │     │
│  │ 4.7 [Wait] Wait for TFE run completion                             │     │
│  │                                                                     │     │
│  │ 4.8 [Verify] Post-deploy health check                              │     │
│  │     - Extended validation period (5 min stability window)           │     │
│  │     - Canary metrics check (error rate, latency p99)                │     │
│  │                                                                     │     │
│  │ 4.9 [Automated Test] Trigger PROD smoke tests                      │     │
│  │                                                                     │     │
│  │ 4.10 [Notification] Send deployment completion notification         │     │
│  │      - Slack, Email, ServiceNow update                              │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  PHASE 5: ROLLBACK (Conditional - only on failure)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │ 5.1 [Auto-trigger] If Phase 4.8 or 4.9 fails                       │     │
│  │                                                                     │     │
│  │ 5.2 [TFE Trigger] Re-trigger TFE with rollback snapshot values     │     │
│  │     image_tag = ${rollback_image_tag}                               │     │
│  │     deploy_scenario = \"rollback\"                                    │     │
│  │                                                                     │     │
│  │ 5.3 [KV Restore] Restore KV imageTag to previous value             │     │
│  │                                                                     │     │
│  │ 5.4 [Verify] Confirm rollback health                               │     │
│  │                                                                     │     │
│  │ 5.5 [Notification] Alert on rollback event (PagerDuty + Slack)      │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 XLR Template Variables

| Variable | Type | Source | Required |
|----------|------|--------|----------|
| `app_name` | String | Jenkins | Yes |
| `tenant_name` | String | Jenkins | Yes |
| `image_tag` | String | Jenkins | Yes (except cd-repave) |
| `deploy_scenario` | Enum | Jenkins | Yes |
| `force_restart` | Boolean | Jenkins | No (default: false) |
| `previous_image_tag` | String | Jenkins/KV | No |
| `repave_reason` | String | Jenkins | No |
| `jenkins_build_url` | String | Jenkins | Yes |
| `change_ticket` | String | Manual Gate | Prod only |
| `image_tag_override` | String | Jenkins | Repave scenarios |
| `target_environments` | List | Jenkins | No (default: all) |

---

## 8. Jenkins Pipeline Design

### 8.1 Shared Pipeline Library

```groovy
// vars/containerCD.groovy
// Shared library entry point for all CD scenarios

def call(Map params) {
    pipeline {
        agent { label 'cd-agent' }

        parameters {
            string(name: 'APP_NAME', description: 'Application name')
            string(name: 'TENANT_NAME', description: 'Tenant/team name')
            choice(name: 'DEPLOY_SCENARIO', choices: ['new-app', 'image-update', 'repave', 'cd-repave'],
                   description: 'Deployment scenario')
            string(name: 'IMAGE_TAG', defaultValue: '',
                   description: 'Image tag to deploy (leave empty for cd-repave)')
            string(name: 'IMAGE_TAG_OVERRIDE', defaultValue: '',
                   description: 'Override tag for repave scenarios')
            booleanParam(name: 'FORCE_RESTART', defaultValue: false,
                         description: 'Force pod restart')
            string(name: 'REPAVE_REASON', defaultValue: '',
                   description: 'Reason for repave (base-image-update, security-patch)')
            string(name: 'TARGET_ENVIRONMENTS', defaultValue: 'dev,lle,prod',
                   description: 'Comma-separated target environments')
            string(name: 'CHANGE_TICKET', defaultValue: '',
                   description: 'Change ticket number for prod (populated at approval)')
        }

        environment {
            XLR_URL       = credentials('xlr-url')
            XLR_CRED      = credentials('xlr-api-credentials')
            REGISTRY_URL  = 'enterprise-registry.com'
        }

        stages {
            stage('Initialize & Validate') {
                steps {
                    script {
                        validateInputParameters()
                        env.DEPLOY_ID = \"${params.APP_NAME}-${params.DEPLOY_SCENARIO}-${BUILD_NUMBER}\"
                        currentBuild.displayName = \"#${BUILD_NUMBER}-${params.APP_NAME}-${params.DEPLOY_SCENARIO}\"
                        currentBuild.description = \"Image: ${resolveImageTag()}\"
                    }
                }
            }

            stage('Pre-flight Checks') {
                parallel {
                    stage('Verify Image') {
                        when {
                            expression { params.DEPLOY_SCENARIO != 'cd-repave' }
                        }
                        steps {
                            script {
                                verifyImageExists(
                                    registry: env.REGISTRY_URL,
                                    appName: params.APP_NAME,
                                    imageTag: resolveImageTag()
                                )
                                verifyImageScan(
                                    registry: env.REGISTRY_URL,
                                    appName: params.APP_NAME,
                                    imageTag: resolveImageTag()
                                )
                            }
                        }
                    }
                    stage('Verify Config') {
                        steps {
                            script {
                                verifyKVConfig(
                                    appName: params.APP_NAME,
                                    environments: params.TARGET_ENVIRONMENTS.split(',')
                                )
                            }
                        }
                    }
                    stage('Capture Current State') {
                        steps {
                            script {
                                env.PREVIOUS_IMAGE_TAG = getCurrentDeployedTag(
                                    appName: params.APP_NAME,
                                    environment: 'prod'
                                )
                            }
                        }
                    }
                }
            }

            stage('Trigger XL Release') {
                steps {
                    script {
                        def xlrTemplateId = resolveXLRTemplate(params.DEPLOY_SCENARIO)

                        def xlrVariables = [
                            'app_name'          : params.APP_NAME,
                            'tenant_name'       : params.TENANT_NAME,
                            'image_tag'         : resolveImageTag(),
                            'deploy_scenario'   : params.DEPLOY_SCENARIO,
                            'force_restart'     : params.DEPLOY_SCENARIO == 'cd-repave' ? 'true' : params.FORCE_RESTART.toString(),
                            'previous_image_tag': env.PREVIOUS_IMAGE_TAG ?: '',
                            'repave_reason'     : params.REPAVE_REASON,
                            'jenkins_build_url' : env.BUILD_URL,
                            'image_tag_override': params.IMAGE_TAG_OVERRIDE,
                            'target_environments': params.TARGET_ENVIRONMENTS
                        ]

                        env.XLR_RELEASE_ID = triggerXLRelease(
                            templateId: xlrTemplateId,
                            releaseTitle: \"CD-${env.DEPLOY_ID}\",
                            variables: xlrVariables
                        )

                        echo \"XLR Release triggered: ${env.XLR_RELEASE_ID}\"
                    }
                }
            }

            stage('Monitor XLR Release') {
                steps {
                    script {
                        monitorXLRelease(
                            releaseId: env.XLR_RELEASE_ID,
                            timeout: 120,         // minutes
                            pollInterval: 30      // seconds
                        )
                    }
                }
            }
        }

        post {
            success {
                script {
                    notifyDeployment(
                        status: 'SUCCESS',
                        appName: params.APP_NAME,
                        scenario: params.DEPLOY_SCENARIO,
                        imageTag: resolveImageTag(),
                        xlrReleaseId: env.XLR_RELEASE_ID
                    )
                }
            }
            failure {
                script {
                    notifyDeployment(
                        status: 'FAILURE',
                        appName: params.APP_NAME,
                        scenario: params.DEPLOY_SCENARIO,
                        imageTag: resolveImageTag(),
                        xlrReleaseId: env.XLR_RELEASE_ID
                    )
                }
            }
            always {
                archiveArtifacts artifacts: 'deploy-*.log', allowEmptyArchive: true
            }
        }
    }
}

// ============================================================
// Helper Functions
// ============================================================

def resolveImageTag() {
    if (params.IMAGE_TAG_OVERRIDE?.trim()) {
        return params.IMAGE_TAG_OVERRIDE
    }
    if (params.IMAGE_TAG?.trim()) {
        return params.IMAGE_TAG
    }
    // For cd-repave: fetch current prod tag from KV
    if (params.DEPLOY_SCENARIO == 'cd-repave') {
        return getCurrentDeployedTag(
            appName: params.APP_NAME,
            environment: 'prod'
        )
    }
    error(\"IMAGE_TAG is required for ${params.DEPLOY_SCENARIO} scenario\")
}

def resolveXLRTemplate(String scenario) {
    def templateMap = [
        'new-app'      : 'Applications/Container-CD/New-App-Deployment',
        'image-update'  : 'Applications/Container-CD/Image-Update-Deployment',
        'repave'        : 'Applications/Container-CD/Repave-Deployment',
        'cd-repave'     : 'Applications/Container-CD/CD-Repave-Deployment'
    ]
    return templateMap[scenario] ?: error(\"Unknown scenario: ${scenario}\")
}

def validateInputParameters() {
    assert params.APP_NAME?.trim() : \"APP_NAME is required\"
    assert params.TENANT_NAME?.trim() : \"TENANT_NAME is required\"
    assert params.DEPLOY_SCENARIO?.trim() : \"DEPLOY_SCENARIO is required\"

    if (params.DEPLOY_SCENARIO == 'new-app' || params.DEPLOY_SCENARIO == 'image-update') {
        assert params.IMAGE_TAG?.trim() : \"IMAGE_TAG required for ${params.DEPLOY_SCENARIO}\"
    }

    if (params.DEPLOY_SCENARIO == 'repave') {
        assert params.IMAGE_TAG?.trim() || params.IMAGE_TAG_OVERRIDE?.trim() :
            \"IMAGE_TAG or IMAGE_TAG_OVERRIDE required for repave\"
        assert params.REPAVE_REASON?.trim() : \"REPAVE_REASON required for repave\"
    }
}
```

### 8.2 XLR Trigger Helper

```groovy
// vars/triggerXLRelease.groovy

def call(Map params) {
    def payload = [
        releaseTitle: params.releaseTitle,
        variables: params.variables.collect { k, v ->
            [key: k, value: v, type: 'xlrelease.StringVariable']
        },
        autoStart: true
    ]

    def response = httpRequest(
        url: \"${env.XLR_URL}/api/v1/templates/${params.templateId}/create\",
        httpMode: 'POST',
        contentType: 'APPLICATION_JSON',
        requestBody: groovy.json.JsonOutput.toJson(payload),
        authentication: 'xlr-api-credentials',
        validResponseCodes: '200:299'
    )

    def result = readJSON text: response.content
    return result.id
}
```

---

## 9. CD Scenario 1 — New App Image Deployment

### 9.1 Overview

First-time deployment of a containerized application to the platform. This scenario creates all necessary Kubernetes resources from scratch.

### 9.2 Prerequisites

| Prerequisite | Owner | Verification |
|-------------|-------|-------------|
| App onboarded to platform registry | App Team + Platform | Container image exists in registry |
| KV config populated for all environments | App Team | Config validation script passes |
| TFE workspaces provisioned | Platform Team | `cd-{tenant}-{app}-{env}` exists for dev/lle/prod |
| K8s namespaces pre-allocated (optional) | Platform Team | Namespace exists OR TF creates it |
| Tenant RBAC configured | Platform Team | ServiceAccount and RoleBindings exist |
| Ingress DNS entries configured | Network Team | DNS records point to cluster ingress |
| Image vulnerability scan passed | CI Pipeline | Scan report in registry metadata |

### 9.3 Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                 SCENARIO 1: NEW APP DEPLOYMENT                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  App Team                                                           │
│  ├── Populates KV config for dev/lle/prod                          │
│  ├── CI pipeline builds image → registry (tag: 1.0.0)             │
│  └── Triggers Jenkins CD pipeline                                  │
│       │                                                             │
│       ▼                                                             │
│  Jenkins Pipeline                                                   │
│  ├── DEPLOY_SCENARIO = \"new-app\"                                   │
│  ├── IMAGE_TAG = \"1.0.0\"                                           │
│  ├── Validates: image exists, scan passed, KV populated            │
│  └── Triggers XLR with params                                      │
│       │                                                             │
│       ▼                                                             │
│  XLR Release                                                        │
│  ├── Phase: Validation ✓                                           │
│  ├── Phase: DEV                                                     │
│  │   ├── Updates KV: apps/{app}/dev/imageTag = \"1.0.0\"            │
│  │   ├── Triggers TFE: cd-{tenant}-{app}-dev                      │
│  │   │   └── TFE:                                                   │
│  │   │       ├── Fetches KV config (global → tenant → app → env)   │
│  │   │       ├── Renders values.yaml from template                 │
│  │   │       ├── Creates namespace (new-app scenario)              │
│  │   │       └── helm_release.app → K8s (creates all resources)    │
│  │   ├── Verifies deployment health                                │
│  │   └── Runs DEV smoke tests                                      │
│  │                                                                  │
│  ├── Phase: LLE                                                     │
│  │   ├── Gate: DEV success ✓                                       │
│  │   ├── Same TFE flow → LLE cluster                               │
│  │   └── LLE integration tests                                     │
│  │                                                                  │
│  └── Phase: PROD                                                    │
│      ├── Gate: LLE success ✓                                       │
│      ├── Manual approval (CAB + App Owner + Release Manager)       │
│      ├── Captures rollback snapshot (empty for new app)            │
│      ├── Same TFE flow → PROD cluster                              │
│      ├── Extended health validation (5 min)                        │
│      └── PROD smoke tests                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.4 Scenario-Specific Behavior

```
┌────────────────────────────────────────┐
│  NEW-APP Specific Configuration        │
├────────────────────────────────────────┤
│  deploy_scenario   = \"new-app\"         │
│  force_restart     = false             │
│  image_pull_policy = \"IfNotPresent\"    │
│  create_namespace  = true (via TF)     │
│  KV imageTag       = UPDATED (new)     │
│  atomic            = true              │
│  recreate_pods     = false             │
└────────────────────────────────────────┘
```

---

## 10. CD Scenario 2 — App Image Update Deployment

### 10.1 Overview

Deploying a new version of an existing application image. The application code has changed; the image tag is different from what is currently running. This is the most common deployment scenario.

### 10.2 Prerequisites

| Prerequisite | Owner | Verification |
|-------------|-------|-------------|
| App already deployed (Scenario 1 completed) | Platform | Helm release exists in all envs |
| New image built and pushed by CI | App Team | New tag exists in registry |
| Image vulnerability scan passed | CI Pipeline | Scan report clean |
| KV config may/may not need updates | App Team | Validated at pipeline start |

### 10.3 Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│               SCENARIO 2: APP IMAGE UPDATE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CI Pipeline (completes)                                            │
│  ├── Builds new image: payment-svc:1.5.0                           │
│  ├── Pushes to registry                                            │
│  ├── Passes vulnerability scan                                     │
│  └── Triggers Jenkins CD pipeline (or auto-trigger)                │
│       │                                                             │
│       ▼                                                             │
│  Jenkins Pipeline                                                   │
│  ├── DEPLOY_SCENARIO = \"image-update\"                              │
│  ├── IMAGE_TAG = \"1.5.0\"                                           │
│  ├── Captures PREVIOUS_IMAGE_TAG from KV (e.g., \"1.4.0\")          │
│  └── Triggers XLR                                                  │
│       │                                                             │
│       ▼                                                             │
│  XLR Release                                                        │
│  ├── Phase: DEV                                                     │
│  │   ├── Updates KV: apps/{app}/dev/imageTag = \"1.5.0\"            │
│  │   ├── Triggers TFE: cd-{tenant}-{app}-dev                      │
│  │   │   └── TFE:                                                   │
│  │   │       ├── Reads KV → image_tag = \"1.5.0\"                   │
│  │   │       ├── Renders values.yaml (new tag)                     │
│  │   │       ├── helm_release.app detects image tag change         │
│  │   │       └── Rolling update → new pods with 1.5.0              │
│  │   ├── Verifies: new pods healthy, old pods terminated           │
│  │   └── DEV smoke tests pass                                      │
│  │                                                                  │
│  ├── Phase: LLE (same pattern)                                      │
│  │   ├── KV update: lle/imageTag = \"1.5.0\"                        │
│  │   ├── TFE → helm rolling update                                 │
│  │   └── Integration tests                                         │
│  │                                                                  │
│  └── Phase: PROD                                                    │
│      ├── Manual approval with change ticket                        │
│      ├── Snapshot: previous_image_tag = \"1.4.0\", revision = N     │
│      ├── KV update: prod/imageTag = \"1.5.0\"                       │
│      ├── TFE → helm rolling update                                 │
│      ├── Extended health check (5 min)                             │
│      ├── PROD smoke tests                                          │
│      └── On failure → Auto-rollback to \"1.4.0\"                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.4 Scenario-Specific Behavior

```
┌──────────────────────────────────────────┐
│  IMAGE-UPDATE Specific Configuration     │
├──────────────────────────────────────────┤
│  deploy_scenario   = \"image-update\"      │
│  force_restart     = false               │
│  image_pull_policy = \"IfNotPresent\"      │
│  create_namespace  = false               │
│  KV imageTag       = UPDATED (new ver)   │
│  atomic            = true                │
│  recreate_pods     = false               │
│  previous_image_tag = captured pre-deploy│
│                                          │
│  Helm detects image change →             │
│  triggers Deployment rolling update      │
│  automatically                           │
└──────────────────────────────────────────┘
```

### 10.5 Difference from Scenario 1

| Aspect | Scenario 1 (New App) | Scenario 2 (Image Update) |
|--------|---------------------|---------------------------|
| Namespace | Created | Already exists |
| Helm release | `install` | `upgrade` |
| Previous state | None | Captured for rollback |
| KV imageTag | First population | Overwritten |
| K8s resources | All created fresh | Updated in-place |
| Rollback target | Delete release | Previous image tag |

---

## 11. CD Scenario 3 — Repave (Base Image Update)

### 11.1 Overview

The application's base image has been updated (e.g., OS security patch, runtime update). A new application image has been built with the same application code but on the new base image. The image tag is new (e.g., `1.4.0-repave.1` or `1.4.0-base.20240115`), but the application version is unchanged.

### 11.2 Trigger Sources

```
┌───────────────────────────────────────────────────────────────┐
│               REPAVE TRIGGER SOURCES                          │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Security Scanner detects base image CVE                   │
│     └── Triggers CI rebuild with new base image               │
│         └── CI produces: payment-svc:1.4.0-base.20240115     │
│             └── Triggers Jenkins CD (scenario = repave)       │
│                                                               │
│  2. Scheduled base image refresh (e.g., monthly)              │
│     └── Platform team publishes new base images               │
│         └── All apps' CI pipelines rebuild                    │
│             └── Each triggers CD (scenario = repave)          │
│                                                               │
│  3. Compliance mandate (e.g., log4j remediation)              │
│     └── Emergency base image update                           │
│         └── Expedited repave across all apps                  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 11.3 Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│            SCENARIO 3: REPAVE (BASE IMAGE UPDATE)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CI Pipeline (rebuilds app on new base)                             │
│  ├── Same app code, new base image                                 │
│  ├── New tag: payment-svc:1.4.0-base.20240115                     │
│  ├── Pushes to registry                                            │
│  └── Triggers Jenkins CD pipeline                                  │
│       │                                                             │
│       ▼                                                             │
│  Jenkins Pipeline                                                   │
│  ├── DEPLOY_SCENARIO = \"repave\"                                    │
│  ├── IMAGE_TAG = \"1.4.0-base.20240115\"                             │
│  ├── IMAGE_TAG_OVERRIDE = \"1.4.0-base.20240115\"                   │
│  ├── REPAVE_REASON = \"base-image-update\"                           │
│  ├── Captures PREVIOUS_IMAGE_TAG = \"1.4.0\"                        │
│  ├── Validates new image exists and scan passed                    │
│  └── Triggers XLR (repave template)                                │
│       │                                                             │
│       ▼                                                             │
│  XLR Release (Repave Template)                                      │
│  ├── Phase: Validation                                              │
│  │   ├── Verifies new image exists                                 │
│  │   ├── Verifies new image scan passed                            │
│  │   └── Verifies same app version, different base                 │
│  │                                                                  │
│  ├── Phase: DEV                                                     │
│  │   ├── Updates KV: dev/imageTag = \"1.4.0-base.20240115\"         │
│  │   ├── TFE run with image_tag_override                           │
│  │   │   └── TFE:                                                   │
│  │   │       ├── resolved_image_tag = override value               │
│  │   │       ├── previous_image_tag = \"1.4.0\"                     │
│  │   │       ├── repave_reason = \"base-image-update\"               │
│  │   │       └── helm upgrade → rolling update to new image        │
│  │   └── DEV verification (lightweight smoke test)                 │
│  │                                                                  │
│  ├── Phase: LLE                                                     │
│  │   ├── Gate: DEV success                                         │
│  │   ├── Same repave flow                                          │
│  │   └── Regression test (can be abbreviated for repave)           │
│  │                                                                  │
│  └── Phase: PROD                                                    │
│      ├── Gate: LLE success                                         │
│      ├── Manual approval (Platform team, expedited for security)   │
│      ├── Snapshot: previous_image_tag = \"1.4.0\"                   │
│      ├── KV update + TFE run                                       │
│      ├── Extended health check                                     │
│      └── On failure → rollback to \"1.4.0\"                         │
│                                                                     │
│  POST-DEPLOY:                                                       │
│  ├── Update KV: all envs reflect new base image tag                │
│  └── Audit: record repave completion with base image lineage       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.4 Scenario-Specific Behavior

```
┌──────────────────────────────────────────────────────────┐
│  REPAVE Specific Configuration                           │
├──────────────────────────────────────────────────────────┤
│  deploy_scenario    = \"repave\"                           │
│  force_restart      = false                              │
│  image_pull_policy  = \"IfNotPresent\"                     │
│  create_namespace   = false                              │
│  image_tag_override = \"1.4.0-base.20240115\"             │
│  previous_image_tag = \"1.4.0\" (captured)                │
│  repave_reason      = \"base-image-update\"               │
│  KV imageTag        = UPDATED to new base tag           │
│  atomic             = true                               │
│  recreate_pods      = false                              │
│                                                          │
│  Key Annotations Added:                                  │
│  enterprise.com/repave-reason: \"base-image-update\"      │
│  enterprise.com/previous-image-tag: \"1.4.0\"             │
│                                                          │
│  Testing: May use abbreviated test suite since           │
│  app code is unchanged                                   │
└──────────────────────────────────────────────────────────┘
```

### 11.5 Repave Image Tagging Convention

```
┌────────────────────────────────────────────────────────────────┐
│                  IMAGE TAG CONVENTIONS                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Original release:   payment-svc:1.4.0                        │
│  Base image update:  payment-svc:1.4.0-base.20240115          │
│  Second repave:      payment-svc:1.4.0-base.20240201          │
│                                                                │
│  Alternative scheme using build metadata:                      │
│  payment-svc:1.4.0+base.20240115                              │
│                                                                │
│  The tag clearly indicates:                                    │
│  - Same app version (1.4.0)                                   │
│  - Different base image (date-stamped)                         │
│  - Traceable lineage                                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 12. CD Scenario 4 — CD-Only Repave (Redeploy Existing Production Image)

### 12.1 Overview

Redeploy the **exact same image** currently running in production without any image change. This is a \"repave\" in the infrastructure sense — tear down and recreate pods to pick up node-level changes, rotate credentials, clear ephemeral state, or satisfy compliance rotation requirements.

**Key distinction from Scenario 3:** No new image is built. The existing production image tag is reused. The goal is to restart workloads.

### 12.2 Use Cases

```
┌───────────────────────────────────────────────────────────────┐
│              CD-REPAVE USE CASES                              │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Node OS patched → need to cycle pods to new nodes         │
│  2. Service mesh sidecar updated → restart to pick up         │
│  3. Credential/certificate rotation → restart to reload       │
│  4. Compliance: max pod age policy (e.g., 30-day rotation)    │
│  5. ConfigMap/Secret updated externally → restart to load     │
│  6. Memory leak mitigation → periodic restart                 │
│  7. Post-incident: clean restart of all pods                  │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 12.3 Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│         SCENARIO 4: CD-ONLY REPAVE (REDEPLOY EXISTING IMAGE)       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Trigger (Platform team or scheduled job)                           │
│  └── Jenkins CD pipeline                                           │
│       │                                                             │
│       ▼                                                             │
│  Jenkins Pipeline                                                   │
│  ├── DEPLOY_SCENARIO = \"cd-repave\"                                 │
│  ├── IMAGE_TAG = \"\" (empty - resolved from KV)                     │
│  ├── FORCE_RESTART = true (auto-set for cd-repave)                 │
│  ├── REPAVE_REASON = \"node-os-patch\" (or other reason)             │
│  ├── Resolves IMAGE_TAG from KV: apps/{app}/prod/imageTag          │
│  │   └── Resolved: \"1.4.0-base.20240115\" (whatever is current)    │
│  └── Triggers XLR (cd-repave template)                             │
│       │                                                             │
│       ▼                                                             │
│  XLR Release (CD-Repave Template)                                   │
│  ├── Phase: Validation                                              │
│  │   ├── Verifies existing image still in registry                 │
│  │   ├── Verifies current deployment is healthy                    │
│  │   └── Skips: image scan (already scanned), build steps          │
│  │                                                                  │
│  ├── Phase: DEV                                                     │
│  │   ├── KV imageTag: NO CHANGE (same tag)                         │
│  │   ├── TFE run:                                                   │
│  │   │   ├── deploy_scenario = \"cd-repave\"                         │
│  │   │   ├── force_restart = true                                  │
│  │   │   ├── image_tag = current tag (from KV)                     │
│  │   │   ├── image_pull_policy = \"Always\" (re-pull same tag)       │
│  │   │   │                                                          │
│  │   │   └── TFE applies helm_release with:                        │
│  │   │       ├── force_update = true                               │
│  │   │       ├── recreate_pods = true                              │
│  │   │       └── restart-nonce annotation (random) forces rollout  │
│  │   │                                                              │
│  │   │   K8s behavior:                                              │
│  │   │   ├── All existing pods terminated                          │
│  │   │   ├── New pods created with same image                      │
│  │   │   ├── Pods scheduled on current nodes                       │
│  │   │   └── PDB respected during rollout                          │
│  │   │                                                              │
│  │   └── DEV health check (pod healthy, endpoint responding)       │
│  │       (Skip functional tests - no app change)                   │
│  │                                                                  │
│  ├── Phase: LLE                                                     │
│  │   ├── Gate: DEV success                                         │
│  │   ├── Same cd-repave flow                                       │
│  │   └── Basic connectivity test only                              │
│  │                                                                  │
│  └── Phase: PROD                                                    │
│      ├── Gate: LLE success                                         │
│      ├── Manual approval: Platform team only (expedited)           │
│      │   (No CAB needed - no application change)                   │
│      ├── Snapshot: current state (same image, new revision)        │
│      ├── Same cd-repave TFE run                                   │
│      ├── Extended health check (5 min)                             │
│      ├── Basic connectivity smoke test                             │
│      └── On failure → pods already terminated;                     │
│          rollback = re-apply same image (should recover)           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.4 Scenario-Specific Behavior

```
┌──────────────────────────────────────────────────────────────┐
│  CD-REPAVE Specific Configuration                            │
├──────────────────────────────────────────────────────────────┤
│  deploy_scenario    = \"cd-repave\"                            │
│  force_restart      = true  ← ALWAYS TRUE                   │
│  image_pull_policy  = \"Always\" ← RE-PULL SAME IMAGE         │
│  create_namespace   = false                                  │
│  image_tag          = resolved from KV (current prod tag)    │
│  image_tag_override = \"\" (empty)                             │
│  previous_image_tag = same as current (no change)            │
│  repave_reason      = \"node-os-patch\" | \"credential-rotation\"│
│  KV imageTag        = NO CHANGE                              │
│  atomic             = true                                   │
│  recreate_pods      = true  ← FORCE POD RECREATION          │
│  force_update       = true  ← FORCE HELM UPGRADE            │
│                                                              │
│  Template Annotation (forces rollout):                       │
│  enterprise.com/restart-nonce: <random-10-char>             │
│                                                              │
│  Testing: Health check only, no functional tests             │
│  Approval: Platform team only (no app team/CAB needed)       │
│                                                              │
│  RISK CONSIDERATION:                                         │
│  Since pods are destroyed and recreated with the same image, │
│  the primary risk is infrastructure-level (node capacity,    │
│  image pull failures). PDB ensures minimum availability      │
│  during the rollout.                                         │
└──────────────────────────────────────────────────────────────┘
```

### 12.5 How Pod Restart is Forced (Technical Detail)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  FORCING POD RESTART — THREE MECHANISMS             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Mechanism 1: Annotation Nonce (Primary)                            │
│  ┌───────────────────────────────────────────────────────────┐     │
│  │  In deployment.yaml template:                             │     │
│  │                                                           │     │
│  │  {{- if .Values.deploy.forceRestart }}                    │     │
│  │  enterprise.com/restart-nonce: {{ randAlphaNum 10 }}      │     │
│  │  {{- end }}                                               │     │
│  │                                                           │     │
│  │  A new random value on every helm upgrade forces the      │     │
│  │  Deployment's pod template to change, triggering a        │     │
│  │  rolling update even though the image is unchanged.       │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Mechanism 2: Helm force_update + recreate_pods (Secondary)        │
│  ┌───────────────────────────────────────────────────────────┐     │
│  │  In main.tf:                                              │     │
│  │                                                           │     │
│  │  resource \"helm_release\" \"app\" {                          │     │
│  │    force_update  = true   # Forces upgrade even if        │     │
│  │                           # no chart changes detected     │     │
│  │    recreate_pods = true   # Deletes and recreates pods    │     │
│  │                           # during upgrade                │     │
│  │  }                                                        │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Mechanism 3: imagePullPolicy = Always (Supporting)                 │
│  ┌───────────────────────────────────────────────────────────┐     │
│  │  Ensures that even with the same tag, the image is        │     │
│  │  re-pulled from registry. Important if the tag is         │     │
│  │  mutable (though immutable tags are recommended).         │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 13. Promotion Model & Gating Strategy

### 13.1 Scenario-Gating Matrix

```
┌─────────────────────┬──────────────┬──────────────┬──────────────────┬───────────────┐
│     Gate            │ New App (S1) │ Img Upd (S2) │ Repave (S3)      │ CD-Repave(S4) │
├─────────────────────┼──────────────┼──────────────┼──────────────────┼───────────────┤
│ Image exists        │      ✅      │      ✅      │       ✅         │    ✅ (KV)    │
│ Vuln scan pass      │      ✅      │      ✅      │       ✅         │    ⏭ Skip     │
│ Image signed        │      ✅      │      ✅      │       ✅         │    ⏭ Skip     │
│ KV config valid     │      ✅      │      ✅      │       ✅         │    ✅         │
│ DEV deploy pass     │      ✅      │      ✅      │       ✅         │    ✅         │
│ DEV smoke test      │      ✅      │      ✅      │    ✅ (light)    │    ⏭ Skip     │
│ LLE deploy pass     │      ✅      │      ✅      │       ✅         │    ✅         │
│ LLE integration test│      ✅      │      ✅      │    ✅ (abbrev)   │    ⏭ Skip     │
│ PROD manual approval│   App+CAB    │   App+CAB    │  Platform+App    │ Platform only │
│ Change ticket       │      ✅      │      ✅      │    ✅ (expedited)│    ⏭ Skip     │
│ PROD deploy window  │      ✅      │      ✅      │       ✅         │    ✅         │
│ PROD health check   │   5 min      │    5 min     │     5 min        │    3 min      │
│ PROD smoke test     │      ✅      │      ✅      │    ✅ (light)    │ Connectivity  │
│ Auto-rollback       │   Delete     │ Prev tag     │   Prev tag       │ Re-apply same │
└─────────────────────┴──────────────┴──────────────┴──────────────────┴───────────────┘
```

### 13.2 Environment Promotion Rules

```
DEV → LLE Promotion:
  ├── Automated gate
  ├── Requires: DEV deployment healthy + smoke tests pass
  ├── No human approval needed
  └── Timeout: 30 min after DEV success

LLE → PROD Promotion:
  ├── Automated gate (technical) + Manual gate (approval)
  ├── Requires: LLE deployment healthy + integration tests pass
  ├── Requires: Manual approval (per scenario matrix above)
  ├── Requires: Within deployment window (configurable)
  └── Timeout: Configurable (default 24h for approval)
```

---

## 14. Rollback Strategy

### 14.1 Rollback Matrix by Scenario

```
┌──────────────┬─────────────────────────────┬──────────────────────────────┐
│   Scenario   │     Automatic Rollback      │     Manual Rollback          │
├──────────────┼─────────────────────────────┼──────────────────────────────┤
│ New App      │ helm atomic → auto-delete   │ TFE run with terraform      │
│              │ failed release              │ destroy on helm_release      │
├──────────────┼─────────────────────────────┼──────────────────────────────┤
│ Image Update │ Re-trigger TFE with         │ Re-run pipeline with        │
│              │ previous_image_tag          │ IMAGE_TAG = previous tag     │
│              │ + restore KV imageTag       │                              │
├──────────────┼─────────────────────────────┼──────────────────────────────┤
│ Repave       │ Re-trigger TFE with         │ Re-run pipeline with        │
│              │ previous_image_tag          │ original pre-repave tag      │
│              │ (pre-repave tag)            │                              │
├──────────────┼─────────────────────────────┼──────────────────────────────┤
│ CD-Repave    │ Re-apply same image         │ Usually self-healing;       │
│              │ (pods should recover)       │ escalate to infra if not     │
│              │ If pod scheduling fails →   │                              │
│              │ investigate node/cluster    │                              │
└──────────────┴─────────────────────────────┴──────────────────────────────┘
```

### 14.2 Rollback Data Stored in KV

```
apps/{app-name}/{env}/rollback/
├── imageTag           = \"1.4.0\"          # Previous image
├── helmRevision       = \"15\"             # Previous Helm revision
├── deployTimestamp    = \"2024-01-14...\"   # When snapshot taken
├── valuesChecksum     = \"sha256:abc...\"  # Previous values hash
└── tfeRunId           = \"run-xyz123\"     # TFE run that deployed it
```

---

## 15. Security & Governance

### 15.1 Access Control

```
┌─────────────────────────────────────────────────────────────────┐
│                    RBAC MODEL                                    │
├──────────────┬──────────────────────────────────────────────────┤
│ Role         │ Permissions                                      │
├──────────────┼──────────────────────────────────────────────────┤
│ App Team     │ Trigger S1, S2 deployments for their apps       │
│              │ View all deployment logs                          │
│              │ Approve LLE→PROD for their apps                  │
│              │ Update KV config for their app paths              │
├──────────────┼──────────────────────────────────────────────────┤
│ Platform     │ Trigger S3, S4 deployments for all apps          │
│ Team         │ Approve S3, S4 to PROD                           │
│              │ Manage TFE workspaces                             │
│              │ Update global/tenant KV config                    │
│              │ Emergency rollback any app                        │
├──────────────┼──────────────────────────────────────────────────┤
│ Release      │ Approve any PROD deployment                      │
│ Manager      │ View all release dashboards                      │
│              │ Override gates (with audit)                       │
├──────────────┼──────────────────────────────────────────────────┤
│ Auditor      │ Read-only access to all deployment history       │
│              │ XLR release reports                               │
│              │ TFE state and run history                         │
└──────────────┴──────────────────────────────────────────────────┘
```

### 15.2 Security Controls

```
┌──────────────────────────────────────────────────────────────────┐
│                   SECURITY CONTROLS                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Image Security:                                                 │
│  ├── All images scanned before deployment (Trivy/Prisma)        │
│  ├── Image signing verified (cosign) in validation phase        │
│  ├── Only images from approved registry allowed                  │
│  └── Image pull secrets managed via sealed-secrets/ESO           │
│                                                                  │
│  Pipeline Security:                                              │
│  ├── Jenkins→XLR: API token (rotated 90 days)                   │
│  ├── XLR→TFE: Team API token (workspace-scoped)                 │
│  ├── TFE→K8s: Short-lived ServiceAccount token                  │
│  ├── TFE→Consul: ACL token (app-path scoped)                    │
│  └── All secrets stored in Vault, injected at runtime            │
│                                                                  │
│  Kubernetes Security:                                            │
│  ├── Pod Security Standards enforced (restricted profile)        │
│  ├── Network Policies applied by common chart                    │
│  ├── No privileged containers allowed                            │
│  ├── Read-only root filesystem enforced                          │
│  └── Resource quotas enforced per namespace                      │
│                                                                  │
│  Terraform Security:                                             │
│  ├── Sentinel policies enforce:                                  │
│  │   ├── Only approved helm chart source                         │
│  │   ├── Image must come from enterprise registry                │
│  │   ├── Resource limits must be set                             │
│  │   └── Security context must be present                        │
│  ├── State encrypted at rest                                     │
│  └── Plan output reviewed (for prod, auto-apply disabled)        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 15.3 Sentinel Policy Example (TFE)

```python
# sentinel/restrict-helm-deployment.sentinel

import \"tfplan/v2\" as tfplan

# Ensure only the approved common chart is used
approved_chart = \"common-app-chart\"
approved_repo  = \"https://charts.enterprise.com\"

helm_releases = filter tfplan.resource_changes as _, rc {
    rc.type is \"helm_release\" and
    rc.mode is \"managed\" and
    (rc.change.actions contains \"create\" or rc.change.actions contains \"update\")
}

chart_approved = rule {
    all helm_releases as _, release {
        release.change.after.chart is approved_chart and
        release.change.after.repository is approved_repo
    }
}

image_from_enterprise_registry = rule {
    all helm_releases as _, release {
        release.change.after.values contains \"enterprise-registry.com\"
    }
}

main = rule {
    chart_approved and image_from_enterprise_registry
}
```

---

## 16. Observability & Audit

### 16.1 Deployment Traceability Chain

```
┌──────────────────────────────────────────────────────────────────────┐
│                     TRACEABILITY CHAIN                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Every deployment can be traced through:                             │
│                                                                      │
│  Jenkins Build #N                                                    │
│  ├── Build URL: https://jenkins.enterprise.com/job/cd-payment/N     │
│  ├── Parameters: APP=payment-svc, TAG=1.5.0, SCENARIO=image-update  │
│  │                                                                   │
│  └──▶ XLR Release: CD-payment-svc-image-update-N                    │
│       ├── Release ID: Release-12345                                  │
│       ├── Phases: DEV ✅ → LLE ✅ → PROD ✅                          │
│       ├── Approvals: John Doe (2024-01-15 14:30 UTC)                │
│       │                                                              │
│       └──▶ TFE Run: run-abc123 (per environment)                    │
│            ├── Workspace: cd-digital-banking-payment-svc-prod        │
│            ├── Plan: Approved                                        │
│            ├── Apply: Succeeded                                      │
│            │                                                         │
│            └──▶ Helm Release: payment-svc (revision 16)              │
│                 ├── Namespace: digital-banking-payment-svc-prod      │
│                 ├── Image: enterprise-registry.com/.../payment:1.5.0│
│                 │                                                    │
│                 └──▶ K8s Deployment                                  │
│                      ├── Annotations:                                │
│                      │   enterprise.com/deployed-by: terraform       │
│                      │   enterprise.com/deploy-scenario: image-update│
│                      │   enterprise.com/source-pipeline: (URL)       │
│                      │   enterprise.com/xlr-release-id: Release-12345│
│                      │   enterprise.com/tfe-run-id: run-abc123       │
│                      └── Pods: 3/3 Running                           │
│                                                                      │
│  KV Deployed State:                                                  │
│  apps/payment-svc/prod/deployed/                                     │
│  ├── imageTag: \"1.5.0\"                                              │
│  ├── deployScenario: \"image-update\"                                  │
│  ├── deployTimestamp: \"2024-01-15T14:45:00Z\"                        │
│  ├── helmRevision: \"16\"                                              │
│  ├── tfeRunId: \"run-abc123\"                                         │
│  ├── xlrReleaseId: \"Release-12345\"                                  │
│  └── jenkinsBuildUrl: \"https://jenkins.../N\"                        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 16.2 Monitoring & Alerting

```
┌─────────────────────────────────────────────────────────────────┐
│                  MONITORING INTEGRATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Deploy-time Monitoring:                                        │
│  ├── Deployment duration (Jenkins → pods ready)                │
│  ├── Rollback rate by scenario type                            │
│  ├── Gate failure rate                                          │
│  ├── TFE run duration and failure rate                         │
│  └── Approval wait time                                        │
│                                                                 │
│  Post-deploy Monitoring (integrated into health checks):       │
│  ├── Pod readiness and restart counts                          │
│  ├── HTTP 5xx error rate (compared to pre-deploy baseline)     │
│  ├── Request latency p50/p95/p99                               │
│  ├── CPU/Memory utilization vs requests/limits                 │
│  └── Custom app-specific metrics (if configured)               │
│                                                                 │
│  Dashboard: Grafana                                             │
│  Alerting: PagerDuty (prod failures), Slack (all envs)         │
│  Logging: ELK/Splunk (deployment events correlated with app)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 17. Appendices

### Appendix A: Scenario Comparison Summary

```
┌─────────────────┬───────────────┬───────────────┬─────────────────┬──────────────────┐
│   Attribute     │  S1: New App  │ S2: Img Update│ S3: Repave      │ S4: CD-Repave    │
├─────────────────┼───────────────┼───────────────┼─────────────────┼──────────────────┤
│ New image?      │ Yes (first)   │ Yes (new ver) │ Yes (new base)  │ No (same image)  │
│ App code change?│ Yes           │ Yes           │ No              │ No               │
│ Base img change?│ N/A           │ Maybe         │ Yes             │ No               │
│ KV tag updated? │ Yes           │ Yes           │ Yes             │ No               │
│ Force restart?  │ No            │ No            │ No              │ Yes              │
│ recreate_pods?  │ No            │ No            │ No              │ Yes              │
│ Pull policy     │ IfNotPresent  │ IfNotPresent  │ IfNotPresent    │ Always           │
│ Create NS?      │ Yes           │ No            │ No              │ No               │
│ Full test suite?│ Yes           │ Yes           │ Abbreviated     │ Health check only│
│ PROD approval   │ App+CAB       │ App+CAB       │ Platform+App    │ Platform only    │
│ Change ticket?  │ Yes           │ Yes           │ Expedited       │ Not required     │
│ Trigger source  │ App team      │ CI pipeline   │ CI (base rebuild│ Platform/Schedule│
│ Rollback target │ Delete release│ Previous tag  │ Pre-repave tag  │ Re-apply same    │
│ Risk level      │ Medium        │ Medium        │ Low             │ Very Low         │
│ Typical freq    │ Once          │ Weekly        │ Monthly         │ As needed        │
└─────────────────┴───────────────┴───────────────┴─────────────────┴──────────────────┘
```

### Appendix B: Pipeline Trigger Matrix

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     TRIGGER → PIPELINE MAPPING                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CI Build Complete (new app code)                                       │
│  └── Jenkins: containerCD(DEPLOY_SCENARIO: 'image-update')             │
│                                                                         │
│  First-time App Onboarding                                              │
│  └── Jenkins: containerCD(DEPLOY_SCENARIO: 'new-app')                  │
│       (Manual trigger by app team after onboarding checklist complete)  │
│                                                                         │
│  Base Image Updated (CI rebuilds app)                                   │
│  └── Jenkins: containerCD(DEPLOY_SCENARIO: 'repave',                   │
│               REPAVE_REASON: 'base-image-update')                      │
│                                                                         │
│  Scheduled Repave (cron)                                                │
│  └── Jenkins: containerCD(DEPLOY_SCENARIO: 'cd-repave',               │
│               REPAVE_REASON: 'scheduled-rotation')                     │
│                                                                         │
│  Emergency Security Patch (base image)                                  │
│  └── Jenkins: containerCD(DEPLOY_SCENARIO: 'repave',                   │
│               REPAVE_REASON: 'security-patch')                         │
│       (Expedited approval flow)                                        │
│                                                                         │
│  Node OS Patch (infra change)                                           │
│  └── Jenkins: containerCD(DEPLOY_SCENARIO: 'cd-repave',               │
│               REPAVE_REASON: 'node-os-patch')                          │
│       (Batch trigger for all apps on affected nodes)                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Appendix C: Batch Repave Orchestration (CD-Repave at Scale)

```groovy
// Jenkinsfile for batch cd-repave across multiple apps
pipeline {
    agent { label 'cd-agent' }
    parameters {
        string(name: 'TENANT_NAME', description: 'Tenant to repave')
        string(name: 'REPAVE_REASON', defaultValue: 'node-os-patch')
        string(name: 'CONCURRENCY', defaultValue: '3',
               description: 'Max parallel repaves')
    }
    stages {
        stage('Discover Apps') {
            steps {
                script {
                    // Fetch all apps for tenant from KV
                    env.APP_LIST = discoverTenantApps(params.TENANT_NAME)
                }
            }
        }
        stage('Batch CD-Repave') {
            steps {
                script {
                    def apps = env.APP_LIST.split(',')
                    def batches = apps.collate(params.CONCURRENCY.toInteger())

                    batches.each { batch ->
                        def parallelSteps = [:]
                        batch.each { app ->
                            parallelSteps[\"repave-${app}\"] = {
                                build job: 'container-cd',
                                      parameters: [
                                          string(name: 'APP_NAME', value: app),
                                          string(name: 'TENANT_NAME', value: params.TENANT_NAME),
                                          string(name: 'DEPLOY_SCENARIO', value: 'cd-repave'),
                                          string(name: 'REPAVE_REASON', value: params.REPAVE_REASON)
                                      ],
                                      wait: true
                            }
                        }
                        parallel parallelSteps
                    }
                }
            }
        }
    }
}
```

### Appendix D: KV Config Update Automation

```groovy
// vars/updateKVImageTag.groovy
// Used by XLR phases to update KV before TFE trigger

def call(Map params) {
    // params: appName, environment, imageTag, scenario
    
    if (params.scenario == 'cd-repave') {
        echo \"CD-Repave: Skipping KV imageTag update (no image change)\"
        return
    }
    
    def kvPath = \"apps/${params.appName}/${params.environment}/imageTag\"
    
    // Backup current value
    def currentTag = readConsulKey(kvPath)
    writeConsulKey(\"apps/${params.appName}/${params.environment}/rollback/imageTag\", currentTag)
    
    // Update to new tag
    writeConsulKey(kvPath, params.imageTag)
    
    echo \"Updated KV: ${kvPath} = ${params.imageTag} (was: ${currentTag})\"
}
```

---

**Document Version:** 2.0
**Last Updated:** 2024-01-15
**Author:** Platform Engineering Team
**Review Cycle:** Quarterly
**Next Review:** 2024-04-15
