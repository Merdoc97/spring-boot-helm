# Spring Boot Helm Chart

A Helm chart for deploying Spring Boot applications to Kubernetes with common configurations and best practices.

## Overview

This chart provides a standardized way to deploy Spring Boot applications to Kubernetes with the following features:

- Deployment with configurable resources and health probes
- Service configuration
- Ingress with retry capabilities
- Horizontal Pod Autoscaler (HPA)
- Database migration hooks
- Prometheus metrics integration
- Volume management

## Installation

### Add the Helm Repository

```shell
helm repo add spring-boot-helm https://raw.githubusercontent.com/Merdoc97/spring-boot-helm/master
helm repo update
```

### Install the Chart

```shell
helm install my-app spring-boot-helm/base-chart \
  --set global.application.name=my-app \
  --set global.namespace=my-namespace \
  --set global.image=my-image \
  --set global.imageVersion=1.0.0
```

Or using a values file:

```shell
helm install my-app spring-boot-helm/base-chart -f my-values.yaml
```

## Configuration

### Basic Configuration

```yaml
global:
  # Docker registry URL
  dockerRegistryUrl: "registry.example.com"
  # Image name
  image: "my-app"
  # Image version/tag
  imageVersion: "1.0.0"
  # Kubernetes namespace
  namespace: "my-namespace"

  application:
    # Application name
    name: "my-app"
    # Application port
    port: 8080
    # Number of replicas
    replicaCount: 2
```

### Ingress Configuration

The chart includes a pre-configured ingress with retry capabilities:

```yaml
global:
  application:
    ingress:
      # Path rewrite configuration
      rewriteTarget: /$2
      targetPath: /api(/|$)(.*)
      annotations:
        # Default retry configuration for 502, 503, 504 status codes
        nginx.ingress.kubernetes.io/proxy-next-upstream: "error timeout http_502 http_503 http_504"
        nginx.ingress.kubernetes.io/proxy-next-upstream-tries: "3"
      host:
        enabled: "true"
        value: "myapp.example.com"
```

### Resource Configuration

```yaml
global:
  application:
    resources:
      java:
        opts: "-XX:InitialRAMPercentage=50.0 -XX:MaxRAMPercentage=70.0 -XX:+ExitOnOutOfMemoryError"
      requests:
        memory: 512Mi
        cpu: 500m
      limits:
        memory: 1Gi
        cpu: 1000m
```

### Health Probes

```yaml
global:
  application:
    deployment:
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: ahttp
          scheme: HTTP
        initialDelaySeconds: 30
        timeoutSeconds: 10
        periodSeconds: 20
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: ahttp
          scheme: HTTP
        initialDelaySeconds: 30
        timeoutSeconds: 10
        periodSeconds: 30
```

### Horizontal Pod Autoscaler (HPA)

Enable and configure the Horizontal Pod Autoscaler:

```yaml
global:
  hpa:
    enabled: "true"
    minReplicas: 2
    maxReplicas: 5
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 80
```

### Database Migration Hook

The chart includes a pre-install and pre-upgrade hook for database migrations:

```yaml
global:
  datasource:
    migration:
      enabled: "true"
      image: "my-migration-image:latest"
      secretRefs:
        - db-credentials
      configMapRefs:
        - db-config
```

### Prometheus Integration

```yaml
global:
  application:
    prometheus:
      actuatorPath: /actuator/prometheus
      scrape: true
      actuatorPort: 8081
```

### Volume Configuration

```yaml
global:
  application:
    volumes:
      - name: config-volume
        configMap:
          name: app-config
    volumeMounts:
      - name: config-volume
        mountPath: /app/config
```

### External Secrets and ConfigMaps

```yaml
global:
  application:
    deployment:
      configMapRefs:
        - app-config
        - common-config
      secretRefs:
        - app-secrets
        - db-credentials
```

### External Secrets from Vault

The chart supports integration with HashiCorp Vault using the External Secrets Operator to fetch secrets before application deployment:

```yaml
global:
  externalSecrets:
    enabled: "true"
    secretStore: "vault-backend"  # Reference to your SecretStore resource
    hooks:
      - name: "app-secrets"  # Name of the Kubernetes Secret to be created
        annotations:
          "helm.sh/hook": "pre-install,pre-upgrade"
          "helm.sh/hook-weight": "-10"
          "helm.sh/hook-delete-policy": "hook-succeeded"
        data:
          - secretKey: "database-password"  # Key in the Kubernetes Secret
            remoteRef:
              key: "myapp/database"  # Path in Vault
              property: "password"   # Key in Vault
          - secretKey: "api-key"
            remoteRef:
              key: "myapp/api"
              property: "key"
```

This configuration creates an ExternalSecret resource as a Helm prehook that fetches secrets from Vault and creates a Kubernetes Secret before your application is deployed.

## Examples

### Basic Spring Boot Application

```yaml
global:
  dockerRegistryUrl: "docker.io"
  image: "myorg/myapp"
  imageVersion: "1.2.3"
  namespace: "production"

  application:
    name: "myapp"
    port: 8080
    replicaCount: 3

    ingress:
      targetPath: /api(/|$)(.*)
      host:
        enabled: "true"
        value: "api.example.com"

    resources:
      requests:
        memory: 512Mi
        cpu: 500m
      limits:
        memory: 1Gi
        cpu: 1000m
```

### Application with Database Migration

```yaml
global:
  dockerRegistryUrl: "docker.io"
  image: "myorg/myapp"
  imageVersion: "1.0.0"
  namespace: "staging"

  application:
    name: "myapp-with-db"

  datasource:
    migration:
      enabled: "true"
      image: "myorg/myapp-migration:1.0.0"
      secretRefs:
        - db-credentials
```

### Application with HPA

```yaml
global:
  dockerRegistryUrl: "docker.io"
  image: "myorg/myapp"
  imageVersion: "1.0.0"
  namespace: "production"

  application:
    name: "scalable-app"

  hpa:
    enabled: "true"
    minReplicas: 3
    maxReplicas: 10
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70
```

### Application with External Secrets from Vault

This example shows how to configure an application to use external secrets from Vault:

```yaml
global:
  dockerRegistryUrl: "docker.io"
  image: "myorg/secure-app"
  imageVersion: "1.0.0"
  namespace: "secure-namespace"

  application:
    name: "secure-app"
    deployment:
      secretRefs:
        - secure-app-secrets  # Reference to the secret created by the external secrets

  # External Secrets configuration
  externalSecrets:
    enabled: "true"
    secretStore: "vault-backend"  # Reference to your SecretStore resource
    hooks:
      - name: "secure-app-secrets"  # Name of the Kubernetes Secret to be created
        annotations:
          "helm.sh/hook": "pre-install,pre-upgrade"
          "helm.sh/hook-weight": "-10"
          "helm.sh/hook-delete-policy": "hook-succeeded"
        data:
          - secretKey: "database-password"  # Key in the Kubernetes Secret
            remoteRef:
              key: "secure-app/database"  # Path in Vault
              property: "password"   # Key in Vault
          - secretKey: "api-key"
            remoteRef:
              key: "secure-app/api"
              property: "key"
          - secretKey: "jwt-secret"
            remoteRef:
              key: "secure-app/jwt"
              property: "secret"
```

In this example:
1. The External Secrets Operator fetches secrets from Vault before the application is deployed
2. The secrets are stored in a Kubernetes Secret named `secure-app-secrets`
3. The application deployment references this secret to access the credentials

## Extending the Chart

### Adding Custom YAML Files

You can extend the chart with your own custom YAML files by adding them to the templates directory when you integrate the chart into your project. This allows you to add additional Kubernetes resources or override existing ones.

To add custom YAML files:

1. Create a directory for your Helm chart in your project
2. Add a `Chart.yaml` file with a dependency on this chart
3. Create a `templates` directory
4. Add your custom YAML files to the templates directory

Example of a custom YAML file (`templates/custom-configmap.yaml`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "{{ .Values.global.application.name }}-custom-config"
  namespace: "{{ .Values.global.namespace }}"
data:
  custom-key: "custom-value"
  app-name: "{{ .Values.global.application.name }}"
```

### Custom External Secrets from Vault

You can create custom YAML files to extend the chart with your own External Secrets configuration. This is particularly useful when you need more complex secret management or when you want to fetch secrets from multiple Vault paths.

To add a custom External Secrets file:

1. Create a file named `external-secrets.yaml` in your templates directory
2. Define an ExternalSecret resource that references your SecretStore and specifies the secrets to fetch
3. Configure it as a Helm hook to ensure it runs before your application is deployed
4. Reference the created secret in your application's deployment

Example structure for `templates/external-secrets.yaml`:

```yaml
# External Secret definition with Helm hook annotations
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-application-vault-secrets
  namespace: my-namespace
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-weight: "-5"
    helm.sh/hook-delete-policy: hook-succeeded
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: my-application-secrets
    creationPolicy: Owner
  data:
    - secretKey: "db-password"
      remoteRef:
        key: "myapp/database"
        property: "password"
    - secretKey: "api-key"
      remoteRef:
        key: "myapp/api"
        property: "key"
```

Then in your `values.yaml` file, you can configure which secrets to fetch:

```yaml
global:
  externalSecrets:
    enabled: "true"
    secretStore: "vault-backend"
    customSecrets:
      - secretKey: "db-password"
        remoteRef:
          key: "myapp/database"
          property: "password"
      - secretKey: "redis-password"
        remoteRef:
          key: "myapp/redis"
          property: "password"
      - secretKey: "api-key"
        remoteRef:
          key: "myapp/api"
          property: "key"
```

This approach gives you more flexibility in how you structure your external secrets configuration while still leveraging the base chart's capabilities.

### Integrating with Your Project

To add this chart to your project, you can use it as a dependency in your own Helm chart:

1. Create a `Chart.yaml` file in your project:

```yaml
apiVersion: v2
name: my-application
description: My Spring Boot Application
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: spring-boot
    version: "0.0.7"  # Use the appropriate version
    repository: "https://raw.githubusercontent.com/Merdoc97/spring-boot-helm/master"
```

2. Create a `values.yaml` file to override default values:

```yaml
global:
  dockerRegistryUrl: "docker.io"
  image: "myorg/my-application"
  imageVersion: "1.0.0"
  namespace: "my-namespace"

  application:
    name: "my-application"
    port: 8080
    # Add other configuration as needed
```

3. Update dependencies and install:

```shell
helm dependency update ./my-application
helm install my-app ./my-application
```

This approach allows you to:
- Maintain your application-specific configuration
- Add custom resources as needed
- Benefit from updates to the base chart
- Keep your deployment configuration DRY (Don't Repeat Yourself)

## Chart Structure

- `deployment.yaml`: Main application deployment
- `service.yml`: Kubernetes service for the application
- `ingress.yaml`: Ingress configuration with retry capabilities
- `hpa.yaml`: Horizontal Pod Autoscaler configuration
- `migration-hook.yaml`: Pre-install/pre-upgrade hook for database migrations
- `service-account.yml`: Service account for the application

## License

This project is licensed under the MIT License.
