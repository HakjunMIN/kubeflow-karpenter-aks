# Kubeflow Karpetner AKS

이 Repo는 [Kubeflow](https://www.kubeflow.org/) ML toolkit을 AKS에 구동하고 Karpenter를 통해 GPU가 필요한 경우에만 (예: 파이프라인 가동)사용할 수 있는 시나리오에 활용 가능함.

## AKS 클러스터 설치

* 이 가이드는 기본적으로 az cli bash를 통한 설치이나 Terraform이나 Bicep등 다른 IaC를 활용할 수 있음. (아래 Note참고)

> [!Note]
> 아래 샘플 활용 가능.
> AKS Landing zone accelerator: https://github.com/Azure/AKS-Landing-Zone-Accelerator/blob/main/Scenarios/AKS-Secure-Baseline-PrivateCluster/Terraform/README.md
> AKS Module: https://registry.terraform.io/modules/Azure/aks/azurerm/latest

* 리소스그룹 및 AKS생성

```bash
export CLUSTER_NAME=kubeflowcl
export RG=kubeflow
export LOCATION=japaneast
export KARPENTER_NAMESPACE=kube-system
export AZURE_SUBSCRIPTION_ID=<Subscription-id>

az group create --resource-group ${RG} --location ${LOCATION}

KMSI_JSON=$(az identity create --name karpentermsi --resource-group "${RG}" --location "${LOCATION}")

az aks create \
  --name "${CLUSTER_NAME}" --resource-group "${RG}" \
  --node-count 5 --generate-ssh-keys \
  --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium \
  --enable-managed-identity \
  --enable-oidc-issuer --enable-workload-identity 

```

* Credential생성 및 Karpenter가 사용할 권한 할당

```bash
AKS_JSON=$(az aks show --name "${CLUSTER_NAME}" --resource-group "${RG}")  
az aks get-credentials --name "${CLUSTER_NAME}" --resource-group "${RG}" --overwrite-existing

az identity federated-credential create --name KARPENTER_FID --identity-name karpentermsi --resource-group "${RG}" \
  --issuer "$(jq -r ".oidcIssuerProfile.issuerUrl" <<< "$AKS_JSON")" \
  --subject system:serviceaccount:${KARPENTER_NAMESPACE}:karpenter-sa \
  --audience api://AzureADTokenExchange

KARPENTER_USER_ASSIGNED_CLIENT_ID=$(jq -r '.principalId' <<< "$KMSI_JSON")
RG_MC=$(jq -r ".nodeResourceGroup" <<< "$AKS_JSON")
RG_MC_RES=$(az group show --name "${RG_MC}" --query "id" -otsv)
for role in "Virtual Machine Contributor" "Network Contributor" "Managed Identity Operator"; do
  az role assignment create --assignee "${KARPENTER_USER_ASSIGNED_CLIENT_ID}" --scope "${RG_MC_RES}" --role "$role"
done  

# use configure-values.sh to generate karpenter-values.yaml
# (in repo you can just do ./hack/deploy/configure-values.sh ${CLUSTER_NAME} ${RG})
curl -sO https://raw.githubusercontent.com/Azure/karpenter-provider-azure/main/hack/deploy/configure-values.sh
chmod +x ./configure-values.sh && ./configure-values.sh ${CLUSTER_NAME} ${RG} karpenter-sa karpentermsi

```

## Nvidia GPU Helm Operator설치 

```bash
helm install --wait --generate-name \
     -n nvidia-gpu-operator --create-namespace \
     nvidia/gpu-operator
```
https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html

### (대안) Helm Operator가 아닌 Nvidia Device Pludgin을 통해 GPU Driver를 설치할 수 있음. 

*  Helm Operator로 Driver를 설치한 경우 아래 절차 Skip

```bash
kubectl create ns gpu-resources
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: gpu-resources
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      # Mark this pod as a critical add-on; when enabled, the critical add-on scheduler
      # reserves resources for critical add-on pods so that they can be rescheduled after
      # a failure.  This annotation works in tandem with the toleration below.
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      # Allow this pod to be rescheduled while the node is in "critical add-ons only" mode.
      # This, along with the annotation above marks this pod as a critical add-on.
      - key: CriticalAddonsOnly
        operator: Exists
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      - key: "sku"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
      containers:
      - image: mcr.microsoft.com/oss/nvidia/k8s-device-plugin:v0.14.1
        name: nvidia-device-plugin-ctr
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
          - name: device-plugin
            mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
```

## Kerpenter 설치

> [!Note]
> Karpenter 대신 Cluster Autoscaler를 구성할 경우 VM 종류 별로 Node Pool을 구성하여 Taint/Toleration등을 이용하여 Pod이 스케쥴링 될 수 있도록 설정

* Helm Operator를 이용하여 Karpenter설치

```bash

export KARPENTER_VERSION=0.4.0

helm upgrade --install karpenter oci://mcr.microsoft.com/aks/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --values karpenter-values.yaml \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait

kubectl logs -f -n "${KARPENTER_NAMESPACE}" -l app.kubernetes.io/name=karpenter -c controller
```

## Karpenter의 기본 노드풀과 노드 클래스 생성

```yaml

apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general-purpose
  annotations:
    kubernetes.io/description: "General purpose NodePool for generic workloads"
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.azure.com/sku-family
          operator: In
          values: [D]
      nodeClassRef:
        name: default
  limits:
    cpu: 100
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: Never
---
apiVersion: karpenter.azure.com/v1alpha2
kind: AKSNodeClass
metadata:
  name: default
  annotations:
    kubernetes.io/description: "General purpose AKSNodeClass for running Ubuntu2204 nodes"
spec:
  imageFamily: Ubuntu2204

```

## GPU 노드풀과 노드클래스 생성

```yaml

apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu
  annotations:
    kubernetes.io/description: "General GPU purpose NodePool for ML workloads"
spec:
  disruption:
    expireAfter: 1h
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.azure.com/sku-gpu-manufacturer
          operator: In
          values: ["nvidia"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
      nodeClassRef:
        name: gpu-nc
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule

---
apiVersion: karpenter.azure.com/v1alpha2
kind: AKSNodeClass
metadata:
  name: gpu-nc
  annotations:
    kubernetes.io/description: "General purpose AKSNodeClass for running Ubuntu2204 nodes"
spec:
  imageFamily: Ubuntu2204
```

## GPU 노드가 Karpenter에 의해 프로비저닝 되는지 Job을 통해 테스트

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app: samples-tf-mnist-demo
  name: samples-tf-mnist-demo
spec:
  template:
    metadata:
      labels:
        app: samples-tf-mnist-demo
    spec:
      containers:
      - name: samples-tf-mnist-demo
        image: mcr.microsoft.com/azuredocs/samples-tf-mnist-demo:gpu
        args: ["--max_steps", "500"]
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            nvidia.com/gpu: "1"
      restartPolicy: OnFailure
      tolerations:
      - key: "sku"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"
```

## Kubeflow 설치

### MinIO 설정

#### Storage Account생성

```bash
STORAGE_ACCOUNT_NAME="kubeflowsa"

# Create a resource group
az group create --name $RG --location $LOCATION

# Create a storage account
az storage account create --name $STORAGE_ACCOUNT_NAME --resource-group $RG --location $LOCATION --kind StorageV2 --sku Standard_LRS

# Retrieve the primary key
STORAGE_KEY=$(az storage account keys list --resource-group $RG --account-name $STORAGE_ACCOUNT_NAME --query '[0].value' --output tsv)

# Print the storage key
echo $STORAGE_KEY
```

#### MinIO를 Azure Blob Storage로 사용하도록 수정

* [minio-deployment.yaml](manifests/apps/pipeline/upstream/third-party/minio/base/minio-deployment.yaml)의 container의 args를 gateway와 azure로 아래와 같이 변경

```yaml
    ...
    spec:
      containers:
      - args:
        - gateway
        - azure
    ...    
```

#### MioIO secret변경

* `accesskey`에 Storage account이름, `secretkey`에 Storage key를 [mlpipeline-minio-artifact-secret.yaml](manifests/apps/pipeline/upstream/third-party/minio/base/mlpipeline-minio-artifact-secret.yaml)파일 내 stringSecret부분에 각각 입력.


### Kustomize로 Kubeflow 설치 
```bash
cd manifests
kustomize build tls | kubectl apply -f -
```

### Istio ingressgateway의 external endpoint IP확인
```bash
kubectl get svc istio-ingressgateway -n istio-system 
```

### 위에서 확인한 Istio GW ip확인후 certificate.yaml주소 변경

* [certificate.yaml](manifests/tls/certificate.yaml) 파일내 ipAddresses 정보 수정 

> [!Note]
> Domain이 있다면 FQDN 인증서 방식을 선택

* 이후 TLS인증서 부문 반영.

```bash
kubectl apply -f tls/certificate.yaml
```

* 최종적으로 certifate변경으로 인한 전체 설정 재반영 
```bash
kustomize build tls | kubectl apply -f -
```

## (옵션) Jupyter notebook GPU Toleration설정

* Kubeflow내 Jupyter Notebook의 GPU 활용을 위한 Toleration Preset를 위한 절차.

```yaml

apiVersion: "kubeflow.org/v1alpha1"
kind: PodDefault
metadata:
  name: gpu-toleration
  namespace: kubeflow-user-example-com
spec:
 selector:
  matchLabels:
    gpu-toleration: "true"
 desc: "utilize GPU nodepool"
 tolerations:
 - key: "sku"
   operator: "Equal"
   value: "gpu"
   effect: "NoSchedule"   
```    

### Jupyter Notebook 테스트

Notebook 생성 시 Image는 `jupyter-tensorflow-cuda-full` 선택

```python
!pip install tensorflow

import tensorflow as tf
print("Num GPUs Available: ", len(tf.config.list_physical_devices('GPU')))



tf.debugging.set_log_device_placement(True)

# Create some tensors
a = tf.constant([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]])
b = tf.constant([[1.0, 2.0], [3.0, 4.0], [5.0, 6.0]])
c = tf.matmul(a, b)

print(c)
     
```

## Reference
https://github.com/Azure/karpenter-provider-azure
https://github.com/azure/kubeflow-aks
https://min.io/product/multicloud-azure-kubernetes-service
https://v1-5-branch.kubeflow.org/docs/distributions/azure/authentication-oidc/




