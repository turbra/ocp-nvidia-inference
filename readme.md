# OpenShift Node Feature Discovery and NVIDIA GPU Operator Deployment
### [Deploying Red Hat AI Inference Server](https://docs.redhat.com/en/documentation/red_hat_ai_inference_server/3.2/html-single/deploying_red_hat_ai_inference_server_in_openshift_container_platform/index#idm140531502886512)

---

This guide demonstrates how to enable and utilize an NVIDIA GPU on an OpenShift cluster.
The environment used for this example is **OpenShift SNO 4.19.16 (bare metal)** with an **NVIDIA GeForce RTX 3090**.
Although the 3090 is a consumer GPU, it provides sufficient capability for validation and testing GPU functionality in OpenShift.

As part of this walkthrough, we will deploy and run the **ibm-granite/granite-3.1-8b-instruct** model from Hugging Face.
**granite-3.1-8b-instruct** is a state-of-the-art open LLM designed by Google for instruction-tuned tasks such as text generation and conversational applications.
This example demonstrates how to pull, configure, and serve a Hugging Face model on OpenShift using GPU acceleration.

---

# Quick Steps

- Installing Node Feature Discovery Operator using the Web Console
  - Accept defaults
  - Create NodeFeatureDiscovery using Form View
    - Accept defaults
- Install Nvidia GPU Operator
  - Accept defaults
  - Create ClusterPolicy 
    - Accept defaults

This process can take a little while. While you wait, you can keep an eye on the progress by checking for NVIDIA pods in the cluster.
```bash
oc get pods -n nvidia-gpu-operator
NAME                                           READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-8nbls                    2/2     Running     0          26h
gpu-operator-6794d46bc7-w254h                  1/1     Running     0          5d
nvidia-container-toolkit-daemonset-68qwg       1/1     Running     0          26h
nvidia-cuda-validator-m9c6n                    0/1     Completed   0          26h
nvidia-dcgm-exporter-2bpmz                     1/1     Running     0          26h
nvidia-dcgm-f92hv                              1/1     Running     0          26h
nvidia-device-plugin-daemonset-zx8zq           2/2     Running     0          26h
nvidia-driver-daemonset-9.6.20251008-0-zc8mx   2/2     Running     0         4d23h
nvidia-node-status-exporter-gxkvl              1/1     Running     0          4d23h
nvidia-operator-validator-v2cqh                1/1     Running     0          26h
```
After all pods have started successfully, you’ll notice that the gpu-cluster-policy changes to Ready.
```bash
oc get clusterpolicy -n nvidia-gpu-operator
NAME                 STATUS   AGE
gpu-cluster-policy   ready    2025-10-24T14:25:09Z
```

---
# NVIDIA GPU Operator commands

To see some more information about the GPU, run the following commands:

```bash
oc project nvidia-gpu-operator
oc get pod -owide -lopenshift.driver-toolkit=true
# Take note of the name of the nvidia-driver-daemonset and replace it in the following command
oc exec -it nvidia-driver-daemonset-<fromyouroutput> -- nvidia-smi
```

```bash
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.82.07              Driver Version: 580.82.07      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3090        On  |   00000000:01:00.0 Off |                  N/A |
| 76%   68C    P2            337W /  370W |   20070MiB /  24576MiB |    100%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A           56285      C   /usr/bin/python3                      20060MiB |
+-----------------------------------------------------------------------------------------+
```

### Adjust power limits
```bash
# Lower 3090 power limit to 250W
oc exec -it -n nvidia-gpu-operator nvidia-driver-daemonset-<replace> -- nvidia-smi -pl 250

# Reset 3090 power limit to 370W
oc exec -it -n nvidia-gpu-operator nvidia-driver-daemonset-<replace> -- nvidia-smi -pl 370
```

# RH-AI-Inference

# Procedure

1. **Create the `Secret` custom resource (CR) for the Hugging Face token**
   The cluster uses the `Secret` CR to pull models from Hugging Face.

   1. Set the `HF_TOKEN` variable using the token you set in [Hugging Face](https://huggingface.co/settings/tokens):

      ```bash
      $ HF_TOKEN=<your_huggingface_token>
      ```

   2. Set the cluster namespace to match where you deployed the Red Hat AI Inference Server image, for example:

      ```bash
      $ NAMESPACE=rhaiis
      ```

   3. Create the `Secret` CR in the cluster:

      ```bash
      $ oc create secret generic hf-secret --from-literal=HF_TOKEN=$HF_TOKEN -n $NAMESPACE
      ```

2. **Create the Docker secret**
   The cluster uses this secret to download the Red Hat AI Inference Server image from the container registry.
   For example, to create a `Secret` CR that contains the contents of your local `~/.docker/config.json` file, run:

   ```bash
   oc create secret generic docker-secret --from-file=.dockercfg=$HOME/.docker/config.json --type=kubernetes.io/dockercfg -n rhaiis
   ```

---



## Testing Examples

```bash
curl -X POST http://rhaiis.apps.ocp4.example.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "granite-3-1-8b-instruct-quantized-w8a8",
    "messages": [{"role": "user", "content": "What is AI?"}],
    "temperature": 0.1
  }'
```

```bash
VLLM="http://rhaiis.apps.ocp4.example.com"
PROMPT='Explain SIMD in one concise paragraph.'

curl -s -w '\nTOTAL_TIME=%{time_total}\n' \
  -H 'Content-Type: application/json' \
  -d "{
    \"model\":\"granite-3-1-8b-instruct-quantized-w8a8\",
    \"messages\":[{\"role\":\"user\",\"content\":\"$PROMPT\"}],
    \"temperature\":0.0,
    \"top_p\":1.0,
    \"max_tokens\":128,
    \"stream\":false
  }" \
  $VLLM/v1/chat/completions \
| tee /dev/tty \
| awk '
  /^TOTAL_TIME=/ { t=$0; sub("TOTAL_TIME=","",t); total=t }
  /^[{[]/ { json = json $0 }
  END {
    # pull completion_tokens from the JSON with a tiny jq if available
    # fallback prints just TOTAL_TIME if jq missing
    cmd = "jq -r \".usage.completion_tokens\" <<<\"" json "\" 2>/dev/null"
    cmd | getline tokens
    close(cmd)
    if (tokens !~ /^[0-9]+$/) { tokens=""; }
    if (tokens!="") {
      printf("{\"tokens_out\":%d,\"total_s\":%f,\"tok_per_s\":%f}\n", tokens, total, (tokens/total))
    } else {
      printf("{\"total_s\":%f}\n", total)
    }
  }'
```




