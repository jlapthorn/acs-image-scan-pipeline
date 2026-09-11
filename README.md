# ACS Image Scan - Tekton Pipeline

A simple Tekton pipeline for OpenShift that downloads `roxctl` from your ACS Central instance and scans container images for vulnerabilities and policy violations.

## Prerequisites

- OpenShift cluster with the Tekton/OpenShift Pipelines operator installed
- ACS (StackRox) Central accessible from the cluster

## Setup

### 1. Create the namespace

```bash
oc new-project acs-pipeline
```

### 2. Generate an ACS API token

- In the ACS Central UI, go to **Platform Configuration > Integrations > Authentication Tokens**
- Create a token with at least the **Continuous Integration** role
- Copy the token value

### 3. Configure the secret

Edit `01-secret.yaml` and replace `REPLACE_WITH_YOUR_ACS_API_TOKEN` with the token from step 2, then apply it:

```bash
oc apply -f 01-secret.yaml
```

### 4. Get your ACS Central route

```bash
oc get route central -n stackrox -o jsonpath='{.spec.host}'
```

Note this value — you'll need it when running the pipeline.

### 5. Deploy the task and pipeline

```bash
oc apply -f 02-task.yaml
oc apply -f 03-pipeline.yaml
```

## Running a scan

### Option A: From the command line

Edit `04-pipelinerun.yaml` and set:
- `image` — the full image reference to scan (e.g. `registry.redhat.io/ubi9/ubi-minimal:latest`)
- `acs_central_endpoint` — the route from step 4 (host:port, no `https://`)

Then run:

```bash
oc create -f 04-pipelinerun.yaml
```

Watch the logs:

```bash
tkn pipelinerun logs -f -L -n acs-pipeline
```

### Option B: From the OpenShift Console

Go to **Pipelines > acs-image-scan > Start** and fill in the parameters.

### Option C: One-liner with tkn

```bash
tkn pipeline start acs-image-scan \
  -p image=registry.redhat.io/ubi9/ubi-minimal:latest \
  -p acs_central_endpoint=central-stackrox.apps.YOUR_CLUSTER_DOMAIN \
  -p output_format=table \
  --showlog \
  -n acs-pipeline
```

## Parameters

| Parameter              | Description                                           | Default |
|------------------------|-------------------------------------------------------|---------|
| `image`                | Full image reference to scan                          | —       |
| `acs_central_endpoint` | ACS Central endpoint (host:port, no `https://`)       | —       |
| `output_format`        | Output format: `table`, `json`, or `csv`              | `table` |

## What it does

The pipeline runs a single task with two steps:

1. **download-roxctl** — pulls the `roxctl` binary directly from your ACS Central instance, so it always matches your ACS version
2. **scan-image** — runs two commands against the target image:
   - `roxctl image scan` — lists CVEs found in the image
   - `roxctl image check` — evaluates the image against your ACS security policies

## File overview

| File                  | Resource     | Description                        |
|-----------------------|--------------|------------------------------------|
| `01-secret.yaml`      | Secret       | ACS API token                      |
| `02-task.yaml`        | Task         | Downloads roxctl and scans image   |
| `03-pipeline.yaml`    | Pipeline     | Wraps the task with parameters     |
| `04-pipelinerun.yaml` | PipelineRun  | Example run (edit before applying) |
