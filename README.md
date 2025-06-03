# Parseable Helm Chart

[Parseable](https://parseable.io) is a fast observability platform on S3 with built-in log search and analytics capabilities. This Helm chart deploys Parseable with optional Fluent Bit and Vector integrations for log collection.

## Installation

### Prerequisites

- Kubernetes cluster (e.g., Minikube, EKS, GKE, AKS)
- Helm 3.x
- kubectl configured to communicate with your cluster

### Create Configuration Secret

**Important:** This step must be completed before installing Parseable with Helm.

Create a secret file with the configuration for Parseable:

```sh
cat << EOF > parseable-env-secret
addr=0.0.0.0:8000
staging.dir=./staging
fs.dir=./data
username=admin
password=admin
EOF
```

Then create the namespace and secret in Kubernetes:

```sh
kubectl create ns parseable
kubectl create secret generic parseable-env-secret --from-env-file=parseable-env-secret -n parseable
```

**Note:** 
- The secret **must** be created before running the Helm install command.
- The namespace must also be created before running the Helm install command.

### Add Required Helm Repositories

```sh
# Add Parseable Helm repository
helm repo add parseable https://charts.parseable.com

# Add Fluent Bit Helm repository (if using Fluent Bit)
helm repo add fluent https://fluent.github.io/helm-charts

# Add Vector Helm repository (if using Vector)
helm repo add vector https://helm.vector.dev

# Update repositories
helm repo update
```

### Install Parseable with Local Storage and Fluent Bit Enabled

**After** creating the secret in the previous step, install Parseable with local storage and Fluent Bit enabled:

```sh
helm install parseable parseable/parseable -n parseable \
  --set parseable.store=local-store \
  --set parseable.persistence.staging.enabled=true \
  --set parseable.persistence.staging.storageClass=standard \
  --set parseable.persistence.data.enabled=true \
  --set parseable.persistence.data.storageClass=standard \
  --set parseable.localModeSecret.enabled=true \
  --set parseable.localModeSecret.name=parseable-env-secret \
  --set fluent-bit.enabled=true \
  --set fluent-bit.serverHost=parseable.parseable.svc.cluster.local
```

**Important Notes:**
- The installation references the `parseable-env-secret` we created in the previous step.
- Make sure the namespace has been created before running this command.
- If you're using a different storage class, replace `standard` with your cluster's storage class name.

### Access the Parseable Dashboard

Once deployed, you can access the Parseable dashboard by port-forwarding the Parseable service:

```sh
kubectl port-forward svc/parseable 8080:80 -n parseable
```

Then open your browser and navigate to http://localhost:8080

Login with the default credentials:
- Username: admin
- Password: admin

## Chart Values

To see all available configuration options for the Parseable Helm chart, run:

```sh
helm show values parseable/parseable
```

## Customization Options

### Storage Options

Parseable supports multiple storage backends:

```sh
# Local storage (default, good for testing)
--set parseable.store=local-store

# S3-compatible storage
--set parseable.store=s3-store
--set parseable.s3ModeSecret.enabled=true

# Azure Blob Storage
--set parseable.store=blob-store
--set parseable.blobModeSecret.enabled=true

# Google Cloud Storage
--set parseable.store=gcs-store
--set parseable.gcsModeSecret.enabled=true
```

### High Availability Mode

For production deployments, you can enable high availability mode (not supported with local storage):

```sh
--set parseable.highAvailability.enabled=true
--set parseable.highAvailability.ingestor.count=3
```

### Log Collection Options

#### Fluent Bit Configuration

Customize Fluent Bit settings:

```sh
# Enable Fluent Bit
--set fluent-bit.enabled=true

# Configure server connection
--set fluent-bit.serverHost=parseable.parseable.svc.cluster.local
--set fluent-bit.serverUsername=admin
--set fluent-bit.serverPassword=admin

# Set log stream name
--set fluent-bit.serverStream=my-logs

# Exclude specific namespaces
--set fluent-bit.excludeNamespaces="kube-system, default"
```

#### Vector Integration

Enable Vector for additional log collection capabilities:

```sh
--set vector.enabled=true
```

## Troubleshooting

### Common Issues

1. **Persistent Volume Claims in Pending State**
   
   If PVCs remain in Pending state, ensure your cluster has a default StorageClass or specify a valid StorageClass:
   ```sh
   kubectl get storageclass
   ```

2. **Fluent Bit Connection Issues**
   
   If Fluent Bit cannot connect to Parseable, check the logs:
   ```sh
   kubectl logs -f daemonset/parseable-fluent-bit -n parseable
   ```
   
   Common issues include incorrect serverHost value. Make sure to use the full service name including namespace:
   ```sh
   --set fluent-bit.serverHost=parseable.parseable.svc.cluster.local
   ```

3. **Parseable UI Not Accessible**
   
   If you cannot access the Parseable UI, check the pod status and logs:
   ```sh
   kubectl get pods -n parseable
   kubectl logs deployment/parseable -n parseable
   ```

4. **Configuration Secret Issues**
   
   If Parseable pods are failing to start and logs show configuration-related errors, verify your secret:
   ```sh
   # Check if the secret exists
   kubectl get secret parseable-env-secret -n parseable
   
   # Recreate the secret if needed
   kubectl create secret generic parseable-env-secret --from-env-file=parseable-env-secret -n parseable --dry-run=client -o yaml | kubectl apply -f -
   ```

## Using Lua scripts with Fluent Bit
Fluent Bit allows us to build filter to modify the incoming records using custom [Lua scripts.](https://docs.fluentbit.io/manual/pipeline/filters/lua)

### How to use Lua scripts with this Chart

First, you should add your Lua scripts to `luaScripts` in values.yaml, for example:

```yaml
luaScripts:
  filter_example.lua: |
    function filter_name(tag, timestamp, record)
        -- put your lua code here.
    end
```

After that, the Lua scripts will be ready to be used as filters. So next step is to add your Fluent bit [filter](https://docs.fluentbit.io/manual/concepts/data-pipeline/filter) to `config.filters` in values.yaml, for example:

```yaml
config:
  filters: |
    [FILTER]
        Name    lua
        Match   <your-tag>
        script  /fluent-bit/scripts/filter_example.lua
        call    filter_name
```
Under the hood, the chart will:
- Create a configmap using `luaScripts`.
- Add a volumeMounts for each Lua scripts using the path `/fluent-bit/scripts/<script>`.
- Add the Lua script's configmap as volume to the pod.

### Note
Remember to set the `script` attribute in the filter using `/fluent-bit/scripts/`, otherwise the file will not be found by fluent bit.
