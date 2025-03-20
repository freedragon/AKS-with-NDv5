## Multi-Node Training with Hugging Face accelerate and AzureML with attached AKS cluster

원본 포스트에서는 Azure ML Workspace에 배포된 Compute Instance 또는 Compute Cluster를 중심으로 다수의 노드로 분산 학습 하는 방법에 대해 설명 하고 있습니다.

본 문서에서는 위의 포스트에 붙여서 ND96isr_H100_v5 sku의 GPU Nodepool 로 학습하는 방법에 대한 내용을 설명하고 있습니다.

AKS를 기반으로 학습을 실행하기 때문에, AKS 클러스터와 Nvidia GPU 및 Infiniband의 구성과 성능 측정이 된 이후 Azure ML의 Training Job을 실행 해야 합니다.

이미지 데이터가 크지 다소 작아서 (약 755MB) 분산 학습의 성능 향상을 충분히 확인 하기에는 부족한 감이 있습니다.

Azure Machine Learning 에 접근할 수 있는 구독과 Workspace가 있어야 테스트가 가능 합니다.

원본 포스트의 실행 환경은 Azure ML Curated Environment 중 ```openmpi3.1.2-ubuntu18.04``` 를 기반으로 실행 합니다.


### AKS를 기반으로 분산 학습 시 달라지는 점.

기본적으로 분산 학습을 위해서 training.py 를 변경해야 하는 부분과 

[Distributed GPU training guide (SDK v2)](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-train-distributed-gpu?view=azureml-api-2)  

```
torch.distributed.init_process_group(backend='nccl', init_method='env://')
```

**Azure ML Studio Workspace**에서 **PyTorch** 분산 학습을 위해서 기본 제공는 환경 변수들은 다음과 같습니다:

- MASTER_ADDR: IP address of the machine that hosts the process with rank 0
- MASTER_PORT: A free port on the machine that hosts the process with rank 0
- WORLD_SIZE: The total number of processes. Should be equal to the total number of devices (GPU) used for distributed training
- RANK: The (global) rank of the current process. The possible values are 0 to (world size - 1)
- LOCAL_RANK: The local (relative) rank of the process within the node. The possible values are 0 to (# of processes on the node - 1). This information is useful because many operations such as data preparation only should be performed once per node, usually on local_rank = 0.
- NODE_RANK: The rank of the node for multi-node training. The possible values are 0 to (total # of nodes - 1).

### 아직 테스트 하지 되지 않은부분:

Nvidia의 Network/GPU operator가 설치되어서 Cuda/PyTorch 및 Infiniband를 사용해서 향상된 성능의 분산 학습을 테스트 하는 중에
**```ACPT-PyTorch-2.2-cuda12.1:30```** ([Azure Container for PyTroch](https://learn.microsoft.com/en-us/azure/machine-learning/resource-azure-container-for-pytorch?view=azureml-api-2) 이미지)를 기반으로 하는 경우 NCCL이 Infiniband 를 인식하지 못하고 TCP 로 통신하는 현상을 확인 하였습니다.

다시 GPU 노드가 확보 되면 IB 기반 NCCL통신을 확인할 예정 입니다.

### References:

[Distributed GPU training guide (SDK v2)](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-train-distributed-gpu?view=azureml-api-2)  

[Optimized Environment for large scale distributed training](https://github.com/Azure/azureml-examples/blob/main/best-practices/largescale-deep-learning/Environment/ACPT.md)  

[acpt-pytorch-2.2-cuda12.1 SPEC](https://github.com/Azure/azureml-assets/blob/main/assets/training/general/environments/acpt-pytorch-2.2-cuda12.1/spec.yaml)  

[DEPRECATED: Containerized Nvidia Mellanox drivers](https://github.com/Mellanox/ofed-docker)  

[Multi-Node Training with Hugging Face accelerate and AzureML](https://nateraw.com/posts/multinode_training_accelerate_azureml.html)  

[Run NCCL tests on GPU to check performance and configuration - Code Samples | Microsoft Learn](https://learn.microsoft.com/en-us/samples/azure/azureml-examples/run-nccl-tests-on-gpu-to-check-performance-and-configuration/)  

[JingchaoZhang/Running-GPU-Enabled-HPC-Applications-on-Azure-Machine-Learning](https://github.com/JingchaoZhang/Running-GPU-Enabled-HPC-Applications-on-Azure-Machine-Learning)  

[Distribute Training with Pytorch Lightning on Azure ML](https://medium.com/@felipe.villa.gen/distribute-traning-with-pytorch-lightning-on-azure-ml-512e0cb1728f)  
