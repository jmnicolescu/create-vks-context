<!--
author: "Julius M. Nicolescu"
date: "October 2026"
description: "Reference guide to creating, activating, and verifying VCF CLI contexts for a Supervisor and DEV and QA VKS clusters in VMware Cloud Foundation 9.1.1. Explains connection prerequisites, authentication, context creation parameters, workload cluster selection, context switching, and Kubernetes access verification through direct commands, with optional Bash scripts as implementation examples."
keywords: ["VCF", "Supervisor", "VKS", "Kubernetes", "VCF CLI", "Contexts"]
lang: en
-->

# Create Supervisor and VKS cluster contexts with VCF CLI

This reference explains how to create local VCF CLI contexts for an existing Supervisor and its VKS workload clusters in VMware Cloud Foundation 9.1.1. Follow the commands to authenticate through the Supervisor API, create the required context, activate it, and verify Kubernetes access.

The DEV and QA procedures use example cluster names and namespaces. Replace the connection details and target names with values for your environment. The accompanying Bash scripts are optional examples of these commands.

## GitHub repository

**Repository:** [github.com/jmnicolescu/create-vks-context](https://github.com/jmnicolescu/create-vks-context)

Access the example Supervisor, DEV, and QA context scripts and their technical documentation in this repository.

## Contents

- [GitHub repository](#github-repository)
- [Prerequisites](#prerequisites)
- [Context creation parameters](#context-creation-parameters)
- [Create the Supervisor context](#create-the-supervisor-context)
- [Create the DEV VKS cluster context](#create-the-dev-vks-cluster-context)
- [Create the QA VKS cluster context](#create-the-qa-vks-cluster-context)
- [Switch between existing contexts](#switch-between-existing-contexts)
- [Recreate an existing context](#recreate-an-existing-context)
- [Local Kubernetes configuration](#local-kubernetes-configuration)
- [Troubleshooting](#troubleshooting)
- [Example Bash scripts](#example-bash-scripts)
- [Reference](#reference)

## Prerequisites

- Install VCF CLI (`vcf`) and `kubectl`, and make them available in `PATH`.
- Establish network connectivity to the Supervisor API and the target Kubernetes API.
- Have a vSphere SSO account authorized to access the intended Supervisor, namespace, and workload cluster.
- Confirm that the Supervisor and any target VKS clusters already exist. Creating a context configures local access; it does not provision a cluster or namespace.

The commands below use `https://192.168.197.31` as the example Supervisor endpoint and `administrator@vsphere.local` as the example SSO username. Complete CLI authentication when prompted. Do not store real passwords in documentation or committed scripts.

The examples include `--insecure-skip-tls-verify` for this lab. This option disables server certificate verification; review your certificate trust requirements before using it elsewhere.

## Context creation parameters

Use `vcf context create <context-name>` with the connection parameters below. The Supervisor API is the entry point for both Supervisor and workload context creation.

| Parameter | Purpose | Example |
| --- | --- | --- |
| `<context-name>` | Name assigned to the local VCF CLI context | `supervisor` or `dev-cluster-01` |
| `--endpoint` | Supervisor API URL used for context creation | `https://192.168.197.31` |
| `--username` | Authorized vSphere SSO identity | `administrator@vsphere.local` |
| `--type` | Context type | `k8s` |
| `--auth-type` | Authentication method used in these examples | `basic` |
| `--workload-cluster-namespace` | Supervisor namespace containing the VKS cluster; used for workload contexts | `dev-namespace` |
| `--workload-cluster-name` | Existing VKS cluster to access; used for workload contexts | `dev-cluster-01` |
| `--insecure-skip-tls-verify` | Disable server certificate verification for the lab connection | Flag with no value |

The examples use the following names:

| Target | Supervisor namespace | Context created | Context activated |
| --- | --- | --- | --- |
| Supervisor | Not specified | `supervisor` | `supervisor` |
| DEV | `dev-namespace` | `dev-cluster-01` | `dev-cluster-01:dev-cluster-01` |
| QA | `qa-namespace` | `qa-cluster-02` | `qa-cluster-02:qa-cluster-02` |

For the workload examples, the activation name combines the local context name and workload cluster name as `<context-name>:<cluster-name>`. Use `vcf context list --wide` to inspect the available contexts.

## Create the Supervisor context

Create a context for the Supervisor API:

```bash
vcf context create supervisor \
  --endpoint https://192.168.197.31 \
  --username administrator@vsphere.local \
  --insecure-skip-tls-verify \
  --type k8s \
  --auth-type basic
```

After creation succeeds, activate the context and check the selection:

```bash
vcf context use supervisor
vcf context list
kubectl config current-context
```

Confirm that the active Kubernetes context is `supervisor` before continuing. Check the target API server and query its nodes:

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
kubectl get nodes
```

Verify that the API server belongs to the intended target and inspect node readiness. If creation or activation fails, resolve the error before proceeding; an earlier context may still be active.

## Create the DEV VKS cluster context

Create a context for `dev-cluster-01` in the Supervisor namespace `dev-namespace`:

```bash
vcf context create dev-cluster-01 \
  --endpoint https://192.168.197.31 \
  --username administrator@vsphere.local \
  --workload-cluster-namespace dev-namespace \
  --workload-cluster-name dev-cluster-01 \
  --insecure-skip-tls-verify \
  --type k8s \
  --auth-type basic
```

### What each argument means

The argument definitions below follow the [VCF CLI context command reference](<https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-consumption/latest/consumer-interfaces-in-vcf/installing-and-using-vcf-cli-v9/command-reference2/system-commands(1)/context2.html>).

| Argument in this example | Explanation |
| --- | --- |
| `dev-cluster-01` after `vcf context create` | Names the local VCF CLI context. This positional argument labels the connection configuration. |
| `--endpoint https://192.168.197.31` | Specifies the Supervisor API URL used to establish the connection. Supply your Supervisor address here. |
| `--username administrator@vsphere.local` | Supplies the login identity for the Kubernetes context. This example uses a vSphere SSO account. |
| `--workload-cluster-namespace dev-namespace` | Selects the Supervisor namespace that contains the existing workload cluster. |
| `--workload-cluster-name dev-cluster-01` | Selects the existing VKS workload cluster within that namespace. This value identifies the cluster; the positional argument above names the local context. |
| `--insecure-skip-tls-verify` | Bypasses endpoint TLS certificate verification. This Boolean flag takes no value and is included for the lab configuration. |
| `--type k8s` | Explicitly selects a Kubernetes context. The reference lists this flag as optional; `k8s` is the short form of `kubernetes`. |
| `--auth-type basic` | Explicitly selects basic authentication instead of leaving authentication type detection to the CLI. |

For this SSO workflow, `--auth-type basic` directs authentication to vCenter SSO identity sources, bypassing the VCF Automation identity-provider flow described in [Broadcom KB 417617](https://knowledge.broadcom.com/external/article/417617). Complete the credential prompt when requested; `--username` supplies only the account name.

The [VCF CLI Quick Start Guide](https://developer.broadcom.com/xapis/vcf-cli-api/latest/api-docs.html) separates context creation from activation and describes the vCenter SSO credential prompt on the first `vcf context use`.

Each trailing `\` continues the same Bash command onto the next line. Keep it as the last character on the line; comments after it would break command continuation.

After creation succeeds, activate the context and check the selection:

```bash
vcf context use dev-cluster-01:dev-cluster-01
vcf context list
kubectl config current-context
```

Confirm that the active Kubernetes context is `dev-cluster-01:dev-cluster-01` before continuing. Check the target API server and query its nodes:

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
kubectl get nodes
```

Verify that the API server belongs to the intended target and inspect node readiness. If creation or activation fails, resolve the error before proceeding; an earlier context may still be active.

## Create the QA VKS cluster context

Create a context for `qa-cluster-02` in the Supervisor namespace `qa-namespace`:

```bash
vcf context create qa-cluster-02 \
  --endpoint https://192.168.197.31 \
  --username administrator@vsphere.local \
  --workload-cluster-namespace qa-namespace \
  --workload-cluster-name qa-cluster-02 \
  --insecure-skip-tls-verify \
  --type k8s \
  --auth-type basic
```

After creation succeeds, activate the context and check the selection:

```bash
vcf context use qa-cluster-02:qa-cluster-02
vcf context list
kubectl config current-context
```

Confirm that the active Kubernetes context is `qa-cluster-02:qa-cluster-02` before continuing. Check the target API server and query its nodes:

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
kubectl get nodes
```

Verify that the API server belongs to the intended target and inspect node readiness. If creation or activation fails, resolve the error before proceeding; an earlier context may still be active.

## Switch between existing contexts

List available contexts:

```bash
vcf context list --wide
```

Run the activation command for the target you want to use:

| Target | Activation command |
| --- | --- |
| Supervisor | `vcf context use supervisor` |
| DEV | `vcf context use dev-cluster-01:dev-cluster-01` |
| QA | `vcf context use qa-cluster-02:qa-cluster-02` |

Confirm the active Kubernetes context before running commands against the cluster:

```bash
kubectl config current-context
```

Once the selection matches the intended target, verify access:

```bash
kubectl get nodes
```

## Recreate an existing context

If a context already exists and its connection settings are correct, activate it using the commands above. To replace a local context, delete its VCF context name and repeat the corresponding creation and verification procedure.

For example, to replace the DEV context:

```bash
vcf context delete dev-cluster-01 -y
```

Then follow [Create the DEV VKS cluster context](#create-the-dev-vks-cluster-context). The deletion command removes the named local context without confirmation; it does not delete the VKS cluster. Use `supervisor` or `qa-cluster-02` when replacing those example contexts.

## Local Kubernetes configuration

Kubernetes access uses local kubeconfig connection and authentication settings. Inspect the active context with `kubectl config current-context` and its API server with the verification commands above. If you use a custom kubeconfig setup, confirm which configuration is active.

Protect kubeconfig files and authentication material. A successful context creation command establishes connection configuration; verify the active target and node access separately.

## Troubleshooting

| Symptom | Check and next action |
| --- | --- |
| `vcf` or `kubectl` is missing | Install the executable or correct `PATH`. |
| Supervisor endpoint is unreachable | Restore lab/VPN access and verify the `--endpoint` value. |
| Authentication fails | Confirm the SSO username, credentials, and access permissions, then complete CLI authentication. |
| Workload context cannot be created | Verify the workload cluster name, Supervisor namespace, permissions, and cluster availability. |
| Plugin discovery or download fails | Inspect the CLI error and network access. Verify context selection and Kubernetes access separately. |
| Nodes are returned for an unexpected cluster | Inspect `kubectl config current-context` and the selected API server; an earlier context may remain active after a failure. |
| Node query returns `Forbidden` | Check Kubernetes access permissions for the selected identity and target. |
| Context deletion succeeded but recreation failed | Correct the connection or authentication error, then repeat the relevant creation procedure. |

## Example Bash scripts

The [GitHub repository](https://github.com/jmnicolescu/create-vks-context) includes three optional Bash examples that automate context deletion, creation, activation, context listing, and a node query.

| Example script | Target |
| --- | --- |
| [01-create-supervisor-context.sh](https://github.com/jmnicolescu/create-vks-context/blob/main/01-create-supervisor-context.sh) | Supervisor context `supervisor` |
| [02-create-dev-cluster-01-context.sh](https://github.com/jmnicolescu/create-vks-context/blob/main/02-create-dev-cluster-01-context.sh) | `dev-cluster-01` in `dev-namespace` |
| [03-create-qa-cluster-02-context.sh](https://github.com/jmnicolescu/create-vks-context/blob/main/03-create-qa-cluster-02-context.sh) | `qa-cluster-02` in `qa-namespace` |

To use an example, review the direct assignments near the top of that script: Supervisor endpoint, SSO username, context name, and any workload cluster name and namespace. Each script has its own settings. The local password assignment is neither explicitly exported nor passed as a CLI argument; complete authentication when prompted.

From the repository directory, run only the example for your intended target:

```bash
bash ./01-create-supervisor-context.sh
```

For DEV, use `bash ./02-create-dev-cluster-01-context.sh`. For QA, use `bash ./03-create-qa-cluster-02-context.sh`.

Each example deletes and recreates its named local context. Strict error handling is commented out, and the final success message is unconditional. Inspect command errors and verify the active context and node access using the procedures in this reference.

## Reference

- [VCF CLI command reference](<https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-consumption/latest/consumer-interfaces-in-vcf/installing-and-using-vcf-cli-v9/command-reference2.html>) — Command groups and CLI reference documentation.
- [VCF CLI Quick Start Guide](<https://developer.broadcom.com/xapis/vcf-cli-api/latest/api-docs.html>) — Context setup, authentication, and basic workflows.
- [VCF CLI context command reference](<https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-consumption/latest/consumer-interfaces-in-vcf/installing-and-using-vcf-cli-v9/command-reference2/system-commands(1)/context2.html>) — Context creation, deletion, listing, refresh, and activation commands and flags.
- [Broadcom KB 417617: Unable to log into the supervisor with the vcf client](<https://knowledge.broadcom.com/external/article/417617>) — Supervisor login troubleshooting and use of --auth-type basic for vCenter SSO authentication.
