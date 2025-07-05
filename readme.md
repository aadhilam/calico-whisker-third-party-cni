# Calico Whisker on Third-party CNI

This guide demonstrates how to deploy Calico for Policy on an AKS cluster with Azure CNI. 

## Prerequisites

- Azure CLI installed and configured
- kubectl configured to access your cluster
- Helm 3.x installed

## Step 1: Deploy AKS Cluster

### 1.1 Create a resource group
```bash
az group create --name calicooss --location eastus2
```

### 1.2 Create the AKS cluster
```bash
az aks create \
  --resource-group calicooss \
  --name calico-whisker \
  --node-count 3 \
  --network-plugin azure \
  --kubernetes-version 1.31.8
```

### 1.3 Get cluster credentials
```bash
az aks get-credentials --resource-group calicooss --name calico-whisker
```

### 1.4 Query the network profile of the cluster
```bash
az aks show --resource-group calicooss --name calico-whisker --query "networkProfile" -o json
```

## Step 2: Install Calico on AKS

### 2.1 Add the Project Calico Helm repository
```bash
helm repo add projectcalico https://docs.projectcalico.org/charts
```

### 2.2 Update Helm repositories
```bash
helm repo update
```

### 2.3 Install Calico using the tigera-operator chart
```bash
helm install calico projectcalico/tigera-operator \
  --version v3.30.0 \
  --namespace tigera-operator \
  --create-namespace \
  --values calico-helm-values/calico-values.yaml
```

### 2.4 Verify Calico installation
```bash
# Check if tigera-operator is running
kubectl get pods -n tigera-operator

# Check Calico installation status
watch kubectl get tigerastatus
```

## Step 3: Deploy the YaoBank Application

### 3.1 Deploy the application
```bash
kubectl apply -f app/yaobank.yaml
```

### 3.2 Verify application deployment
```bash
# Check all pods are running
kubectl get pods -n yaobank

# Wait for all deployments to be ready
kubectl wait --for=condition=Available deployment --all -n yaobank --timeout=300s
```

## Step 4: Set up port-forward to Calico Whisker

```bash
# Port-forward to access Calico Whisker backend
kubectl port-forward service/whisker 8081 -n calico-system
```

### 4.1 Access Calico Whisker UI
Open your browser and navigate to:
```
http://localhost:8081
```

### 4.2 Query flow logs via API
```bash
# Query recent flow logs (last 60 seconds)
curl -G 'http://127.0.0.1:8081/whisker-backend/flows' \
  --data-urlencode startTimeGte=-60 \
  --data-urlencode startTimeLt=0 | jq
```

## Cleanup

### Uninstall everything
```bash
# Remove the application
kubectl delete -f app/yaobank.yaml

# Uninstall Calico
helm uninstall calico -n tigera-operator
kubectl delete namespace tigera-operator

# Delete the AKS cluster
az aks delete --resource-group calicooss --name calico-whisker --yes --no-wait

# Optional: Delete the resource group (this will delete all resources in the group)
az group delete --name calicooss --yes --no-wait
```
## Additional Resources

- [Calico Documentation](https://docs.projectcalico.org/)