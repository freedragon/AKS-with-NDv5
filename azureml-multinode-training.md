## Multi-Node Training with Hugging Face accelerate and AzureML with attached AKS cluster

원본 포스트에서는 Azure ML Workspace에 배포된 Compute Instance 또는 Compute Cluster를 중심으로 다수의 노드로 분산 학습 하는 방법에 대해 설명 하고 있습니다.

본 문서에서는 위의 포스트에 붙여서 ND96isr_H100_v5 sku의 GPU Nodepool 로 학습하는 방법에 대한 내용을 설명하고 있습니다.

AKS를 기반으로 학습을 실행하기 때문에, AKS 클러스터와 Nvidia GPU 및 Infiniband의 구성과 성능 측정이 된 이후 Azure ML의 Training Job을 실행 해야 합니다.

이미지 데이터가 크지 다소 작아서 (약 755MB) 분산 학습의 성능 향상을 충분히 확인 하기에는 부족한 감이 있습니다.

Azure Machine Learning 에 접근할 수 있는 구독과 Workspace가 있어야 테스트가 가능 합니다.

원본 포스트의 실행 환경은 Azure ML Curated Environment 중 ```openmpi3.1.2-ubuntu18.04``` 를 기반으로 실행 합니다.


### AKS를 기반으로 분산 학습 시 달라지는 점.

기본적으로 분산 학습을 위해서 training.py 를 변경해야 하는 부분과 학습을 진행하는 파이프라인의 실행 코드 (Command)의 파라미터가 변경 되어야 합니다.

[Distributed GPU training guide (SDK v2)](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-train-distributed-gpu?view=azureml-api-2)  

AKS용 AzureML Extension을 설치하면 학습 Pod들의 Orchestration을 위해서 **Volcano** 가 같이 설치 됩니다. 물론, Azure ML Training Job을 API 호출을 통해서 구성해 주시면 Volcano를 이용해서 Job을 배포 하고 실행 및 모니터링 하는 등의 작업은 Extension이 제공해 줍니다.

Job이 실행 되면 학습 진행 중에 정확도(Accuracy), LOSS 및 노드의 CPU, Memory, Disk, Network 리소스 사용량을 Azure ML Studio에서 모니터링 할 수 있습니다. 

#### 학습 코드 변경

```
import torch.distributed as dist
```

학습용 스크립트에서 실제 학습 코드를 실행하기 전에 아래의 코드를 추가 해서 분산 학습을 위한 준비를 합니다.
```
dist.init_process_group(backend='nccl', init_method='env://')
```

**Azure ML Studio Workspace**에서 **PyTorch** 분산 학습을 위해서 기본 제공는 환경 변수들은 다음과 같습니다:

- MASTER_ADDR: IP address of the machine that hosts the process with rank 0
- MASTER_PORT: A free port on the machine that hosts the process with rank 0
- WORLD_SIZE: The total number of processes. Should be equal to the total number of devices (GPU) used for distributed training
- RANK: The (global) rank of the current process. The possible values are 0 to (world size - 1)
- LOCAL_RANK: The local (relative) rank of the process within the node. The possible values are 0 to (# of processes on the node - 1). This information is useful because many operations such as data preparation only should be performed once per node, usually on local_rank = 0.
- NODE_RANK: The rank of the node for multi-node training. The possible values are 0 to (total # of nodes - 1).

#### 학습 파이프라인 코드 변경

```python
num_training_nodes = 2 # Number of nodes available for distributed training
num_gpus_per_node = 8 # Number of GPUs per node (8 for ND96isr_H100_v5)
# Unlike document, command line should be change to run distributed learning.
str_command = "OMP_NUM_THREADS=12 torchrun --rdzv_id 202503 train.py --data_dir ${{inputs.pets}} --with_tracking --checkpointing_steps epoch --output_dir ./outputs"

# Define the job!
job = command(
    code=src_dir,
    inputs=inputs,
    command=str_command,
    environment=train_environment,
    compute=gpu_compute_target,
    instance_count=num_training_nodes * num_gpus_per_node,  # For AKS, we need to set the number of instances to WORLD_SIZE instead of # of nodes.
    distribution={
        "type": "PyTorch",
        # set process count to the number of gpus per node
        "process_count_per_instance": 1, # 1 GPU per POD
    },
    experiment_name=experiment_name,
    display_name='train-step',
    environment_variables={
        'NCCL_DEBUG': 'INFO',
        "NCCL_TOPO_FILE": "ndv5-topo.xml",
    },
    shm_size="4g",
)
```

> [!NOTE] 
> ND96isr_H100_v5 Topology 파일 링크
> https://github.com/Azure/azhpc-images/blob/master/topology/ndv5-topo.xml
>


NCCL이 실행시 GPU Topology를 탐색을 통해 구성 하지만 미리 구성된 내용이 있다면 제공해 주는 것이 좋겠습니다. 이때 **NCCL_TOPO_FILE** 환경 변수를 통해서 설정해 줄 수 있습니다.

### 아직 테스트 하지 되지 않은부분:

Nvidia의 Network/GPU operator가 설치되어서 Cuda/PyTorch 및 Infiniband를 사용해서 향상된 성능의 분산 학습을 테스트 하는 중에
**```ACPT-PyTorch-2.2-cuda12.1:30```** ([Azure Container for PyTroch](https://learn.microsoft.com/en-us/azure/machine-learning/resource-azure-container-for-pytorch?view=azureml-api-2) 이미지)를 기반으로 하는 경우 NCCL이 Infiniband 를 인식하지 못하고 TCP 로 통신하는 현상을 확인 하였습니다.

다시 GPU 노드가 확보 되면 IB 기반 NCCL통신을 확인할 예정 입니다.

### References:

[Distributed GPU training guide (SDK v2)](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-train-distributed-gpu?view=azureml-api-2)  

[Optimized Environment for large scale distributed training](https://github.com/Azure/azureml-examples/blob/main/best-practices/largescale-deep-learning/Environment/ACPT.md)  

[NCCL 2.26: Environment Variables](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)  

[Performance considerations for large scale deep learning training on Azure NDv4 (A100) series](https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/performance-considerations-for-large-scale-deep-learning-training-on-azure-ndv4-/2693834)  

[acpt-pytorch-2.2-cuda12.1 SPEC](https://github.com/Azure/azureml-assets/blob/main/assets/training/general/environments/acpt-pytorch-2.2-cuda12.1/spec.yaml)  

[DEPRECATED: Containerized Nvidia Mellanox drivers](https://github.com/Mellanox/ofed-docker)  

[Multi-Node Training with Hugging Face accelerate and AzureML](https://nateraw.com/posts/multinode_training_accelerate_azureml.html)  

[Run NCCL tests on GPU to check performance and configuration - Code Samples | Microsoft Learn](https://learn.microsoft.com/en-us/samples/azure/azureml-examples/run-nccl-tests-on-gpu-to-check-performance-and-configuration/)  

[JingchaoZhang/Running-GPU-Enabled-HPC-Applications-on-Azure-Machine-Learning](https://github.com/JingchaoZhang/Running-GPU-Enabled-HPC-Applications-on-Azure-Machine-Learning)  

[Distribute Training with Pytorch Lightning on Azure ML](https://medium.com/@felipe.villa.gen/distribute-traning-with-pytorch-lightning-on-azure-ml-512e0cb1728f)  

[Azure Machine Learning - Environment](https://azure.github.io/azureml-cheatsheets/docs/cheatsheets/python/v1/environment/#:~:text=To%20set%20environment%20variables%20use%20the%20environment_variables%3A%20Dict%5Bstr%2C,the%20process%20where%20the%20user%20script%20is%20executed.)  
