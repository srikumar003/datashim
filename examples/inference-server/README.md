# Serving Models stored in S3 buckets using Datashim Datasets

(_Note:_ Credit to [YAML file from @zioproto](https://github.com/zioproto/kube-cheshire-cat/blob/1ae8be76e333482a2656431c9e6de59f2132c79c/kubernetes/tgi.yaml) for TGI deployment in Kubernetes which provided the basis for the TGI deployment shown in this example)

Large language models are the most interesting cloud workloads of the day. This example demonstrates reducing the friction of loading models from an S3 bucket using Datashim. For this example, we use the open-source [Text Generation Inference (TGI)](https://github.com/huggingface/text-generation-inference) from [HuggingFace](https://huggingface.co/) as the inference service that loads the models and makes it available for prompt inputs. 

But first, let's define a Dataset that points to an S3 bucket. We will use the [FLAN-T5-Base model](https://huggingface.co/google/flan-t5-base) in this example. We will assume that the model has been downloaded to the bucket prior to this exercise. 

## Populating the bucket

In this example, we use a bucket called `model-weights` to store the weights of the model. Now, we define a `Dataset` object for this bucket as:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: model-weights-secret
stringData:
  accessKeyID: "{ACCESS_KEY_ID}"
  secretAccessKey: "{SECRET_ACCESS_KEY}"
---
apiVersion: datashim.io/v1alpha1
kind: Dataset
metadata:
  name: model-weights
spec:
  local:
    bucket: {BUCKET_NAME}
    endpoint: {S3_PROVIDER_ENDPOINT}
    secret-name: model-weights-secret
    type: COS
```

where `S3_PROVIDER_ENDPOINT` is the URL for the provider, `BUCKET_NAME` is the bucket in which the weights are stored, and the secret is populated with our access credentials for our provider. Store the above to a file `model-weights.yaml`. Then, we create the dataset:

```bash
kubectl create -f model-weights.yaml -n <target_namespace>
```

where `target_namespace` is the namespace where we are creating our deployment. You can verify that the Dataset is created and working, and the corresponding PVC named `model_weights` has been created.

> [!IMPORTANT]
> Remember to label `target_namespace` with `monitor-pods-datasets=enabled` so that the pods obtain volumes automatically!

The weights (e.g. `model.safetensors` which is the default format used by HuggingFace) should be downloaded into this bucket. Following is an example of Kubernetes Job that uses the dataset to download the weights for `google/flan-t5-base`: 

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: download
spec:
  template:
    metadata:
      labels:
        dataset.0.id: "model-weights"
        dataset.0.useas: "mount"
    spec:
      containers:
      - image: nicolaka/netshoot
        command: ["/usr/bin/wget"]
        args:
          - "https://huggingface.co/google/flan-t5-base/resolve/main/model.safetensors?download=true"
          - "-O"
          - "/mnt/datasets/model-weights/model.safetensors"
        imagePullPolicy: IfNotPresent
        name: netshoot
      restartPolicy: Never
```

## Creating the deployment

Next, we create the TGI deployment
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: text-generation-inference
  labels:
    run: text-generation-inference
    dataset.0.id: "model-weights"
    dataset.0.useas: "mount"
spec:
  containers:
    - name: text-generation-inference
      image: ghcr.io/huggingface/text-generation-inference:1.3.4
      env:
        - name: RUST_BACKTRACE
          value: "1"
      command:
        - "text-generation-launcher"
        - "--model-id"
        - "google/flan-t5-base"
        - "--sharded"
        - "false"
        - "--port"
        - "8080"
        - "--huggingface-hub-cache"
        - "/tmp"
        - "--weights-cache-override"
        - "/mnt/datasets/model-weights"
      ports:
        - containerPort: 8080
          name: http
      readinessProbe:
        tcpSocket:
            port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
  nodeSelector:
    nvidia.com/gpu.product: ${GPU_TYPE}
  restartPolicy: Never
---
apiVersion: v1
kind: Service
metadata:
  name: text-generation-inference
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: text-generation-inference
  type: ClusterIP
```

The key lines are the labels starting with `dataset.0.` which define the `model_weights` dataset as an input to the TGI pod and the command arguments `"--weights-cache-override"` which indicates to TGI to load the model weights from a specific directory. In this example, the directory location points to the volume where the bucket will eventually be mounted (`/mnt/datasets/model_weights`) and where the model weights will be found. 

Another parameter to pay attention to is the `nodeSelector` that determines where the pod will be scheduled. As LLM inference requires GPUs, you will need a node with atleast one GPU available to run this example. Please consult the documentation for your cloud provider to understand how to specify the `nodeSelector`.

Populate and store the above file in `inference_service.yaml`. Then, create the pod by:
```bash
kubectl create -f inference_service.yaml -n <target_namespace>
```

We can wait for the service to come up using the command:
```bash
kubectl -n <target_namespace> wait pod --for=condition=Ready text-generation-inference --timeout=-1s
```

We can also monitor the pods by looking at the logs:
```bash
kubectl -n <target_namespace> logs -f text-generation-inference 
```

If all goes well, you will see the following:
```
...
2024-02-02T10:11:42.898835Z  INFO download: text_generation_launcher: Starting download process.
2024-02-02T10:11:47.594562Z  INFO text_generation_launcher: Files are already present on the host. Skipping download.

2024-02-02T10:11:48.104228Z  INFO download: text_generation_launcher: Successfully downloaded weights.
2024-02-02T10:11:48.104564Z  INFO shard-manager: text_generation_launcher: Starting shard rank=0
2024-02-02T10:11:58.119392Z  INFO shard-manager: text_generation_launcher: Waiting for shard to be ready... rank=0
...
2024-02-02T10:17:47.316383Z  INFO shard-manager: text_generation_launcher: Shard ready in 628.50105993s rank=0
2024-02-02T10:17:47.404081Z  INFO text_generation_launcher: Starting Webserver
```
which indicates that the service has been set up successfully and is ready to reply to prompts.

## Validating the deployment

To test this, lets do the following:
```bash
kubectl -n <target_namespace> port-forward  --address localhost pod/text-generation-inference 8888:8080
```

```bash
 curl -s http://localhost:8888/generate -X POST -d '{"inputs":"The square root of x is the cube root of y. What is y to the power of 2, if x = 4?", "parameters":{"max_new_tokens":1000}}'  -H 'Content-Type: application/json' | jq -r .generated_text
```
which should return the answer
```
x = 4 * 2 = 8 x = 16 y = 16 to the power of 2
```
