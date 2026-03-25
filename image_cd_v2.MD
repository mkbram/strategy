# Container Continuous Delivery (CD) Design Document

## Enterprise Multi-Tenant Container CD Platform

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Toolchain & Component Responsibilities](#3-toolchain--component-responsibilities)
4. [Common Helm Template Design](#4-common-helm-template-design)
5. [Configuration Management & Values Resolution](#5-configuration-management--values-resolution)
6. [Terraform Enterprise Wrapper Design](#6-terraform-enterprise-wrapper-design)
7. [XL Release Template Design](#7-xl-release-template-design)
8. [Jenkins Pipeline Design](#8-jenkins-pipeline-design)
9. [CD Scenario 1 — New App Image Deployment](#9-cd-scenario-1--new-app-image-deployment)
10. [CD Scenario 2 — App Image Update Deployment](#10-cd-scenario-2--app-image-update-deployment)
11. [CD Scenario 3 — Repave for Base Image Update](#11-cd-scenario-3--repave-for-base-image-update)
12. [CD Scenario 4 — CD-Only Repave (Redeploy Existing Production Image)](#12-cd-scenario-4--cd-only-repave-redeploy-existing-production-image)
13. [Environment Promotion Strategy](#13-environment-promotion-strategy)
14. [Rollback Strategy](#14-rollback-strategy)
15. [Security & Governance](#15-security--governance)
16. [Observability & Audit](#16-observability--audit)
17. [Appendices](#17-appendices)

---

## 1. Overview

### 1.1 Purpose

This document defines the Continuous Delivery architecture and process for deploying containerized applications to Kubernetes clusters across a multi-tenant enterprise platform. It covers four distinct deployment scenarios under a unified toolchain and governance model.

### 1.2 Scope

| In Scope | Out of Scope |
|----------|--------------|
| CD pipeline design for 4 scenarios | CI (build/test/scan) pipeline design |
| Helm template standardization | Kubernetes cluster provisioning |
| Environment promotion (DEV → LLE → PROD) | Image build process |
| Configuration externalization | Secret engine implementation details |
| Rollback strategy | Application-specific functional testing |

### 1.3 Definitions

| Term | Definition |
|------|------------|
| **New App Deployment** | First-time deployment of a containerized application to an environment |
| **Image Update** | Deployment of a new version of the application image (code change) |
| **Repave (Base Image Update)** | Rebuild and deploy application image using an updated base image (OS/runtime patches) with no app code change |
| **CD-Only Repave** | Restart/redeploy the currently running production image without any image change — forces pod recreation |
| **LLE** | Lower Lifecycle Environment (staging/pre-prod) |
| **Common Helm Template** | Shared, parameterized Helm chart used by all tenant applications |
| **Thin Wrapper** | Minimal Terraform configuration that delegates to the Helm provider |
| **KV Config Manager** | External key-value configuration store (e.g., Consul, Spring Cloud Config, Azure App Config) |

---

## 2. Architecture

### 2.1 High-Level Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         TRIGGER LAYER                                    │
│                                                                          │
│  ┌─────────────┐   Webhook/     ┌──────────────────────┐                │
│  │   Jenkins    │──────────────►│    XL Release (XLR)   │                │
│  │  Pipelines   │   API Call    │  Environment Templates │                │
│  └─────────────┘               └──────────┬───────────┘                │
│        ▲                                   │                             │
│        │                                   │ REST API / Plugin           │
│   Trigger Sources:                         ▼                             │
│   - CI completion                ┌──────────────────────┐                │
│   - Manual                       │ Terraform Enterprise  │                │
│   - Schedule (repave)            │   (Workspaces)        │                │
│   - Base image event             └──────────┬───────────┘                │
│                                             │                             │
└─────────────────────────────────────────────┼────────────────────────────┘
                                              │
┌─────────────────────────────────────────────┼────────────────────────────┐
│                    EXECUTION LAYER           │                            │
│                                             ▼                            │
│  ┌────────────────┐       ┌──────────────────────┐                      │
│  │  KV Config     │◄──────│  Terraform Helm       │                      │
│  │  Manager       │ fetch │  Provider (thin       │                      │
│  │ (Consul/etc.)  │ values│  wrapper + templated  │                      │
│  └────────────────┘       │  values file)         │                      │
│                           └──────────┬───────────┘                      │
│                                      │ helm install/upgrade              │
│                                      ▼                                   │
│                           ┌──────────────────────┐                      │
│                           │  Common Helm Chart    │                      │
│                           │  (shared template)    │                      │
│                           └──────────┬───────────┘                      │
│                                      │                                   │
│                                      ▼                                   │
│                           ┌──────────────────────┐                      │
│                           │  Kubernetes Cluster   │                      │
│                           │  (multi-tenant)       │                      │
│                           └──────────────────────┘                      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Interaction Sequence (Generic)

```
Jenkins        XL Release       TFE              KV Config       Helm/K8s
  │                │              │                  │               │
  │──trigger──────►│              │                  │               │
  │  (params)      │              │                  │               │
  │                │──run TFE────►│                  │               │
  │                │  workspace   │                  │               │
  │                │              │──fetch values───►│               │
  │                │              │◄─resolved vals───│               │
  │                │              │                  │               │
  │                │              │──render values───│               │
  │                │              │  template        │               │
  │                │              │                  │               │
  │                │              │──helm upgrade────────────────────►
  │                │              │  (common chart + │               │
  │                │              │   custom values) │               │
  │                │              │◄─────────────────────────────────│
  │                │◄─status──────│                  │               │
  │◄──status───────│              │                  │               │
  │                │              │                  │               │
```

---

## 3. Toolchain & Component Responsibilities

### 3.1 Jenkins Pipelines

| Responsibility | Details |
|----------------|---------|
| **Trigger Orchestration** | Entry point for all CD scenarios |
| **Parameter Collection** | Gather app name, image tag, environment, scenario type |
| **XLR Invocation** | Call XL Release API to start the appropriate release template |
| **Status Reporting** | Poll and report XLR release status back to caller |
| **RBAC Gating** | Enforce who can trigger which scenario and environment |

Jenkins does **not** perform the deployment itself.

### 3.2 XL Release (XLR)

| Responsibility | Details |
|----------------|---------|
| **Deployment Orchestration** | Manage environment-specific deployment workflow (DEV → LLE → PROD) |
| **Approval Gates** | Manual/automated approvals between environments |
| **TFE Invocation** | Trigger Terraform Enterprise runs per environment |
| **Governance & Audit** | Full release audit trail, compliance evidence |
| **Environment Templates** | Maintain separate templates for DEV, LLE, PROD |
| **Notification** | Slack/email/ServiceNow integration for status updates |

### 3.3 Terraform Enterprise (TFE)

| Responsibility | Details |
|----------------|---------|
| **Thin Wrapper Execution** | Execute minimal Terraform code that invokes the Helm provider |
| **State Management** | Track Helm release state per app per environment |
| **Values Resolution** | Resolve templated values file by fetching from KV Config Manager |
| **Workspace Isolation** | One workspace per app per environment |
| **Policy as Code** | Sentinel policies for image provenance, namespace enforcement |
| **Plan/Apply Model** | Plan shown in XLR for approval; Apply on confirmation |

### 3.4 Common Helm Chart

| Responsibility | Details |
|----------------|---------|
| **Standardized K8s Manifests** | Deployment, Service, Ingress, HPA, PDB, ConfigMap, ServiceAccount |
| **Parameterized Templates** | All app-specific values injected via values file |
| **Versioned** | Stored in Helm chart repository; chart version pinned per release |
| **Multi-Tenancy Support** | Namespace, resource quotas, labels, annotations parameterized |

### 3.5 KV Config Manager

| Responsibility | Details |
|----------------|---------|
| **Externalized Configuration** | Store environment-specific, app-specific config values |
| **Hierarchical Keys** | `/{env}/{app}/{key}` structure |
| **Version History** | Track config change history |
| **Access Control** | Per-app, per-environment read/write policies |

---

## 4. Common Helm Template Design

### 4.1 Chart Structure

```
common-helm-chart/
├── Chart.yaml
├── values.yaml                    # Defaults (overridden by custom values)
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   ├── networkpolicy.yaml
│   ├── podsecuritypolicy.yaml
│   └── NOTES.txt
└── README.md
```

### 4.2 Key Values Schema

```yaml
# values.yaml — default schema (overridden per app)

global:
  environment: \"\"            # dev | lle | prod
  tenant: \"\"                 # team/business-unit identifier
  clusterName: \"\"

app:
  name: \"\"
  namespace: \"\"
  version: \"\"                # app semantic version (label)

image:
  registry: \"\"
  repository: \"\"
  tag: \"\"
  pullPolicy: IfNotPresent
  pullSecrets: []

deployment:
  replicaCount: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  annotations: {}
  podAnnotations: {}
  restartTimestamp: \"\"        # ← KEY: used for CD-only repave (Scenario 4)

container:
  ports:
    - name: http
      containerPort: 8080
      protocol: TCP
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  livenessProbe:
    httpGet:
      path: /healthz
      port: http
    initialDelaySeconds: 30
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /ready
      port: http
    initialDelaySeconds: 10
    periodSeconds: 5
  env: []
  envFrom: []

service:
  type: ClusterIP
  port: 80
  targetPort: http

ingress:
  enabled: false
  className: \"\"
  annotations: {}
  hosts: []
  tls: []

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70

pdb:
  enabled: true
  minAvailable: 1

serviceAccount:
  create: true
  name: \"\"
  annotations: {}

networkPolicy:
  enabled: true

labels: {}
commonAnnotations: {}
```

### 4.3 Deployment Template (Key Excerpt)

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include \"common.fullname\" . }}
  namespace: {{ .Values.app.namespace }}
  labels:
    {{- include \"common.labels\" . | nindent 4 }}
  annotations:
    {{- include \"common.annotations\" . | nindent 4 }}
spec:
  replicas: {{ .Values.deployment.replicaCount }}
  strategy:
    {{- toYaml .Values.deployment.strategy | nindent 4 }}
  selector:
    matchLabels:
      {{- include \"common.selectorLabels\" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include \"common.selectorLabels\" . | nindent 8 }}
        image-tag: {{ .Values.image.tag | quote }}
      annotations:
        # Forces rolling restart when timestamp changes (Scenario 4)
        rollout/restartTimestamp: {{ .Values.deployment.restartTimestamp | default \"\" | quote }}
        checksum/config: {{ include (print .Template.BasePath \"/configmap.yaml\") . | sha256sum }}
        {{- with .Values.deployment.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      serviceAccountName: {{ include \"common.serviceAccountName\" . }}
      imagePullSecrets:
        {{- toYaml .Values.image.pullSecrets | nindent 8 }}
      containers:
        - name: {{ .Values.app.name }}
          image: \"{{ .Values.image.registry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}\"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            {{- toYaml .Values.container.ports | nindent 12 }}
          resources:
            {{- toYaml .Values.container.resources | nindent 12 }}
          livenessProbe:
            {{- toYaml .Values.container.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.container.readinessProbe | nindent 12 }}
          env:
            {{- toYaml .Values.container.env | nindent 12 }}
          envFrom:
            {{- toYaml .Values.container.envFrom | nindent 12 }}
```

---

## 5. Configuration Management & Values Resolution

### 5.1 KV Store Key Hierarchy

```
/{environment}/{app-name}/{key}

Examples:
/dev/order-service/replica_count        = 1
/lle/order-service/replica_count        = 2
/prod/order-service/replica_count       = 3
/prod/order-service/cpu_request         = 500m
/prod/order-service/memory_limit        = 1Gi
/prod/order-service/ingress_host        = orders.prod.example.com
/global/order-service/image_registry    = harbor.enterprise.com
/global/order-service/image_repository  = platform/order-service
```

### 5.2 Templated Values File

Each application maintains a **templated values file** stored alongside the Terraform wrapper code. The template uses placeholders resolved at Terraform plan/apply time.

```yaml
# apps/order-service/values.yaml.tpl

global:
  environment: \"${environment}\"
  tenant: \"${tenant}\"
  clusterName: \"${cluster_name}\"

app:
  name: \"order-service\"
  namespace: \"${namespace}\"
  version: \"${app_version}\"

image:
  registry: \"${image_registry}\"
  repository: \"${image_repository}\"
  tag: \"${image_tag}\"
  pullPolicy: \"${image_pull_policy}\"
  pullSecrets:
    - name: \"${image_pull_secret}\"

deployment:
  replicaCount: ${replica_count}
  restartTimestamp: \"${restart_timestamp}\"
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

container:
  resources:
    requests:
      cpu: \"${cpu_request}\"
      memory: \"${memory_request}\"
    limits:
      cpu: \"${cpu_limit}\"
      memory: \"${memory_limit}\"
  env:
    - name: SPRING_PROFILES_ACTIVE
      value: \"${spring_profile}\"
    - name: DB_HOST
      value: \"${db_host}\"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: ${ingress_enabled}
  hosts:
    - host: \"${ingress_host}\"
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: ${autoscaling_enabled}
  minReplicas: ${min_replicas}
  maxReplicas: ${max_replicas}
  targetCPUUtilization: ${target_cpu}

pdb:
  enabled: true
  minAvailable: ${pdb_min_available}
```

### 5.3 Resolution Flow

```
┌──────────────┐    ┌───────────────┐    ┌───────────────┐    ┌──────────┐
│ values.yaml  │    │   Terraform   │    │  KV Config    │    │ Resolved │
│   .tpl       │───►│  templatefile │───►│  Manager      │───►│ values   │
│ (placeholders)│   │  function     │    │  (data source)│    │ .yaml    │
└──────────────┘    └───────────────┘    └───────────────┘    └──────────┘
```

---

## 6. Terraform Enterprise Wrapper Design

### 6.1 Workspace Strategy

```
Workspace Naming:   cd-{app-name}-{environment}
Examples:           cd-order-service-dev
                    cd-order-service-lle
                    cd-order-service-prod

VCS Repo Structure:
terraform-cd-wrappers/
├── modules/
│   └── helm-deploy/              # Reusable module
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
├── apps/
│   └── order-service/
│       ├── main.tf               # Calls module
│       ├── variables.tf
│       ├── backend.tf
│       ├── values.yaml.tpl       # Templated values
│       └── terraform.auto.tfvars # Static defaults
└── policies/                     # Sentinel policies
    ├── allowed-registries.sentinel
    ├── required-labels.sentinel
    └── namespace-enforcement.sentinel
```

### 6.2 Reusable Helm Deploy Module

```hcl
# modules/helm-deploy/main.tf

terraform {
  required_providers {
    helm = {
      source  = \"hashicorp/helm\"
      version = \"~> 2.12\"
    }
    consul = {                         # Or your KV provider
      source  = \"hashicorp/consul\"
      version = \"~> 2.20\"
    }
  }
}

# ─────────────────────────────────────
# Fetch values from KV Config Manager
# ─────────────────────────────────────
data \"consul_keys\" \"app_config\" {
  key {
    name    = \"replica_count\"
    path    = \"${var.environment}/${var.app_name}/replica_count\"
    default = \"2\"
  }
  key {
    name    = \"cpu_request\"
    path    = \"${var.environment}/${var.app_name}/cpu_request\"
    default = \"100m\"
  }
  key {
    name    = \"memory_request\"
    path    = \"${var.environment}/${var.app_name}/memory_request\"
    default = \"256Mi\"
  }
  key {
    name    = \"cpu_limit\"
    path    = \"${var.environment}/${var.app_name}/cpu_limit\"
    default = \"500m\"
  }
  key {
    name    = \"memory_limit\"
    path    = \"${var.environment}/${var.app_name}/memory_limit\"
    default = \"512Mi\"
  }
  key {
    name    = \"ingress_host\"
    path    = \"${var.environment}/${var.app_name}/ingress_host\"
    default = \"\"
  }
  key {
    name    = \"ingress_enabled\"
    path    = \"${var.environment}/${var.app_name}/ingress_enabled\"
    default = \"false\"
  }
  key {
    name    = \"db_host\"
    path    = \"${var.environment}/${var.app_name}/db_host\"
    default = \"\"
  }
  key {
    name    = \"spring_profile\"
    path    = \"${var.environment}/${var.app_name}/spring_profile\"
    default = var.environment
  }
  key {
    name    = \"autoscaling_enabled\"
    path    = \"${var.environment}/${var.app_name}/autoscaling_enabled\"
    default = \"false\"
  }
  key {
    name    = \"min_replicas\"
    path    = \"${var.environment}/${var.app_name}/min_replicas\"
    default = \"2\"
  }
  key {
    name    = \"max_replicas\"
    path    = \"${var.environment}/${var.app_name}/max_replicas\"
    default = \"10\"
  }
  key {
    name    = \"target_cpu\"
    path    = \"${var.environment}/${var.app_name}/target_cpu\"
    default = \"70\"
  }
  key {
    name    = \"pdb_min_available\"
    path    = \"${var.environment}/${var.app_name}/pdb_min_available\"
    default = \"1\"
  }
  key {
    name    = \"image_pull_secret\"
    path    = \"global/${var.app_name}/image_pull_secret\"
    default = \"harbor-pull-secret\"
  }
  key {
    name    = \"image_registry\"
    path    = \"global/${var.app_name}/image_registry\"
    default = \"harbor.enterprise.com\"
  }
  key {
    name    = \"image_repository\"
    path    = \"global/${var.app_name}/image_repository\"
    default = \"\"
  }
}

# ─────────────────────────────────────
# Resolve templated values file
# ─────────────────────────────────────
locals {
  resolved_values = templatefile(\"${var.values_template_path}\", {
    environment          = var.environment
    tenant               = var.tenant
    cluster_name         = var.cluster_name
    namespace            = var.namespace
    app_version          = var.app_version
    image_registry       = data.consul_keys.app_config.var.image_registry
    image_repository     = data.consul_keys.app_config.var.image_repository
    image_tag            = var.image_tag
    image_pull_policy    = var.image_pull_policy
    image_pull_secret    = data.consul_keys.app_config.var.image_pull_secret
    replica_count        = data.consul_keys.app_config.var.replica_count
    restart_timestamp    = var.restart_timestamp
    cpu_request          = data.consul_keys.app_config.var.cpu_request
    memory_request       = data.consul_keys.app_config.var.memory_request
    cpu_limit            = data.consul_keys.app_config.var.cpu_limit
    memory_limit         = data.consul_keys.app_config.var.memory_limit
    ingress_enabled      = data.consul_keys.app_config.var.ingress_enabled
    ingress_host         = data.consul_keys.app_config.var.ingress_host
    db_host              = data.consul_keys.app_config.var.db_host
    spring_profile       = data.consul_keys.app_config.var.spring_profile
    autoscaling_enabled  = data.consul_keys.app_config.var.autoscaling_enabled
    min_replicas         = data.consul_keys.app_config.var.min_replicas
    max_replicas         = data.consul_keys.app_config.var.max_replicas
    target_cpu           = data.consul_keys.app_config.var.target_cpu
    pdb_min_available    = data.consul_keys.app_config.var.pdb_min_available
  })
}

# ─────────────────────────────────────
# Helm Release
# ─────────────────────────────────────
resource \"helm_release\" \"app\" {
  name             = var.app_name
  namespace        = var.namespace
  create_namespace = var.create_namespace

  repository = var.helm_chart_repository
  chart      = var.helm_chart_name
  version    = var.helm_chart_version

  values = [local.resolved_values]

  timeout         = var.helm_timeout
  atomic          = true
  cleanup_on_fail = true
  wait            = true
  wait_for_jobs   = true

  # Force resource update in Scenario 4 (CD-only repave)
  force_update = var.force_update

  lifecycle {
    # Prevent accidental destruction
    prevent_destroy = false
  }
}
```

### 6.3 Module Variables

```hcl
# modules/helm-deploy/variables.tf

variable \"app_name\" {
  type        = string
  description = \"Application name\"
}

variable \"environment\" {
  type        = string
  description = \"Target environment: dev, lle, prod\"
  validation {
    condition     = contains([\"dev\", \"lle\", \"prod\"], var.environment)
    error_message = \"Environment must be dev, lle, or prod.\"
  }
}

variable \"tenant\" {
  type        = string
  description = \"Tenant/team identifier\"
}

variable \"namespace\" {
  type        = string
  description = \"Kubernetes namespace\"
}

variable \"cluster_name\" {
  type        = string
  description = \"Target Kubernetes cluster name\"
}

variable \"image_tag\" {
  type        = string
  description = \"Container image tag to deploy\"
}

variable \"app_version\" {
  type        = string
  description = \"Application semantic version\"
  default     = \"\"
}

variable \"image_pull_policy\" {
  type        = string
  description = \"Image pull policy\"
  default     = \"IfNotPresent\"
}

variable \"restart_timestamp\" {
  type        = string
  description = \"Timestamp to force pod restart (Scenario 4)\"
  default     = \"\"
}

variable \"force_update\" {
  type        = bool
  description = \"Force Helm update even if no changes detected\"
  default     = false
}

variable \"create_namespace\" {
  type        = bool
  description = \"Create namespace if it does not exist\"
  default     = false
}

variable \"values_template_path\" {
  type        = string
  description = \"Path to the templated values.yaml.tpl file\"
}

variable \"helm_chart_repository\" {
  type        = string
  description = \"Helm chart repository URL\"
  default     = \"https://charts.enterprise.com\"
}

variable \"helm_chart_name\" {
  type        = string
  description = \"Helm chart name\"
  default     = \"common-app-chart\"
}

variable \"helm_chart_version\" {
  type        = string
  description = \"Helm chart version\"
}

variable \"helm_timeout\" {
  type        = number
  description = \"Helm operation timeout in seconds\"
  default     = 600
}
```

### 6.4 App-Specific Wrapper (Thin Caller)

```hcl
# apps/order-service/main.tf

module \"deploy\" {
  source = \"../../modules/helm-deploy\"

  app_name             = \"order-service\"
  environment          = var.environment
  tenant               = var.tenant
  namespace            = \"${var.tenant}-${var.environment}\"
  cluster_name         = var.cluster_name
  image_tag            = var.image_tag
  app_version          = var.app_version
  image_pull_policy    = var.image_pull_policy
  restart_timestamp    = var.restart_timestamp
  force_update         = var.force_update
  create_namespace     = false
  values_template_path = \"${path.module}/values.yaml.tpl\"
  helm_chart_repository = var.helm_chart_repository
  helm_chart_name       = \"common-app-chart\"
  helm_chart_version    = var.helm_chart_version
}
```

### 6.5 Sentinel Policy Examples

```python
# policies/allowed-registries.sentinel
import \"tfplan/v2\" as tfplan

allowed_registries = [\"harbor.enterprise.com\", \"registry.enterprise.io\"]

helm_releases = filter tfplan.resource_changes as _, rc {
    rc.type is \"helm_release\" and
    rc.mode is \"managed\" and
    (rc.change.actions contains \"create\" or rc.change.actions contains \"update\")
}

validate_registry = rule {
    all helm_releases as _, release {
        any allowed_registries as registry {
            release.change.after.values[0] contains registry
        }
    }
}

main = rule { validate_registry }
```

---

## 7. XL Release Template Design

### 7.1 Template Hierarchy

```
XLR Templates/
├── Container-CD-Master/                    # Master template (orchestrator)
│   ├── Scenario-Router                     # Routes to correct sub-template
│   └── Status-Aggregator
├── Container-CD-DEV/                       # DEV deployment
│   ├── Pre-Deploy-Validation
│   ├── TFE-Plan
│   ├── TFE-Apply
│   ├── Post-Deploy-Smoke-Test
│   └── Notify
├── Container-CD-LLE/                       # LLE deployment
│   ├── Pre-Deploy-Validation
│   ├── Change-Request-Check
│   ├── TFE-Plan
│   ├── Approval-Gate (auto or manual)
│   ├── TFE-Apply
│   ├── Post-Deploy-Integration-Test
│   ├── Performance-Gate
│   └── Notify
└── Container-CD-PROD/                      # PROD deployment
    ├── Pre-Deploy-Validation
    ├── Change-Request-Validation
    ├── Pre-Deploy-Snapshot
    ├── TFE-Plan
    ├── Approval-Gate (manual)
    ├── TFE-Apply
    ├── Post-Deploy-Smoke-Test
    ├── Canary-Validation (if applicable)
    ├── Rollback-Decision-Gate
    └── Notify
```

### 7.2 Master Template Variables

| Variable | Type | Description | Set By |
|----------|------|-------------|--------|
| `app_name` | String | Application identifier | Jenkins |
| `image_tag` | String | Container image tag | Jenkins |
| `environment` | String | Target environment | Jenkins |
| `scenario` | Enum | `new_deploy`, `image_update`, `base_repave`, `cd_repave` | Jenkins |
| `tenant` | String | Team/BU identifier | Jenkins |
| `helm_chart_version` | String | Common chart version | Jenkins/Default |
| `restart_timestamp` | String | Timestamp for repave (Scenario 4) | Jenkins (auto) |
| `force_update` | Boolean | Force Helm update | Jenkins (auto per scenario) |
| `app_version` | String | App semantic version | Jenkins |
| `previous_image_tag` | String | Rollback reference | XLR (auto-captured) |
| `change_ticket` | String | Change/ServiceNow ticket | Jenkins/Manual |

### 7.3 XLR DEV Template Detail

```
┌─────────────────────────────────────────────────────────────────┐
│                    Container-CD-DEV Template                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Phase 1: Pre-Deployment                                         │
│  ├── [Script] Validate input parameters                         │
│  ├── [Script] Verify image exists in registry                   │
│  ├── [Script] Check image vulnerability scan status             │
│  ├── [Script] Validate KV config entries exist for DEV          │
│  └── [Script] Verify namespace exists in target cluster         │
│                                                                  │
│  Phase 2: Terraform Plan                                         │
│  ├── [TFE] Set workspace variables:                             │
│  │         environment   = \"dev\"                                │
│  │         image_tag     = \"${image_tag}\"                       │
│  │         restart_timestamp = \"${restart_timestamp}\"            │
│  │         force_update  = ${force_update}                      │
│  │         helm_chart_version = \"${helm_chart_version}\"         │
│  ├── [TFE] Trigger Plan on cd-${app_name}-dev                  │
│  ├── [TFE] Wait for Plan completion                             │
│  └── [Script] Parse and log plan output                         │
│                                                                  │
│  Phase 3: Terraform Apply                                        │
│  ├── [TFE] Confirm Apply                                        │
│  ├── [TFE] Wait for Apply completion                            │
│  └── [Script] Capture apply output (release revision)           │
│                                                                  │
│  Phase 4: Post-Deployment                                        │
│  ├── [Script] Verify deployment rollout status (kubectl)        │
│  ├── [Script] Run smoke tests                                   │
│  ├── [Script] Validate pod health and readiness                 │
│  └── [Notification] Notify team (Slack/Email)                   │
│                                                                  │
│  Phase 5: Capture Metadata                                       │
│  ├── [Script] Record deployed image tag in release vars         │
│  ├── [Script] Record Helm revision number                       │
│  └── [Script] Update deployment tracker/CMDB                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4 XLR PROD Template Detail (Additional Gates)

```
┌─────────────────────────────────────────────────────────────────┐
│                   Container-CD-PROD Template                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Phase 1: Pre-Deployment                                         │
│  ├── [Script] Validate input parameters                         │
│  ├── [Script] Verify SAME image tag was deployed to LLE         │
│  ├── [Script] Verify LLE tests passed                           │
│  ├── [Script] Validate change ticket approved (ServiceNow)      │
│  ├── [Script] Check deployment window (maintenance calendar)    │
│  └── [Script] Capture current PROD image tag as rollback ref    │
│                                                                  │
│  Phase 2: Terraform Plan                                         │
│  ├── [TFE] Set workspace variables                              │
│  ├── [TFE] Trigger Plan on cd-${app_name}-prod                 │
│  ├── [TFE] Wait for Plan completion                             │
│  ├── [Script] Log plan diff                                     │
│  └── [Gate] Sentinel policy check pass                          │
│                                                                  │
│  Phase 3: Approval                                               │
│  └── [Manual Gate] Prod deployment approval                     │
│       Approvers: ${tenant}-prod-approvers group                 │
│       Timeout:   4 hours                                        │
│                                                                  │
│  Phase 4: Terraform Apply                                        │
│  ├── [Script] Create pre-deploy snapshot tag                    │
│  ├── [TFE] Confirm Apply                                        │
│  ├── [TFE] Wait for Apply completion                            │
│  └── [Script] Capture apply output                              │
│                                                                  │
│  Phase 5: Post-Deployment Validation                             │
│  ├── [Script] Verify deployment rollout status                  │
│  ├── [Script] Run production smoke tests                        │
│  ├── [Script] Check error rate metrics (Prometheus/Datadog)     │
│  ├── [Script] Check latency metrics                             │
│  └── [Gate] Automated health check pass/fail                    │
│                                                                  │
│  Phase 6: Rollback Decision                                      │
│  ├── [Gate] Manual: Confirm deployment healthy OR rollback      │
│  └── [Conditional] If rollback:                                 │
│       ├── [TFE] Set image_tag = ${previous_image_tag}           │
│       ├── [TFE] Trigger Plan + Apply (rollback)                 │
│       └── [Notification] Alert rollback executed                │
│                                                                  │
│  Phase 7: Finalize                                               │
│  ├── [Notification] Notify stakeholders                         │
│  ├── [Script] Update CMDB/deployment tracker                    │
│  └── [Script] Close change ticket                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Jenkins Pipeline Design

### 8.1 Shared Library Structure

```
jenkins-shared-library/
├── vars/
│   ├── containerCD.groovy              # Main entry point
│   ├── triggerXLR.groovy               # XLR API wrapper
│   ├── validateCDParams.groovy         # Parameter validation
│   └── notifyCD.groovy                 # Notification helper
└── src/
    └── com/enterprise/cd/
        ├── CDScenario.groovy           # Scenario enum
        ├── XLRClient.groovy            # XLR REST client
        └── ImageValidator.groovy       # Registry validation
```

### 8.2 Main Pipeline (Declarative)

```groovy
// Jenkinsfile - Container CD Pipeline

@Library('enterprise-cd-shared-lib') _

pipeline {
    agent { label 'cd-agent' }

    parameters {
        choice(name: 'SCENARIO', choices: ['new_deploy', 'image_update', 'base_repave', 'cd_repave'],
               description: 'CD Scenario')
        string(name: 'APP_NAME', description: 'Application name')
        string(name: 'TENANT', description: 'Tenant/Team identifier')
        string(name: 'IMAGE_TAG', defaultValue: '', 
               description: 'Image tag (leave empty for cd_repave)')
        string(name: 'APP_VERSION', defaultValue: '', 
               description: 'Application semantic version')
        choice(name: 'TARGET_ENV', choices: ['dev', 'lle', 'prod'],
               description: 'Target environment')
        string(name: 'HELM_CHART_VERSION', defaultValue: 'latest',
               description: 'Common Helm chart version')
        string(name: 'CHANGE_TICKET', defaultValue: '',
               description: 'Change ticket number (required for prod)')
        booleanParam(name: 'PROMOTE_THROUGH', defaultValue: false,
               description: 'Auto-promote through all envs (dev→lle→prod)')
    }

    environment {
        XLR_CREDENTIALS = credentials('xlr-api-token')
        XLR_URL         = 'https://xlrelease.enterprise.com'
        REGISTRY_URL    = 'https://harbor.enterprise.com'
    }

    stages {

        // ── STAGE 1: VALIDATE INPUTS ──────────────────────────
        stage('Validate Parameters') {
            steps {
                script {
                    validateCDParams(
                        scenario:       params.SCENARIO,
                        appName:        params.APP_NAME,
                        tenant:         params.TENANT,
                        imageTag:       params.IMAGE_TAG,
                        targetEnv:      params.TARGET_ENV,
                        changeTicket:   params.CHANGE_TICKET
                    )
                }
            }
        }

        // ── STAGE 2: RESOLVE SCENARIO PARAMETERS ──────────────
        stage('Resolve Scenario Parameters') {
            steps {
                script {
                    env.RESOLVED_IMAGE_TAG      = ''
                    env.RESOLVED_RESTART_TS     = ''
                    env.RESOLVED_FORCE_UPDATE   = 'false'
                    env.RESOLVED_PULL_POLICY    = 'IfNotPresent'

                    switch(params.SCENARIO) {
                        case 'new_deploy':
                            // New app: image tag required, standard deploy
                            env.RESOLVED_IMAGE_TAG    = params.IMAGE_TAG
                            env.RESOLVED_PULL_POLICY  = 'Always'
                            echo \"Scenario: NEW DEPLOY - ${params.APP_NAME}:${params.IMAGE_TAG}\"
                            break

                        case 'image_update':
                            // Updated app image: new tag
                            env.RESOLVED_IMAGE_TAG    = params.IMAGE_TAG
                            echo \"Scenario: IMAGE UPDATE - ${params.APP_NAME}:${params.IMAGE_TAG}\"
                            break

                        case 'base_repave':
                            // Same app code, new base image: new tag (rebuilt image)
                            env.RESOLVED_IMAGE_TAG    = params.IMAGE_TAG
                            env.RESOLVED_PULL_POLICY  = 'Always'
                            echo \"Scenario: BASE REPAVE - ${params.APP_NAME}:${params.IMAGE_TAG}\"
                            break

                        case 'cd_repave':
                            // Redeploy existing image: fetch current prod tag
                            env.RESOLVED_IMAGE_TAG    = fetchCurrentImageTag(
                                                            params.APP_NAME, 
                                                            params.TARGET_ENV)
                            env.RESOLVED_RESTART_TS   = new Date().format(
                                                            \"yyyy-MM-dd'T'HH:mm:ss'Z'\", 
                                                            TimeZone.getTimeZone('UTC'))
                            env.RESOLVED_FORCE_UPDATE = 'true'
                            env.RESOLVED_PULL_POLICY  = 'Always'
                            echo \"Scenario: CD-ONLY REPAVE - ${params.APP_NAME}:${env.RESOLVED_IMAGE_TAG} with restartTs=${env.RESOLVED_RESTART_TS}\"
                            break
                    }
                }
            }
        }

        // ── STAGE 3: VERIFY IMAGE (skip for cd_repave with no new image) ──
        stage('Verify Image in Registry') {
            when {
                expression { params.SCENARIO != 'cd_repave' }
            }
            steps {
                script {
                    verifyImageExists(
                        registry:   env.REGISTRY_URL,
                        appName:    params.APP_NAME,
                        imageTag:   env.RESOLVED_IMAGE_TAG
                    )
                }
            }
        }

        // ── STAGE 4: TRIGGER XLR RELEASE ──────────────────────
        stage('Trigger XL Release') {
            steps {
                script {
                    def xlrTemplate = resolveXLRTemplate(params.TARGET_ENV)

                    def releaseId = triggerXLR(
                        xlrUrl:       env.XLR_URL,
                        credentials:  env.XLR_CREDENTIALS,
                        templateId:   xlrTemplate,
                        variables: [
                            'app_name':           params.APP_NAME,
                            'tenant':             params.TENANT,
                            'image_tag':          env.RESOLVED_IMAGE_TAG,
                            'environment':        params.TARGET_ENV,
                            'scenario':           params.SCENARIO,
                            'app_version':        params.APP_VERSION,
                            'helm_chart_version': params.HELM_CHART_VERSION,
                            'restart_timestamp':  env.RESOLVED_RESTART_TS,
                            'force_update':       env.RESOLVED_FORCE_UPDATE,
                            'image_pull_policy':  env.RESOLVED_PULL_POLICY,
                            'change_ticket':      params.CHANGE_TICKET
                        ]
                    )

                    env.XLR_RELEASE_ID = releaseId
                    echo \"XLR Release started: ${releaseId}\"
                }
            }
        }

        // ── STAGE 5: MONITOR XLR RELEASE ──────────────────────
        stage('Monitor Release') {
            steps {
                script {
                    def status = monitorXLRRelease(
                        xlrUrl:      env.XLR_URL,
                        credentials: env.XLR_CREDENTIALS,
                        releaseId:   env.XLR_RELEASE_ID,
                        timeout:     120  // minutes
                    )

                    if (status != 'COMPLETED') {
                        error \"XLR Release failed with status: ${status}\"
                    }
                }
            }
        }

        // ── STAGE 6: PROMOTE (optional) ───────────────────────
        stage('Promote to Next Environment') {
            when {
                allOf {
                    expression { params.PROMOTE_THROUGH == true }
                    expression { params.TARGET_ENV != 'prod' }
                    expression { params.SCENARIO != 'cd_repave' }
                }
            }
            steps {
                script {
                    def nextEnv = (params.TARGET_ENV == 'dev') ? 'lle' : 'prod'
                    echo \"Auto-promoting to ${nextEnv}...\"

                    build job: env.JOB_NAME, parameters: [
                        choice(name: 'SCENARIO',          value: params.SCENARIO),
                        string(name: 'APP_NAME',          value: params.APP_NAME),
                        string(name: 'TENANT',            value: params.TENANT),
                        string(name: 'IMAGE_TAG',         value: env.RESOLVED_IMAGE_TAG),
                        string(name: 'APP_VERSION',       value: params.APP_VERSION),
                        choice(name: 'TARGET_ENV',        value: nextEnv),
                        string(name: 'HELM_CHART_VERSION',value: params.HELM_CHART_VERSION),
                        string(name: 'CHANGE_TICKET',     value: params.CHANGE_TICKET),
                        booleanParam(name: 'PROMOTE_THROUGH', value: true)
                    ], wait: true
                }
            }
        }
    }

    post {
        success {
            notifyCD(status: 'SUCCESS', appName: params.APP_NAME, 
                     env: params.TARGET_ENV, scenario: params.SCENARIO,
                     imageTag: env.RESOLVED_IMAGE_TAG)
        }
        failure {
            notifyCD(status: 'FAILURE', appName: params.APP_NAME, 
                     env: params.TARGET_ENV, scenario: params.SCENARIO,
                     imageTag: env.RESOLVED_IMAGE_TAG)
        }
    }
}

// ── HELPER FUNCTIONS ──────────────────────────────────────────

def resolveXLRTemplate(String env) {
    def templates = [
        'dev':  'Applications/Container-CD-DEV',
        'lle':  'Applications/Container-CD-LLE',
        'prod': 'Applications/Container-CD-PROD'
    ]
    return templates[env]
}

def fetchCurrentImageTag(String appName, String environment) {
    // Query TFE state or Kubernetes to get current running image tag
    def result = sh(
        script: \"\"\"
            kubectl get deployment ${appName} \\
                -n ${params.TENANT}-${environment} \\
                -o jsonpath='{.spec.template.spec.containers[0].image}' \\
                | awk -F: '{print \\$NF}'
        \"\"\",
        returnStdout: true
    ).trim()

    if (!result) {
        error \"Could not determine current image tag for ${appName} in ${environment}\"
    }
    return result
}
```

---

## 9. CD Scenario 1 — New App Image Deployment

### 9.1 Description

First-time deployment of a containerized application to an environment. The Helm release does not yet exist; all Kubernetes resources will be created fresh.

### 9.2 Prerequisites

| # | Prerequisite | Owner |
|---|-------------|-------|
| 1 | Application image built, scanned, and pushed to registry | CI Pipeline |
| 2 | KV Config Manager populated with all required keys for target env | App Team |
| 3 | TFE workspace created: `cd-{app_name}-{env}` | Platform Team |
| 4 | Kubernetes namespace pre-created with quotas and RBAC | Platform Team |
| 5 | App-specific Terraform wrapper code merged to VCS | App Team |
| 6 | Common Helm chart version validated | Platform Team |
| 7 | Image pull secret exists in target namespace | Platform Team |

### 9.3 Flow

```
┌───────────┐   ┌─────────┐   ┌────────────┐   ┌──────────┐   ┌──────────┐
│  Jenkins   │──►│  XLR    │──►│   TFE      │──►│  Helm    │──►│   K8s    │
│            │   │  DEV    │   │  Plan/     │   │ install  │   │  Create  │
│ SCENARIO:  │   │  Tmpl   │   │  Apply     │   │ (new)    │   │  all     │
│ new_deploy │   │         │   │            │   │          │   │  objects │
│ IMAGE_TAG: │   │ Pre:    │   │ image_tag  │   │          │   │          │
│ v1.0.0     │   │ validate│   │ = v1.0.0   │   │          │   │          │
│            │   │ img scan│   │            │   │          │   │          │
└───────────┘   └─────────┘   └────────────┘   └──────────┘   └──────────┘
```

### 9.4 Scenario-Specific Parameters

```yaml
# Parameters passed from Jenkins → XLR → TFE
scenario:          \"new_deploy\"
image_tag:         \"v1.0.0\"              # First version
restart_timestamp: \"\"                     # Not used
force_update:      false                  # Not needed
image_pull_policy: \"Always\"              # Ensure fresh pull
create_namespace:  false                  # Pre-created by platform
```

### 9.5 TFE Behavior

- `terraform plan` shows all resources as **new additions** (`+` create)
- `helm_release` resource is created for the first time
- TFE state is initialized with the Helm release metadata
- Sentinel validates image registry, namespace, and required labels

### 9.6 Helm Behavior

- `helm install` (first release)
- All Kubernetes resources created: Deployment, Service, Ingress, HPA, PDB, ServiceAccount, etc.
- Deployment initially creates pods pulling `image_tag: v1.0.0`

### 9.7 Validation Checklist

| Check | Method | Automated |
|-------|--------|-----------|
| All pods running and ready | `kubectl rollout status` | Yes |
| Service endpoint responding | HTTP health check | Yes |
| Image tag matches expected | `kubectl get pods -o jsonpath` | Yes |
| Resource limits applied | `kubectl describe pod` | Yes |
| Namespace correct | Helm release metadata | Yes |
| Smoke test pass | Configurable test suite | Yes |

---

## 10. CD Scenario 2 — App Image Update Deployment

### 10.1 Description

Deploy a new version of the application image. Application code has changed, resulting in a new image tag. The Helm release already exists and will be upgraded.

### 10.2 Prerequisites

| # | Prerequisite | Owner |
|---|-------------|-------|
| 1 | New application image built, scanned, and pushed | CI Pipeline |
| 2 | Existing Helm release in target environment | Previous deployment |
| 3 | KV Config values reviewed (may or may not change) | App Team |
| 4 | TFE workspace exists with prior state | Previous deployment |

### 10.3 Flow

```
┌───────────┐   ┌─────────┐   ┌────────────┐   ┌──────────┐   ┌──────────┐
│  Jenkins   │──►│  XLR    │──►│   TFE      │──►│  Helm    │──►│   K8s    │
│            │   │  LLE/   │   │  Plan:     │   │ upgrade  │   │ Rolling  │
│ SCENARIO:  │   │  PROD   │   │  image_tag │   │ (update) │   │ Update   │
│ image_     │   │  Tmpl   │   │  changed   │   │          │   │          │
│ update     │   │         │   │            │   │  values: │   │ Old pods │
│ IMAGE_TAG: │   │ Gate:   │   │ Apply:     │   │  new tag │   │ replaced │
│ v1.1.0     │   │ LLE pass│   │  update    │   │          │   │ by new   │
│            │   │ approval│   │  helm_rel  │   │          │   │          │
└───────────┘   └─────────┘   └────────────┘   └──────────┘   └──────────┘
```

### 10.4 Scenario-Specific Parameters

```yaml
scenario:          \"image_update\"
image_tag:         \"v1.1.0\"              # New version
restart_timestamp: \"\"                     # Not used — tag change triggers rollout
force_update:      false                  # Tag change is sufficient
image_pull_policy: \"IfNotPresent\"        # Standard — new tag will be pulled
```

### 10.5 TFE Behavior

- `terraform plan` shows `helm_release.app` as **update in-place** (`~`)
- Only the values content changes (new `image_tag`)
- TFE state updated with new Helm release revision
- Plan diff clearly shows: `image.tag: \"v1.0.0\" → \"v1.1.0\"`

### 10.6 Helm Behavior

- `helm upgrade` on existing release
- Kubernetes performs **RollingUpdate**: new ReplicaSet with `v1.1.0`, old ReplicaSet scaled down
- Zero-downtime if `maxUnavailable: 0` and `maxSurge: 1`

### 10.7 What Changes vs What Stays

| Component | Changes? | Details |
|-----------|----------|---------|
| Image tag | ✅ Yes | `v1.0.0` → `v1.1.0` |
| Deployment spec | ✅ Yes | New pod template triggers rollout |
| Service | ❌ No | Selector unchanged |
| Ingress | ❌ No | Unless config changed in KV |
| ConfigMap | ❌ / ✅ | Only if KV values changed |
| HPA / PDB | ❌ No | Unless config changed in KV |

### 10.8 Promotion Path

```
DEV (auto-deploy on CI)  ──►  LLE (auto after DEV smoke)  ──►  PROD (manual approval)
     v1.1.0                        v1.1.0                          v1.1.0

Validation at each stage:
DEV:  Smoke tests only
LLE:  Integration tests + performance baseline
PROD: Smoke tests + metric monitoring + manual confirmation
```

---

## 11. CD Scenario 3 — Repave for Base Image Update

### 11.1 Description

The application's base image (e.g., `openjdk:17-slim`, `node:18-alpine`) has been updated with OS/runtime security patches. The application image is rebuilt with the updated base, producing a **new image tag** with no application code changes. The rebuilt image is then deployed through the standard promotion path.

### 11.2 Trigger Source

```
┌────────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Base Image Update  │────►│  CI Pipeline     │────►│  CD Pipeline     │
│  (CVE patch, OS    │     │  Rebuild app     │     │  (This scenario) │
│   update, runtime  │     │  image with new  │     │                  │
│   patch)           │     │  base; new tag   │     │  SCENARIO:       │
│                    │     │  e.g.,           │     │  base_repave     │
│  harbor webhook    │     │  v1.0.0-base.2   │     │                  │
│  or scheduled scan │     │                  │     │                  │
└────────────────────┘     └──────────────────┘     └──────────────────┘
```

### 11.3 Image Tagging Convention for Base Repave

```
Standard release:    v1.1.0
Base repave:         v1.1.0-base.2    (same app code, 2nd base image rebuild)
                     v1.1.0-base.3    (3rd base image rebuild)

Alternative:         v1.1.0-20240115  (date-based suffix)

The key point: THE TAG IS DIFFERENT from production, even though app code is identical.
```

### 11.4 Scenario-Specific Parameters

```yaml
scenario:          \"base_repave\"
image_tag:         \"v1.1.0-base.2\"       # Rebuilt image with new base
restart_timestamp: \"\"                      # Not needed — tag change triggers rollout
force_update:      false                   # Tag change is sufficient
image_pull_policy: \"Always\"               # Force pull to ensure patched layers
app_version:       \"v1.1.0\"              # App version unchanged
```

### 11.5 Flow

```
┌───────────────┐   ┌─────────────┐   ┌──────────────┐   ┌──────────┐   ┌──────────┐
│  Jenkins       │──►│  XLR        │──►│  TFE         │──►│  Helm    │──►│   K8s    │
│                │   │  DEV→LLE→   │   │  Plan:       │   │ upgrade  │   │ Rolling  │
│ SCENARIO:      │   │  PROD       │   │  image_tag   │   │          │   │ Update   │
│ base_repave    │   │             │   │  changed     │   │ new tag  │   │          │
│ IMAGE_TAG:     │   │ Same gates  │   │              │   │ same     │   │ Pods get │
│ v1.1.0-base.2 │   │ as image_   │   │ Apply:       │   │ config   │   │ patched  │
│                │   │ update      │   │  update      │   │          │   │ base     │
│ APP_VERSION:   │   │             │   │  helm_rel    │   │          │   │          │
│ v1.1.0         │   │ + CVE gate  │   │              │   │          │   │          │
└───────────────┘   └─────────────┘   └──────────────┘   └──────────┘   └──────────┘
```

### 11.6 Additional XLR Gates for Base Repave

```
Phase 0 (Additional for base_repave):
├── [Script] Verify new image resolves CVEs from previous base
├── [Script] Compare image layers — confirm only base layers changed
├── [Script] Validate no application dependency changes
└── [Script] Confirm image scan shows reduced/zero CVE count
```

### 11.7 Differences from Scenario 2 (Image Update)

| Aspect | Scenario 2: Image Update | Scenario 3: Base Repave |
|--------|--------------------------|-------------------------|
| **App code changes** | Yes | No |
| **Base image changes** | Maybe | Yes (primary purpose) |
| **Image tag** | New app version (`v1.2.0`) | Base suffix (`v1.1.0-base.2`) |
| **Trigger** | Developer merge / CI | Base image CVE / scheduled |
| **Testing emphasis** | Functional + integration | Regression + security scan |
| **Rollback urgency** | App issue → rollback to prior version | Only if regression introduced |
| **Approval scrutiny** | Standard | May be expedited for critical CVEs |
| **app_version** | Changes | Unchanged |
| **CVE gate** | Optional | Mandatory |

### 11.8 Batch Repave Strategy

For enterprise-wide base image updates affecting many apps:

```
Jenkins Orchestrator Pipeline (Batch Repave):
├── Input: base_image_name, base_image_new_tag
├── Query: All apps using this base image (from registry metadata)
├── For each app (parallel where possible):
│   ├── Trigger CI rebuild → produces new app image tag
│   └── Trigger CD pipeline (this Scenario 3) with new tag
├── Dashboard: Track repave progress across all apps
└── Report: Compliance status (which apps are on latest base)
```

---

## 12. CD Scenario 4 — CD-Only Repave (Redeploy Existing Production Image)

### 12.1 Description

Restart/redeploy the currently running production image **without any image change**. This forces Kubernetes to recreate all pods, which is useful for:

- Refreshing pod state / clearing accumulated memory issues
- Rotating ephemeral credentials or mounted secrets
- Applying updated ConfigMap/Secret values that require restart
- Compliance requirement for periodic pod rotation
- Validating disaster recovery (prove the image can be re-pulled and pods recreated)

**The image tag does NOT change.** The only mechanism to trigger a rollout is changing the pod template annotation.

### 12.2 The Restart Timestamp Mechanism

Since the image tag is unchanged, Kubernetes will not detect a change in the pod spec. The solution is to update a pod annotation that forces a new ReplicaSet.

```yaml
# In the common Helm chart deployment template:
spec:
  template:
    metadata:
      annotations:
        rollout/restartTimestamp: {{ .Values.deployment.restartTimestamp | default \"\" | quote }}
```

When `restartTimestamp` changes from `\"\"` to `\"2024-01-15T10:30:00Z\"`, Kubernetes sees a pod template change and initiates a rolling update — even though the image is identical.

### 12.3 Flow

```
┌───────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────┐   ┌──────────┐
│  Jenkins       │──►│  XLR         │──►│  TFE         │──►│  Helm    │──►│   K8s    │
│                │   │  PROD Tmpl   │   │  Plan:       │   │ upgrade  │   │ Rolling  │
│ SCENARIO:      │   │              │   │  restart_ts  │   │          │   │ Restart  │
│ cd_repave      │   │ Lighter      │   │  changed     │   │ Same tag │   │          │
│                │   │ gates —      │   │  force=true  │   │ New      │   │ New pods │
│ IMAGE_TAG:     │   │ no image     │   │              │   │ restart  │   │ same     │
│ (auto-fetched  │   │ verify       │   │ Apply:       │   │ ts in    │   │ image    │
│  from current  │   │              │   │  update      │   │ annot.   │   │          │
│  deployment)   │   │              │   │  helm_rel    │   │          │   │          │
│                │   │              │   │              │   │          │   │          │
│ restart_ts:    │   │              │   │              │   │          │   │          │
│ 2024-01-15T..  │   │              │   │              │   │          │   │          │
│ force_update:  │   │              │   │              │   │          │   │          │
│ true           │   │              │   │              │   │          │   │          │
└───────────────┘   └──────────────┘   └──────────────┘   └──────────┘   └──────────┘
```

### 12.4 Scenario-Specific Parameters

```yaml
scenario:          \"cd_repave\"
image_tag:         \"v1.1.0\"               # AUTO-FETCHED from current deployment
restart_timestamp: \"2024-01-15T10:30:00Z\" # AUTO-GENERATED by Jenkins
force_update:      true                    # Force Helm to update even if it thinks nothing changed
image_pull_policy: \"Always\"               # Force re-pull to verify image availability
app_version:       \"v1.1.0\"              # Unchanged
```

### 12.5 Jenkins Auto-Resolution Logic

```groovy
case 'cd_repave':
    // 1. Fetch current image tag from running deployment
    env.RESOLVED_IMAGE_TAG = fetchCurrentImageTag(params.APP_NAME, params.TARGET_ENV)
    
    // 2. Generate unique restart timestamp
    env.RESOLVED_RESTART_TS = new Date().format(\"yyyy-MM-dd'T'HH:mm:ss'Z'\", 
                                                 TimeZone.getTimeZone('UTC'))
    
    // 3. Force Helm to process update
    env.RESOLVED_FORCE_UPDATE = 'true'
    
    // 4. Always pull to prove image is still available
    env.RESOLVED_PULL_POLICY = 'Always'
    
    echo \"\"\"
    CD-Only Repave:
      App:           ${params.APP_NAME}
      Current Tag:   ${env.RESOLVED_IMAGE_TAG} (no change)
      Restart TS:    ${env.RESOLVED_RESTART_TS} (new — triggers rollout)
      Force Update:  true
      Pull Policy:   Always (verify image pullability)
    \"\"\"
    break
```

### 12.6 TFE Plan Output (Expected)

```
Terraform will perform the following actions:

  # module.deploy.helm_release.app will be updated in-place
  ~ resource \"helm_release\" \"app\" {
        name       = \"order-service\"
        namespace  = \"team-a-prod\"
      ~ values     = [
          ~ <<-EOT
              ...
              deployment:
            -   restartTimestamp: \"2024-01-10T08:00:00Z\"
            +   restartTimestamp: \"2024-01-15T10:30:00Z\"
              ...
            EOT
        ]
        # (all other attributes unchanged)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

### 12.7 XLR Template Variations for CD-Only Repave

```
┌─────────────────────────────────────────────────────────────────┐
│              Container-CD-PROD (cd_repave variant)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Phase 1: Pre-Deployment (LIGHTER)                               │
│  ├── [Script] Validate app exists in target env                 │
│  ├── [Script] Fetch current image tag (verify match)            │
│  ├── [SKIP]   Image registry verification (no new image)        │
│  ├── [SKIP]   Image vulnerability scan (no new image)           │
│  ├── [Script] Validate change ticket (if required by policy)    │
│  └── [Script] Record pre-repave pod names and status            │
│                                                                  │
│  Phase 2: Terraform Plan                                         │
│  ├── [TFE] Set workspace variables (restart_ts, force_update)   │
│  ├── [TFE] Trigger Plan                                         │
│  ├── [Script] Verify plan only shows restartTimestamp change     │
│  └── [Script] ASSERT: image_tag is UNCHANGED in plan            │
│                                                                  │
│  Phase 3: Approval (may be lighter for repave)                   │
│  └── [Gate] Based on policy:                                    │
│       - Scheduled repave: Auto-approved                          │
│       - Manual repave: Requires approval                         │
│                                                                  │
│  Phase 4: Terraform Apply                                        │
│  ├── [TFE] Apply                                                │
│  └── [Script] Capture Helm revision                             │
│                                                                  │
│  Phase 5: Post-Deployment                                        │
│  ├── [Script] Verify rollout completes                          │
│  ├── [Script] Verify ALL pods are new (different pod names)     │
│  ├── [Script] Verify pod image matches expected tag             │
│  ├── [Script] Run smoke tests                                   │
│  └── [Script] Verify no error rate increase                     │
│                                                                  │
│  Phase 6: Finalize                                               │
│  └── [Script] Record successful repave with timestamp           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.8 Scheduled Repave (Automation)

```groovy
// Jenkins scheduled pipeline for periodic CD repave
// Cron: every Sunday at 2 AM UTC

pipeline {
    triggers {
        cron('0 2 * * 0')
    }
    stages {
        stage('Batch CD Repave') {
            steps {
                script {
                    def apps = getAppsRequiringRepave()  // From CMDB or config
                    apps.each { app ->
                        build job: 'container-cd-pipeline', parameters: [
                            choice(name: 'SCENARIO',    value: 'cd_repave'),
                            string(name: 'APP_NAME',    value: app.name),
                            string(name: 'TENANT',      value: app.tenant),
                            choice(name: 'TARGET_ENV',  value: 'prod'),
                            string(name: 'CHANGE_TICKET', value: \"AUTO-REPAVE-${new Date().format('yyyyMMdd')}\")
                        ], wait: false  // Parallel execution
                    }
                }
            }
        }
    }
}
```

---

## 13. Environment Promotion Strategy

### 13.1 Promotion Matrix

```
              ┌────────┐       ┌────────┐       ┌────────┐
              │  DEV   │──────►│  LLE   │──────►│  PROD  │
              └────────┘       └────────┘       └────────┘
                  │                │                 │
Trigger:     Auto (CI)     Auto/Manual         Manual Only
Approval:    None          Team Lead           Change Board
Gate:        Smoke test    Integration +       Smoke + Metrics
                           Performance         + Manual Confirm
TFE Workspace: cd-app-dev  cd-app-lle         cd-app-prod
Sentinel:    Basic          Standard           Strict
```

### 13.2 Image Tag Traceability

```
┌─────────────────────────────────────────────────────────────────┐
│  Image Tag Promotion Tracking (recorded in XLR + CMDB)          │
├──────────┬──────────────┬──────────────┬───────────────────────┤
│ Scenario │ DEV Tag      │ LLE Tag      │ PROD Tag              │
├──────────┼──────────────┼──────────────┼───────────────────────┤
│ Scen 1   │ v1.0.0       │ v1.0.0       │ v1.0.0               │
│ Scen 2   │ v1.1.0       │ v1.1.0       │ v1.1.0               │
│ Scen 3   │ v1.1.0-base.2│ v1.1.0-base.2│ v1.1.0-base.2       │
│ Scen 4   │ N/A          │ N/A          │ v1.1.0-base.2 (same) │
│          │              │              │ (only restartTs new)   │
└──────────┴──────────────┴──────────────┴───────────────────────┘
```

### 13.3 Environment Parity Enforcement

XLR PROD template validates that the image being deployed to PROD was previously successfully deployed to LLE:

```
[Script] Pre-Deploy Check:
  1. Query XLR: Get last successful LLE release for ${app_name}
  2. Extract LLE deployed image tag
  3. ASSERT: LLE_image_tag == requested PROD image_tag
  4. If mismatch → FAIL with: \"Image v1.2.0 has not been validated in LLE. 
     LLE is running v1.1.0. Deploy to LLE first.\"
```

---

## 14. Rollback Strategy

### 14.1 Rollback Methods by Scenario

| Scenario | Rollback Method | Details |
|----------|----------------|---------|
| **1. New Deploy** | `terraform destroy` OR `helm uninstall` | Remove all resources |
| **2. Image Update** | Re-deploy previous tag | TFE Apply with `image_tag = previous_tag` |
| **3. Base Repave** | Re-deploy previous tag | TFE Apply with `image_tag = previous_base_tag` |
| **4. CD-Only Repave** | No rollback needed (same image) | If pod issues, investigate pod-level — image hasn't changed |

### 14.2 Automatic Rollback (Helm Atomic)

```hcl
resource \"helm_release\" \"app\" {
  ...
  atomic          = true      # Auto-rollback on failed install/upgrade
  cleanup_on_fail = true      # Clean up new resources on failure
  wait            = true      # Wait for resources to be ready
  timeout         = 600       # 10 minute timeout
}
```

### 14.3 Manual Rollback via Jenkins

```groovy
// Rollback pipeline — triggered manually or from XLR
pipeline {
    parameters {
        string(name: 'APP_NAME')
        string(name: 'TARGET_ENV')
        string(name: 'ROLLBACK_IMAGE_TAG', description: 'Previous image tag to restore')
    }
    stages {
        stage('Rollback') {
            steps {
                script {
                    // Trigger standard CD with the previous tag
                    build job: 'container-cd-pipeline', parameters: [
                        choice(name: 'SCENARIO',    value: 'image_update'),
                        string(name: 'APP_NAME',    value: params.APP_NAME),
                        string(name: 'IMAGE_TAG',   value: params.ROLLBACK_IMAGE_TAG),
                        choice(name: 'TARGET_ENV',  value: params.TARGET_ENV)
                    ], wait: true
                }
            }
        }
    }
}
```

### 14.4 TFE State-Based Rollback

```
Previous State (in TFE):
  helm_release.app:
    values.image.tag = \"v1.0.0\"   ← TFE knows this was the last applied state

Rollback Process:
  1. XLR sets TFE variable: image_tag = \"v1.0.0\"  (captured from pre-deploy phase)
  2. TFE Plan shows: image.tag: \"v1.1.0\" → \"v1.0.0\"
  3. TFE Apply → Helm upgrade → RollingUpdate back to v1.0.0
```

---

## 15. Security & Governance

### 15.1 RBAC Matrix

| Role | Jenkins Trigger | XLR Approval | TFE Apply | K8s Namespace |
|------|----------------|--------------|-----------|---------------|
| Developer | DEV only | N/A | N/A (auto) | DEV: edit |
| Tech Lead | DEV, LLE | LLE approve | N/A | LLE: view |
| Release Manager | DEV, LLE, PROD | PROD approve | PROD confirm | PROD: view |
| Platform Admin | All | All | All | All: admin |
| Service Account (CI) | DEV (auto) | N/A | N/A | N/A |

### 15.2 Sentinel Policies (TFE)

| Policy | Enforcement | Description |
|--------|-------------|-------------|
| `allowed-registries` | Hard Mandatory | Images must come from approved registries |
| `required-labels` | Hard Mandatory | Tenant, app, environment labels required |
| `namespace-enforcement` | Hard Mandatory | Deployment to correct namespace only |
| `resource-limits` | Soft Mandatory | CPU/memory limits within allowed range |
| `no-privileged` | Hard Mandatory | No privileged containers |
| `image-tag-no-latest` | Hard Mandatory | `latest` tag prohibited |
| `prod-change-ticket` | Hard Mandatory | PROD deployments require change ticket |

### 15.3 Image Provenance

```
Registry → Notary/Cosign Signature Verification → TFE Sentinel → Deploy

Pre-Deploy Check (XLR Script):
  1. Verify image signature (cosign verify)
  2. Verify image SBOM exists
  3. Verify CVE scan results below threshold:
     - PROD: Zero Critical, Zero High
     - LLE:  Zero Critical
     - DEV:  Warning only
```

---

## 16. Observability & Audit

### 16.1 Deployment Tracking

Every deployment records the following to a central deployment tracker (CMDB / deployment database):

```json
{
  \"deployment_id\": \"uuid\",
  \"app_name\": \"order-service\",
  \"tenant\": \"team-a\",
  \"environment\": \"prod\",
  \"scenario\": \"image_update\",
  \"image_tag\": \"v1.1.0\",
  \"previous_image_tag\": \"v1.0.0\",
  \"helm_chart_version\": \"3.2.1\",
  \"helm_revision\": 5,
  \"restart_timestamp\": null,
  \"jenkins_build\": \"container-cd-pipeline#142\",
  \"xlr_release_id\": \"Release-12345\",
  \"tfe_run_id\": \"run-abc123\",
  \"change_ticket\": \"CHG0012345\",
  \"triggered_by\": \"john.doe@enterprise.com\",
  \"approved_by\": \"jane.smith@enterprise.com\",
  \"deployed_at\": \"2024-01-15T10:35:00Z\",
  \"rollout_duration_seconds\": 120,
  \"status\": \"SUCCESS\",
  \"rollback\": false,
  \"kv_config_snapshot\": { \"replica_count\": \"3\", \"cpu_limit\": \"500m\" }
}
```

### 16.2 Metrics & Dashboards

| Metric | Source | Dashboard |
|--------|--------|-----------|
| Deployment frequency per app | Jenkins/XLR | Grafana |
| Deployment lead time | Jenkins start → K8s rollout complete | Grafana |
| Deployment failure rate | XLR status | Grafana |
| Mean time to recovery (rollback) | XLR rollback events | Grafana |
| Repave compliance (Scenario 4 frequency) | Deployment tracker | Compliance dashboard |
| Base image currency (Scenario 3 lag) | Registry + Deployment tracker | Security dashboard |
| Approval wait time | XLR gate durations | Grafana |

### 16.3 Audit Trail

```
Complete audit chain:
  Jenkins Build Log (who triggered, parameters)
    └─► XLR Release (approval records, gate results)
        └─► TFE Run (plan output, sentinel results, apply log)
            └─► Helm Release History (revision, values diff)
                └─► K8s Events (pod scheduling, image pull, readiness)
```

---

## 17. Appendices

### Appendix A: Scenario Comparison Matrix

| Attribute | Scenario 1: New Deploy | Scenario 2: Image Update | Scenario 3: Base Repave | Scenario 4: CD-Only Repave |
|-----------|----------------------|--------------------------|-------------------------|---------------------------|
| **Image tag changes** | N/A (first) | ✅ Yes (new app ver) | ✅ Yes (new base ver) | ❌ No |
| **App code changes** | N/A (first) | ✅ Yes | ❌ No | ❌ No |
| **Base image changes** | N/A (first) | Optional | ✅ Yes | ❌ No |
| **Helm operation** | `install` | `upgrade` | `upgrade` | `upgrade` |
| **K8s rollout trigger** | New Deployment | Image tag change | Image tag change | Annotation change |
| **restart_timestamp** | Not set | Not set | Not set | Set (new) |
| **force_update** | `false` | `false` | `false` | `true` |
| **image_pull_policy** | `Always` | `IfNotPresent` | `Always` | `Always` |
| **Typical trigger** | Manual / onboarding | CI completion | Base image CVE | Schedule / manual |
| **CI involved** | Yes (first build) | Yes (new build) | Yes (rebuild) | **No** |
| **Image scan gate** | Yes | Yes | Yes (mandatory) | No (skip) |
| **Promotion path** | DEV→LLE→PROD | DEV→LLE→PROD | DEV→LLE→PROD | Per-environment (typically PROD only) |
| **Rollback method** | Delete release | Previous image tag | Previous image tag | N/A (same image) |

### Appendix B: TFE Workspace Variable Mapping

| TFE Variable | Scenario 1 | Scenario 2 | Scenario 3 | Scenario 4 |
|-------------|------------|------------|------------|------------|
| `image_tag` | `v1.0.0` | `v1.1.0` | `v1.1.0-base.2` | `v1.1.0-base.2` (unchanged) |
| `restart_timestamp` | `\"\"` | `\"\"` | `\"\"` | `2024-01-15T10:30:00Z` |
| `force_update` | `false` | `false` | `false` | `true` |
| `image_pull_policy` | `Always` | `IfNotPresent` | `Always` | `Always` |
| `app_version` | `v1.0.0` | `v1.1.0` | `v1.1.0` | `v1.1.0` |
| `helm_chart_version` | `3.2.1` | `3.2.1` | `3.2.1` | `3.2.1` |

### Appendix C: Onboarding Checklist (New App to Platform)

```
□  1. App team creates Dockerfile using approved base image
□  2. CI pipeline created (image build, scan, push)
□  3. KV Config Manager populated for all environments
□  4. App-specific Terraform wrapper created:
       □  main.tf (calls helm-deploy module)
       □  variables.tf
       □  values.yaml.tpl (custom values template)
□  5. TFE workspaces created:
       □  cd-{app}-dev
       □  cd-{app}-lle
       □  cd-{app}-prod
□  6. TFE workspace variables configured (credentials, cluster endpoints)
□  7. Kubernetes namespaces created with RBAC and quotas
□  8. Image pull secrets deployed to namespaces
□  9. Jenkins pipeline parameterized for the new app
□ 10. XLR templates configured (or verify shared templates work)
□ 11. First deployment executed (Scenario 1) to DEV
□ 12. Smoke tests configured and passing
□ 13. Promotion to LLE and PROD validated
□ 14. Rollback tested
□ 15. CD-only repave validated (Scenario 4)
```

### Appendix D: Failure Modes and Recovery

| Failure Point | Detection | Recovery |
|---------------|-----------|----------|
| Jenkins fails to trigger XLR | Jenkins error log | Re-trigger Jenkins job |
| XLR fails to trigger TFE | XLR task failure | XLR retry mechanism / manual restart |
| TFE Plan fails (config error) | TFE run output | Fix config in KV store or wrapper, re-run |
| TFE Sentinel policy blocks | TFE policy check | Fix violation, re-plan |
| TFE Apply fails | TFE run output | Investigate error, re-apply |
| Helm timeout (pods not ready) | Helm `atomic` rollback | Investigate pod events, fix, re-deploy |
| K8s rollout stuck | `kubectl rollout status` timeout | `kubectl rollout undo` or Helm rollback |
| Post-deploy smoke test fails | XLR test task | Trigger rollback pipeline |
| Image pull fails (registry issue) | K8s events (`ErrImagePull`) | Verify registry connectivity, image exists |
| KV Config Manager unavailable | TFE plan failure | Retry after KV service restored |

---

*Document Version: 1.0*
*Last Updated: January 2025*
*Owner: Platform Engineering — CI/CD Team*
*Review Cycle: Quarterly*
