# Llama Stack/LMEval tutorial

Assume a common root for all projects (e.g. `~/lls`)

## LLama Stack setup


### Build Llama Stack

Create a virtualenv in `~/lls`:

```sh
cd ~/lls
python -m venv ~/lls/.venv
source .venv/bin/activate
```

Clone the Llama Stack branch with the LMEval provider

```sh
git clone git@github.com:ruivieira/llama-stack.git
cd llama-stack
git checkout release-0.1.9-lmeval
```

Still from `~/lls/llama-stack`:

```sh
pip install -e . --force
llama stack build --template lmeval --image-type venv
```

You can now build the container image

```sh
export LLS_IMAGE="quay.io/<my-username>/distribution-lmeval:latest"
CONTAINER_BINARY=docker USE_COPY_NOT_MOUNT=true LLAMA_STACK_DIR=. \
llama stack build --template lmeval --image-type container
docker tag docker.io/library/distribution-lmeval:dev ${LLS_IMAGE}
```

Push it to a registry:

```sh
docker push ${LLS_IMAGE}
```

### Deploy Llama Stack (Kubernetes)

> Note
>
> Deployment assumes a model, served by vLLM, is already present in your cluster
> with an exposed service named `vllm-server` (although the name can be changed in `manifests/llama-stack.yaml`)

Let's assume a namespace `test` for the remainder of this document.

```sh
kubectl create namespace test
```

> Note
> Perform any configuration necessary at the `run-config` ConfigMap in `manifests/llama-stack.yaml`.
> As an example, if you're using `tinyllama` as your model, you might want to reduce `max_tokens`.

Deploy Llama Stack:

```sh
kubectl apply -f manifests/llama-stack.yaml -n test
```

Now apply the necessary permission for Llama Stack to deploy LMEval jobs:

```sh
kubectl apply -f manifests/lmeval-rbac.yaml
kubectl apply -f manifests/lmeval-sa.yaml
```

Port-forward your Llama Stack API service:

```sh
kubectl port-forward -n test svc/llamastack-service 8321:8321
```

## Requests

Install the Llama Stack client

```sh
pip install llama-stack-client==0.1.9
```

List the available Llama Stack models

```sh
llama-stack-client models list
```

You should get a result similar to

```text
Available Models

┏━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━┓
┃ model_type ┃ identifier       ┃ provider_resource_id ┃ metadata                 ┃ provider_id          ┃
┡━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━┩
│ llm        │ tinyllama        │ tinyllama            │                          │ vllm-inference       │
├────────────┼──────────────────┼──────────────────────┼──────────────────────────┼──────────────────────┤
│ embedding  │ all-MiniLM-L6-v2 │ all-MiniLM-L6-v2     │ {'embedding_dimension':  │ sentence-transforme… │
│            │                  │                      │ 384.0}                   │                      │
└────────────┴──────────────────┴──────────────────────┴──────────────────────────┴──────────────────────┘

Total models: 2
```

Now list the providers, you should see `lmeval`

```sh
llama-stack-client providers list
```

```text
┏━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ API          ┃ Provider ID            ┃ Provider Type                  ┃
┡━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ inference    │ vllm-inference         │ remote::vllm                   │
│ inference    │ sentence-transformers  │ inline::sentence-transformers  │
│ vector_io    │ faiss                  │ inline::faiss                  │
│ eval         │ lmeval                 │ remote::lmeval                 │
│ telemetry    │ meta-reference         │ inline::meta-reference         │
│ tool_runtime │ tavily-search          │ remote::tavily-search          │
│ tool_runtime │ rag-runtime            │ inline::rag-runtime            │
│ tool_runtime │ model-context-protocol │ remote::model-context-protocol │
└──────────────┴────────────────────────┴────────────────────────────────┘
```

