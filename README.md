# AKS-with-NDv5
<!-- How to deploy AKS node pool (not NAP) with ND96isr H100 v5 nodes as worker and run nccl-test. -->

Azure AKS에서 ND96isr H100 v5 sku의 노드 풀을 배포하고, Nvidia Node Feature Discovery, Network Operator 및 GPU Operator를 Helm Chart로 배포 하여 Infiniband 구동을 NCCL-Test를 실행해서 확인하는 방법에 대한 가이드 입니다. (2025년 2월 마지막 주 기준)

아래 참조 사이트의 [Microsoft Community Hub](https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/deploy-ndm-v4-a100-kubernetes-cluster/3838871)와 [Github 리포](https://github.com/edwardsp/ai-on-aks)에서 배포 방법 등에 대해 자세히 가이드 하고 있습니다만, Infiniband용 Network Driver와 Nvidia GPU Driver의 버젼이 업그레이드 되면서 가이드에서 최신 Helm Chart를 설치 하면 정상적으로 설정이 되지 않는 이슈 있습니다. (추가 설정을 하지 않으먄 gpu-operator가 설치 되지 않고 Pod가 Init 상태로 계속 유지 되었습니다)

본 가이드는 2025년 2월 마지막 주 기준으로 아래 버젼의 오퍼레이터들을 설치하고 Infiniband가 정상적으로 작동하는지를 테스트 하는 방법에 대해 설명하고 있습니다.

> [!NOTE]
> AKS의 배포 방법은 Azure CLI를 기준으로 설명하고 있으며, operator들의 설치는 Helm Chart를 이용 합니다.

> [!WARNING]
> [이전 가이드](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster?tabs=add-ubuntu-gpu-node-pool) 에 기술 되어 있는 GPU Device Plugin 설치를 하시게 되면 GPU Operator와 충돌이 발생 할 수 있습니다. Azure Reosurce Manager 또느 Terraform등으로 IaC 방식 배포를 하시는 경우 반드시 드라이버가 설치 되지 않도록 주의 해 주세요.
> 

#### \[준비 작업\] 배포용 환경 변수 정의

전체적인 배포 과정에서 필요한 리소스 그룹이름, 데이터 센터 (LOCATION), AKS 클러스터 이름 및 ACR 의 이름등을 정의 합니다.
<!-- The following are variables will be used in the deployment steps: -->

```bash
export RESOURCE_GROUP=<리소스 그룹이름>
export LOCATION=<Azure 리젼. 예를 들어 KoreaCentral>
export CLUSTER_NAME=<AKS 클러스터 이름>
export ACR_NAME=<ACR 이름>
```

#### \[준비 작업\] aks-preview 확장을 Azure CLI로 설치 

AKS 배포 중에 skip-gpu-driver-install 옵션을 사용하기 위해 필요한 작업 입니다.
<!-- The aks-preview extension is required to deploy the AKS cluster with GPU nodes by enabling the option. -->
```bash
az extension add --name aks-preview
```

#### AKS Infiniband 지원 

AKS 배포 전에 Infiniband 를 지원 하도록 Feature를 Azure 구독에 Azure CLI로 등록 해 주셔야 합니다. 
<!-- The feature need to be registered to ensure the AKS cluster is deployed with Infiniband support.  The following command will register the feature: -->
```bash
az feature register --name AKSInfinibandSupport --namespace Microsoft.ContainerService
```

<!-- Note: check the feature status with the following command to ensure it is reporting `Registered`: -->
> [!NOTE]  
> Feature들의 등록 상태 확인을 위해서 다음의 명령을 수행 하세요. 등록이 완료되면 "Registered" 가 출력 됩니다.
> 
> ```bash
> az feature show --name AKSInfinibandSupport --namespace Microsoft.ContainerService --query properties.state --output tsv
> ```

#### Azure 리소스 그룹 생성

```
az group create --name $RESOURCE_GROUP --location $LOCATION
```

#### AKS 에서 사용할 Container Registry 생성 (이미 생성된 ACR 이 있는 경우 클러스터 생성으로 진행 하시면 됩니다)

```bash
az acr create \
    --resource-group $RESOURCE_GROUP \
    --name $ACR_NAME \
    --sku Basic \
    --admin-enabled
```

이미 ACR 이 AKS가 배포 될 동일 구독에 배포된 경우, 다음의 가이드를 참고하셔서 AKS에 Attach 하실 수 있습니다.

[Attach an ACR to an existing AKS cluster](https://learn.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli#attach-an-acr-to-an-existing-aks-cluster) 

```bash
# Attach using acr-name
az aks update --name myAKSCluster --resource-group myResourceGroup --attach-acr <acr-name>
```

#### AKS 클러스터 배포

미리 정의된 환경 변수들의 값을 이용해서 AKS Cluster를 배포 합니다. 배포시 Standard_DS2_v3 sku로 2개의 노드를 생성 합니다.
```bash
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

AKS 클러스터의 배포가 끝나면 이어서 GPU가 장착된 sku (Standard_ND96isr_H100_v5)의 노드들을 추가로 배포 합니다. 배포 시 노드들은 별도의 노드풀에 배포 됩니다. 아래 명령줄을 보시면 노드풀의 이름음 `ndv5` 입니다.  `SkipGPUDriverInstall=true`  태그는 기본적으로 드라이버가 포함된 이미지 대신 기본 OS 이미지의 노드를 배포 합니다. 서비스 구동에 필요한 드라이버들은 이후 과정에서 직접 설치 하게 됩니다.

````bash
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

###Nvidia 드라이버 설정들들 (Helm chart version들 기준) :
- Node Feature Discovery:
    - Helmchart 이름: **node-feature-discovery**
    - 버젼: v0.17.2
- Network Operator: 
    - Helmchart 이름: **network-operator**
    - 버젼: v24.07.0
- GUP Operator (gpu-operator): 
    - Helmchart 이름: **gpu-operator**
    - 버젼: v24.9.2

#### Helm 설치

Heml을 처음 사용 하시는 거라면 다음의 명령줄들을 실행 하셔서 Helm CLI를 다운로드 하실 수 있습니다. 이미 사용 중이라면 무시 하시면 됩니다.
<!--Kubernets의 패키지 매니져인 Helm is a package manager for Kubernetes that allows you to easily deploy and manage applications on your AKS cluster.  The following commands will get the latest version of Helm and install it locally: -->

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### NVIDIA Drivers

NC/ND sku의 GPU 노드들을 배포 한 후, Infiniband와 GPU 드라이버를 설치 해야 Tensorflow나 Pythoch등을 이용해서 멀티노드 학습이나 서비스가 가능 합니다. 설정을 위해서는 노드에 장치된 물리 장치를 식별하는 Node Feature Discovery 가 우선 설치 되어야 합니다. 이 패키지들은 별도의 네임스페이스로 분리 해서 진행 합니다.
<!-- The NVIDIA CPU and Network operators are used to manage the GPU drivers and Infiniband drivers on the NDv5 nodes.  In this configuration we will install the Node Feature Discovery separately as it is used by both operators.  The installations will all use Helm. -->

#### NVIDIA operator들을 위한 네임스페이스 생성

NDF와 Operator들이 설치/운영 될 네임스페이스를 생성 합니다.
```bash
kubectl create namespace nvidia-operator
```

> [!NOTE]
> 네임스페이스를 다른 이름으로 변경 하시는 경우, 아래 Helm Chart 설치 명령줄의 -n 옵션 들의 nvidia-operator역시 동일하게 변경해 주셔야 합니다.

#### Node Feature Discovery 설치

[Node Feature Discovery](https://github.com/kubernetes-sigs/node-feature-discovery) 는 실행시 노드의 하드웨어를 검색해서 노드의 레이블에 표시해 주는 역활을 수행 합니다.

```bash
helm upgrade -i --wait \
  -n nvidia-operator node-feature-discovery node-feature-discovery \
  --repo https://kubernetes-sigs.github.io/node-feature-discovery/charts \
  --set-json master.nodeSelector='{"kubernetes.azure.com/mode": "system"}' \
  --set-json worker.nodeSelector='{"kubernetes.azure.com/accelerator": "nvidia"}' \
  --set-json worker.config.sources.pci.deviceClassWhitelist='["02","03","0200","0207"]' \
  --set-json worker.config.sources.pci.deviceLabelFields='["vendor"]'
```

#### the Network Operator 설치

고속, 고성능 네트워크 디바이스가 장착된 노드에 드라이버들을 설치하고 Node에 Label(`nvidia.com/mlnxnics` )과 디바이스 숫자 (ND96isr_H100_v5의 경우 8) 를 업데이트 합니다.

참조 문서들과 달리 테스트 결과 Network operator의 24.07.0 버젼을 설치 해야 Network Operator가 정상적으로 설치 되는 것을 확인 하였습니다.

```bash
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
> network-operator가 정상적으로 설치 된 경우 다수의 Pod들이 오퍼레이터용 네임스페이스에서 실행 됩니다. 다음의 명령을 수행 하면 노드의 장치 들 중에 GPU와 Mellanox (Infiniband) 장치의 숫자를 표시 해 줍니다. (최대값은 ND H100 v5의 경우 8 입니다)
>
> ```bash
> kubectl describe node <NDv5_AKS_node> | grep -e "nvidia.com/mlnxnics" -e "nvidia.com/gpu"
> ```

#### GPU Operator 설치

Helm 으로 Nvidia GPU 드라이버를 설치 합니다.

```bash
helm upgrade -i --wait \
  -n nvidia-operator gpu-operator gpu-operator \
  --repo https://helm.ngc.nvidia.com/nvidia \
  --set nfd.enabled=false \
  --set driver.enabled=true \
  --set driver.rdma.enabled=true \
  --set toolkit.enabled=true
```

###Volcano 설치

Kubernest로 HPC나 AI 학습등의 고성능 분산 작업을 쉽게 수행 할 수 있게하는 Orchestrator 중 하나가 Volcano 입니다.

참조 사이트들에서 모두 Volcano의 1.7.0 버젼을 기준으로 설명을 하고 있습니다만, 2025년 2월 현재 AKS의 기본 설치 버젼은 **1.30.5** 입니다. 

[Volcano-sh]: https://volcano.sh/en/ (Job Orchestrator on Kubernetes): v1.11.0
	• Volcano do nothing when there are not enough resources available (from node description)

```bash
kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/release-1.11/installer/volcano-development.yaml
```

> [!NOTE]
> 참조 설치 가이드에서는 Volcano 설치시 Service Account, RoleBinding등을 추가로 생성하는 명령들의 샐행을 요구 합니다만, 최신 버젼의 경우 설치 YAML 파일 실행 시 같이 생성이 되니 무시 하셔도 됩니다.
> 

### NCCL Allreduce 실행

AKS 클러스터에 배포된 GPU노드들 사이에서 Infiniband로 고속 데이터 전송 속도 측정을 위한 샘플 작업 중 대표적인 것이 AllReduce 샘플 입니다.
참조 문서에서 설명하고 있는 방식으로 Dockerfile, nccl-test폴더 및 nccl-test.sh 파일을 이용해서 AKS 노드들에 배포되어 실행될 컨테이너 이미지를 우선 제작 해야 합니다. 만들어진 이미지를 AKS노드가 접근 하는 (또는 Attach된) ACR에 push 해 주고, Volcano를 통해서 Test Job을 생성 해 주면 (Kubectl로 합니다) Pod 배포, 이미지 다운로드 등의 작업을 거쳐서 최종적으로 성능 데이터를 테이블 형태로 출력해 줍니다.

#### NCCL Allreduce 컨데이너 이미지 빌드

```bash
cd nccl-test
az acr login -n $ACR_NAME
docker build -t $ACR_NAME.azurecr.io/nccltest .
docker push $ACR_NAME.azurecr.io/nccltest
```

#### NCCL Allreduce Job 실행

```bash
helm install nccl-allreduce-2n ./examples/nccl-allreduce --set image=$ACR_NAME.azurecr.io/nccltest,numNodes=2
```

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: nccl-allreduce-job1
spec:
  minAvailable: 3
  schedulerName: volcano
  plugins:
    ssh: []
    svc: []
  tasks:
    - replicas: 1
      name: mpimaster
      policies:
        - event: TaskCompleted
          action: CompleteJob
      template:
        spec:
          initContainers:
            - command:
                - /bin/bash
                - -c
                - |
                  until [[ "$(kubectl get pod -l volcano.sh/job-name=nccl-allreduce-job1,volcano.sh/task-spec=mpiworker -o json | jq '.items | length')" != 0 ]]; do
                    echo "Waiting for MPI worker pods..."
                    sleep 3
                  done
                  echo "Waiting for MPI worker pods to be ready..."
                  kubectl wait pod -l volcano.sh/job-name=nccl-allreduce-job1,volcano.sh/task-spec=mpiworker --for=condition=Ready --timeout=600s
              image: mcr.microsoft.com/oss/kubernetes/kubectl:v1.26.3
              name: wait-for-workers
          serviceAccount: mpi-worker-view
          containers:
            - command:
                - /bin/bash
                - -c
                - |
                  MPI_HOST=$(cat /etc/volcano/mpiworker.host | tr "\n" ",")
                  mkdir -p /var/run/sshd; /usr/sbin/sshd
                  echo "HOSTS: $MPI_HOST"
                  mpirun --allow-run-as-root -np 16 -npernode 8 --bind-to numa --map-by ppr:8:node -hostfile /etc/volcano/mpiworker.host -x NCCL_DEBUG=info -x UCX_TLS=tcp -x NCCL_TOPO_FILE=/workspace/ndv4-topo.xml -x UCX_NET_DEVICES=eth0 -x CUDA_DEVICE_ORDER=PCI_BUS_ID -x NCCL_SOCKET_IFNAME=eth0 -mca coll_hcoll_enable 0 /workspace/nccl-tests/build/all_reduce_perf -b 8 -f 2 -g 1 -e 8G -c 1 | tee /home/re
              image: cgacr2.azurecr.io/pytorch_nccl_tests_2303:latest
              securityContext:
                capabilities:
                  add: ["IPC_LOCK"]
              name: mpimaster
              ports:
                - containerPort: 22
                  name: mpijob-port
              workingDir: /workspace
              resources:
                requests:
                  cpu: 1
          restartPolicy: OnFailure
    - replicas: 2
      name: mpiworker
      template:
        metadata:
        annotations:
          k8s.v1.cni.cncf.io/networks: hostdev-net,hostdev-net,hostdev-net,hostdev-net,hostdev-net,hostdev-net,hostdev-net,hostdev-net
        spec:
          containers:
            - command:
                - /bin/bash
                - -c
                - |
                  mkdir -p /var/run/sshd; /usr/sbin/sshd -D;
              image: cgacr2.azurecr.io/pytorch_nccl_tests_2303:latest
              securityContext:
                capabilities:
                  add: ["IPC_LOCK"]
              name: mpiworker
              ports:
                - containerPort: 22
                  name: mpijob-port
              workingDir: /workspace
              resources:
                requests:
                  nvidia.com/gpu: 8
                  nvidia.com/mlnxnics: 8
                limits:
                  nvidia.com/gpu: 8
                  nvidia.com/mlnxnics: 8
              volumeMounts:
              - mountPath: /dev/shm
                name: shm
          restartPolicy: OnFailure
          terminationGracePeriodSeconds: 0
          volumes:
          - name: shm
            emptyDir:
              medium: Memory
              sizeLimit: 8Gi
---
```

참조 사이트:
	- [GitHub - Deployment scripts for AKS with AI examples]: https://github.com/edwardsp/ai-on-aks  
	- [Deploy NDm_v4 (A100) Kubernetes Cluster | Microsoft Community Hub]: https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/deploy-ndm-v4-a100-kubernetes-cluster/3838871  
	- [Nccl test on aks ndmv4 vm - Jingchao’s Website]: https://jingchaozhang.github.io/NCCL-test-on-AKS-NDmV4-VM/  
	- [Releases · Azure/azhpc-images]: https://github.com/Azure/azhpc-images/releases  

