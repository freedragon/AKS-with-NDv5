ARG FROM_IMAGE_NAME=nvcr.io/nvidia/pytorch:23.05-py3

FROM ${FROM_IMAGE_NAME}

RUN apt-get update
RUN apt-get -y install build-essential infiniband-diags openssh-server kmod

COPY nccl-tests.sh .
COPY ndv5-topo.xml .

RUN ./nccl-tests.sh
