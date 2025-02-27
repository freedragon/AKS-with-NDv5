# AKS-with-NDv5
How to deploy AKS node pool (not NAP) with ND96isr H100 v5 nodes as worker and run nccl-test.

Azure AKS에서 ND96isr H100 v5 sku의 노드 풀을 배포하고, Nvidia Node Feature Discovery, Network Operator 및 GPU Operator를 Helm Chart로 배포 하여 Infiniband 구동을 NCCL-Test를 실행해서 확인하는 방법에 대한 가이드 입니다. (2025년 2워 마지막 주 기준)

아래 참조 사이트의 [Microsoft Community Hub](https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/deploy-ndm-v4-a100-kubernetes-cluster/3838871)와 [Github 리포](https://github.com/edwardsp/ai-on-aks)에서 배포 방법 등에 대해 자세히 가이드 하고 있습니다만, Infiniband용 Network Driver와 Nvidia GPU Driver의 버젼이 업그레이드 되면서 가이드에서 최신 Helm Chart를 설치 하면 정상적으로 설정이 되지 않는 이슈 있습니다. (추가 설정을 하지 않으먄 gpu-operator가 설치 되지 않고 Pod가 Init 상태로 계속 유지 되었습니다)

본 가이드는 2025년 2월 마지막 주 기준으로 아래 버젼의 오퍼레이터들을 설치하고 Infiniband가 정상적으로 작동하는지를 테스트 하는 방법에 대해 설명하고 있습니다.

> [!NOTE]
> AKS의 배포 방법은 Azure CLI를 기준으로 설명하고 있으며, operator들의 설치는 Helm Chart를 이용 합니다.

> [!WARNING]
> [이전 가이드](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster?tabs=add-ubuntu-gpu-node-pool) 가 포함된 AKS + GPU 사용 가이드의 GPU Device Plugin 설치를 하시게 되면 GPU Operator와 충돌이 발생 할 수 있습니다. Azure Reosurce Manager 또느 Terraform등으로 IaC 방식 배포를 하시는 경우 반드시 드라이버가 설치 되지 않도록 주의 해 주세요.
> 

#### \[준비 작업\] aks-preview 확장을 Azure CLI로 설치 

AKS 배포 중에 skip-gpu-driver-install 옵션을 사용하기 위해 필요한 작업 입니다.
<!-- The aks-preview extension is required to deploy the AKS cluster with GPU nodes by enabling the option. -->
```
az extension add --name aks-preview
```

#### AKS Infiniband 지원 

AKS 배포 전에 Infiniband 를 지원 하도록 Feature를 Azure 구독에 Azure CLI로 등록 해 주셔야 합니다. 
<!-- The feature need to be registered to ensure the AKS cluster is deployed with Infiniband support.  The following command will register the feature: -->
```
az feature register --name AKSInfinibandSupport --namespace Microsoft.ContainerService
```

<!-- Note: check the feature status with the following command to ensure it is reporting `Registered`: -->
> [!NOTE]  
> Feature들의 등록 상태 확인을 위해서 다음의 명령을 수행 하세요. 등록이 완료되면 "Registered" 가 출력 됩니다.
> 
> ```
> az feature show --name AKSInfinibandSupport --namespace Microsoft.ContainerService --query properties.state --output tsv
> ```

#### Azure 리소스 그룹 생성

```
az group create --name $RESOURCE_GROUP --location $LOCATION
```

#### AKS 에서 사용할 Container Registry 생성 (이미 생성된 ACR 이 있는 경우 클러스터 생성으로 진행 하시면 됩니다)

```
az acr create \
    --resource-group $RESOURCE_GROUP \
    --name $ACR_NAME \
    --sku Basic \
    --admin-enabled
```

#### AKS 클러스터 배포

```
az aks create \
  --resource-group $RESOURCE_GROUP \
  --node-resource-group ${RESOURCE_GROUP}-nrg \
  --name $CLUSTER_NAME \
  --enable-managed-identity \
  --node-count 2 \
  --generate-ssh-keys \
  --location $LOCATION \
  --node-vm-size Standard_D2s_v3 \
  --nodepool-name system \
  --os-sku Ubuntu \
  --attach-acr $ACR_NAME
```

#### GPU (ND96isr H100 v5) 노드풀 배포

This will create a node pool using for NDv5 VMs.  The `SkipGPUDriverInstall=true` tag is used to ensure AKS is not managing the GPU drivers.  Instead we will manage this with the NVIDIA GPU operator.

````
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name ndv5 \
  --node-count 1 \
  --node-vm-size Standard_ND96isr_H100_v5 \
  --node-osdisk-size 128 \
  --os-sku Ubuntu \
  --tags SkipGPUDriverInstall=true
````

###Nvidia 드라이버 설정 (Helm chart version들 기준) :
	- Node Feature Discovery:
    - Helmchart 이름: **node-feature-discovery**
    - 버젼: v0.17.2
	- Network Operator: 
    - Helmchart 이름: **network-operator**
    - 버젼: v24.07.0
	- GUP Operator (gpu-operator): 
    - Helmchart 이름: **gpu-operator**
    - 버젼: v24.9.2

#### Helm

Helm is a package manager for Kubernetes that allows you to easily deploy and manage applications on your AKS cluster.  The following commands will get the latest version of Helm and install it locally:

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### NVIDIA Drivers

The NVIDIA CPU and Network operators are used to manage the GPU drivers and Infiniband drivers on the NDv5 nodes.  In this configuration we will install the Node Feature Discovery separately as it is used by both operators.  The installations will all use Helm.

#### NVIDIA operator들을 위한 네임스페이스 생성

```
kubectl create namespace nvidia-operator
```

> [!NOTE]
> 네임스페이스를 다른 이름으로 변경 하시는 경우, 아래 Helm Chart 설치 명령줄의 -n 옵션 들의 nvidia-operator역시 동일하게 변경해 주셔야 합니다.

#### Node Feature Discovery 설치

```
helm upgrade -i --wait \
  -n nvidia-operator node-feature-discovery node-feature-discovery \
  --repo https://kubernetes-sigs.github.io/node-feature-discovery/charts \
  --set-json master.nodeSelector='{"kubernetes.azure.com/mode": "system"}' \
  --set-json worker.nodeSelector='{"kubernetes.azure.com/accelerator": "nvidia"}' \
  --set-json worker.config.sources.pci.deviceClassWhitelist='["02","03","0200","0207"]' \
  --set-json worker.config.sources.pci.deviceLabelFields='["vendor"]'
```

#### the Network Operator 설치

참조 문서들과 달리 테스트 결과 Network operator의 24.07.0 버젼을 설치 해야 Network Operator가 정상적으로 설치 되는 것을 확인 하였습니다.

```
helm upgrade -i --wait \
  -n nvidia-operator network-operator network-operator \
  --repo https://helm.ngc.nvidia.com/nvidia \
  --version v24.7.0 \
  --set deployCR=true \
  --set nfd.enabled=false \
  --set ofedDriver.deploy=true \
  --set secondaryNetwork.deploy=false \
  --set rdmaSharedDevicePlugin.deploy=true \
  --set sriovDevicePlugin.deploy=true \
  --set-json sriovDevicePlugin.resources='[{"name":"mlnxnics","linkTypes": ["infiniband"], "vendors":["15b3"]}]'
```

> [!Note]
> MOFED 드라이버를 특정 버젼으로 설정을 원하시면  `--set ofedDriver.version="<MOFED-VERSION>"` 옵션을 추가해 주세요.


> [!Note]
> network-operator가 정상적으로 설치 된 경우 다수의 Pod들이 오퍼레이터용 네임스페이스에서 실행 됩니다. GPU 노드의 

#### GPU Operator 설치

```
helm upgrade -i --wait \
  -n nvidia-operator gpu-operator gpu-operator \
  --repo https://helm.ngc.nvidia.com/nvidia \
  --set nfd.enabled=false \
  --set driver.enabled=true \
  --set driver.rdma.enabled=true \
  --set toolkit.enabled=true
```


###Volcano 설치

참조 사이트들에서 모두 Volcano의 1.7.0 버젼을 기준으로 설명을 하고 있습니다만, 2025년 2월 현재 AKS의 기본 설치 버젼은 **1.30.5** 입니다. 
[Volcano-sh]: https://volcano.sh/en/ (Job Orchestrator on Kubernetes): v1.11.0
	• Volcano do nothing when there are not enough resources available (from node description)

```
kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/release-1.11/installer/volcano-development.yaml
```

> [!NOTE]
> 참조 설치 가이드에서는 Volcano 설치시 Service Account, RoleBinding등을 추가로 생성하는 명령들의 샐행을 요구 합니다만, 최신 버젼의 경우 설치 YAML 파일 실행 시 같이 생성이 되니 무시 하셔도 됩니다.
> 

### NCCL Allreduce 실행

#### Build the NCCL Allreduce Container Image

```
cd nccl-test
az acr login -n $ACR_NAME
docker build -t $ACR_NAME.azurecr.io/nccltest .
docker push $ACR_NAME.azurecr.io/nccltest
```

#### Launching the NCCL Allreduce Job

```
helm install nccl-allreduce-2n ./examples/nccl-allreduce --set image=$ACR_NAME.azurecr.io/nccltest,numNodes=2
```


참조 사이트:
	- [GitHub - Deployment scripts for AKS with AI examples]: https://github.com/edwardsp/ai-on-aks
	- [Deploy NDm_v4 (A100) Kubernetes Cluster | Microsoft Community Hub]: https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/deploy-ndm-v4-a100-kubernetes-cluster/3838871
	- [Nccl test on aks ndmv4 vm - Jingchao’s Website]: https://jingchaozhang.github.io/NCCL-test-on-AKS-NDmV4-VM/
	- [Releases · Azure/azhpc-images]: https://github.com/Azure/azhpc-images/releases

