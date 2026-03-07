# Trail of Bits AIxCC Finals CRS

## Repository use case

This repository contains Trail of Bits' AIxCC Finals Cyber Reasoning System (CRS): a multi-service platform that accepts challenge tasks, runs fuzzing and analysis workflows, and submits proof-of-vulnerability (PoV), patch, and SARIF-style findings back to the competition API.

In practice, you can use this repo as a reference implementation for running an AI-assisted vulnerability research pipeline end-to-end:

* Ingest challenge metadata and source bundles.
* Build and fuzz targets continuously.
* Triage crashes and produce reproducers.
* Generate and validate candidate patches.
* Submit artifacts and track round status.

## Using this repository for your own code

If you want to apply this system to a different codebase, follow this simple workflow:

1. **Deploy the stack locally** using the configuration and deployment instructions below.
2. **Prepare your target project** so it can be built and fuzzed in a containerized environment (build scripts, dependencies, and harnesses).
3. **Replace sample task inputs** (for example, `example-libpng`) with task data that points to your project bundle and fuzz targets.
4. **Run the orchestrator flow** to send tasks and SARIF updates, then monitor scheduler logs for PoV/patch submission behavior.
5. **Iterate on configuration** (model provider keys, runtime limits, profiles) to match your project's language, build system, and scale.

For component-level details while adapting the pipeline, see:

* `common/README.md` for shared task/data utilities.
* `fuzzer/README.md` for fuzzing infrastructure behavior.
* `program-model/README.md` for program analysis/modeling support.
* `deployment/README.md` for deployment-specific operations.

## Dependencies

Follow the install instructions for the required dependencies:

* [Docker install guide](https://docs.docker.com/engine/install/ubuntu/)
* [kubectl install guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
* [helm install guide](https://helm.sh/docs/intro/install/):
* [minikube install guide](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fdebian+package)
* Git LFS for some tests

## Configuration

Create a new configuration file, starting from the default template:

```shell
cp \
  deployment/env.template \
  deployment/env
```

Next, configure the following options. Follow the instructions in the comments when setting the `GHCR_AUTH` value.

```shell
SCANTRON_GITHUB_PAT
GHCR_AUTH
OPENAI_API_KEY
ANTHROPIC_API_KEY
DOCKER_USERNAME
DOCKER_PAT
```

### Settings specific to local development and testing

Use the hardcoded test credentials found in the comments:

```shell
AZURE_ENABLED=false
TAILSCALE_ENABLED=false
COMPETITION_API_KEY_ID: `11111111-1111-1111-1111-111111111111`
COMPETITION_API_KEY_TOKEN: `secret`
CRS_KEY_ID="515cc8a0-3019-4c9f-8c1c-72d0b54ae561"
CRS_KEY_TOKEN="VGuAC8axfOnFXKBB7irpNDOKcDjOlnyB"
CRS_API_HOSTNAME="<generated with: openssl rand -hex 16>"
BUTTERCUP_K8S_VALUES_TEMPLATE="k8s/values-minikube.template"
OTEL_ENDPOINT="<insert endpoint url from aixcc vault, is pseudo secret>"
OTEL_PROTOCOL="http"
```

Keep empty:

```shell
AZURE_API_BASE=""
AZURE_API_KEY=""
```

Commented out:

```shell
CRS_URL
CRS_API_HOSTNAME
LANGFUSE_HOST
LANGFUSE_PUBLIC_KEY
LANGFUSE_SECRET_KEY
OTEL_TOKEN
```

When [re-running unscored rounds](orchestrator/src/buttercup/orchestrator/mock_competition_api/README.md), set this to `true`:

```shell
MOCK_COMPETITION_API_ENABLED
```

## Authentication

### Docker

Log into ghcr.io:

```shell
docker login ghcr.io -u <username>
```

## Docker Compose

A Compose setup is provided at the repository root.

* `compose.yaml` contains the full stack definition.
* `docker-compose.yml` is a compatibility wrapper so tools that expect the legacy filename still work.

Start the default stack:

```shell
docker compose up -d
```

Start optional services (example: graph database profile):

```shell
docker compose --profile graphdb up -d
```

Stop and remove containers:

```shell
docker compose down --remove-orphans
```

## Running the CRS

### Starting the services

```shell
cd deployment && make up
```

### Stopping the services

```shell
cd deployment && make down
```

### Sending the example-libpng task to the system

```shell
kubectl port-forward -n crs service/buttercup-competition-api 31323:1323
```

```shell
./orchestrator/scripts/task_crs.sh
```

Send a SARIF message

```shell
./orchestrator/scripts/send_sarif.sh <TASK-ID-FROM-TASK-CRS>
```

### Simulating Unscored Round 2

```shell
kubectl port-forward -n crs service/buttercup-competition-api 31323:1323
```

```shell
./orchestrator/scripts/challenge.sh
```

Check that patches get submitted to the bundler.

```shell
kubectl logs -n crs -l app=scheduler --tail=-1 --prefix | grep "WAIT_PATCH_PASS -> SUBMIT_BUNDLE"
```

If needing to debug, run the following to log into the pod.

```shell
kubectl get pods -n crs

kubectl exec -it -n crs <pod-name> -- /bin/bash
```

## Run Unscored Challenges

See [UNSCORED.md](UNSCORED.md)
