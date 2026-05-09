### What you should know before starting with this project?

1. Kubernetes
    * 1.1 Should know k8s/AKS
    * 1.2 How taint and tolerations work
    * 1.3 CRDs
    * 1.4 Deployments, Services
2. GPUs
    * 2.1 How GPUs work
    * 2.2 Internal components of GPU such as SM
    * 2.3 Memory hierarchy of GPU and internal working
    * 2.4 CUDA Cores
    * 2.5 Tensor cores
    * 2.6 GPU time slicing
3. LLM and vLLM
4. Pytorch Basics


### We are deploying a Standard_NC4as_T4_v3 VM having 1x NVIDIA T4 GPU.
#### The GPU will serve the LLM model TinyLlama 1.1B (FP16) which has following specs - 
- Model: TinyLlama 1.1B (FP16)
- VRAM requirement: ~5.5 GB
- Fits in T4 (15GB)?: Fits comfortably
- Tokens/sec on T4: ~20 tok/s

#### About the model TinyLlama:

- TinyLlama is a 1.1 billion parameter language model released by researchers at Singapore University. 
- It was trained on 3 trillion tokens using the same architecture as Meta's Llama 2 — but compressed down to a tiny size.

#### The objective is to deploy the LLM model to GPU over AKS and test the results.

