# Kubernetes Pod Cleanup Solution

## Overview

This project implements a custom solution for cleaning up pods in a Kubernetes cluster that are in a non-running state. Specifically, a Python script is used to periodically check for failed or pending pods in a designated namespace and delete them if they are older than 5 minutes. The solution is designed to maintain the health of the cluster by ensuring that non-running pods do not linger and consume resources unnecessarily.

This solution is deployed in an Amazon EKS (Elastic Kubernetes Service) cluster, using a pod in one namespace to monitor and clean up pods in another namespace.

## Project Structure

The project consists of the following Kubernetes YAML files:

1. `01-namespace.yml`: Creates two namespaces—`first-namespace` (where the cleaner pod will run) and `second-namespace` (where the pods to be monitored are located).
2. `02-serviceacc.yml`: Creates a ServiceAccount (`pod-cleaner-sa`) in the `first-namespace` to run the cleaner pod.
3. `03-role-rolebinding.yml`: Defines a Role and RoleBinding in `second-namespace` to grant the necessary permissions (list and delete pods) to the ServiceAccount running in `first-namespace`.
4. `04-config.yml`: Creates a ConfigMap containing the Python cleanup script (`cleanup.py`) that will be mounted into the cleaner pod.
5. `05-pod.yml`: Deploys the `pod-cleanup` pod in the `first-namespace` which runs the Python script periodically to clean up non-running pods in the `second-namespace`.
6. `06-secondpod.yml`: Creates two example pods in `second-namespace`:
   - `myapp-pod`: A normal pod with a working image.
   - `myapp-pod-error`: A pod with an incorrect image that simulates a failed pod, to be deleted by the cleanup script.

## Solution Design

The pod cleanup process is managed by the `pod-cleanup` pod, which runs in the `first-namespace`. The Python script in this pod performs the following tasks:

1. **Check pod status**: The script lists all pods in `second-namespace`.
2. **Filter non-running pods**: It filters pods that are in a `Failed`, `Pending`, or `Unknown` state.
3. **Check pod age**: If the non-running pod is older than 5 minutes, it is scheduled for deletion.
4. **Delete pods**: The script deletes the non-running pods using the Kubernetes API.

The script runs in a loop, checking and cleaning up pods every 5 minutes.

## Prerequisites

- **Amazon EKS Cluster**: Ensure that an EKS cluster has been created. You can follow the [official AWS EKS documentation](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) to set up the cluster.
- **kubectl**: Install `kubectl` to interact with your Kubernetes cluster.
- **AWS CLI**: Ensure that the AWS CLI is configured with the correct credentials and region for your EKS cluster.

## Steps to Deploy the Solution

1. **Set up the EKS cluster**:
   - Ensure that your EKS cluster is up and running. You can create the EKS cluster using the following command:
     ```bash
     eksctl create cluster --name <your-cluster-name> --region <your-region>
     ```

2. **Apply the namespaces**:
   - Apply the namespace configuration to create `first-namespace` and `second-namespace`:
     ```bash
     kubectl apply -f 01-namespace.yml
     ```

3. **Create the ServiceAccount**:
   - Apply the ServiceAccount configuration to create `pod-cleaner-sa` in `first-namespace`:
     ```bash
     kubectl apply -f 02-serviceacc.yml
     ```

4. **Create Role and RoleBinding**:
   - Grant the necessary permissions for cleaning up pods by applying the Role and RoleBinding:
     ```bash
     kubectl apply -f 03-role-rolebinding.yml
     ```

5. **Create the ConfigMap**:
   - Deploy the ConfigMap containing the Python cleanup script:
     ```bash
     kubectl apply -f 04-config.yml
     ```

6. **Deploy the cleanup pod**:
   - Deploy the `pod-cleanup` pod that will run the cleanup script:
     ```bash
     kubectl apply -f 05-pod.yml
     ```

7. **Deploy test pods in second-namespace**:
   - Deploy the two example pods (`myapp-pod` and `myapp-pod-error`) in `second-namespace`:
     ```bash
     kubectl apply -f 06-secondpod.yml
     ```

8. **Monitor the cleanup process**:
   - The `pod-cleanup` pod will periodically check for and delete any non-running pods in `second-namespace`. You can monitor its logs with:
     ```bash
     kubectl logs -f pod-cleanup -n first-namespace
     ```

## Python Cleanup Script

The cleanup logic is implemented in `cleanup.py`, which is mounted into the `pod-cleanup` pod via a ConfigMap. Below is a brief explanation of the key functions:

- **`delete_failed_pods(namespace)`**: This function lists all pods in the specified namespace and checks for non-running pods (Failed, Pending, Unknown) that are older than 5 minutes. Such pods are deleted using the Kubernetes API.

- **Main Loop**: The script runs in an infinite loop, checking and cleaning up pods every 5 minutes (300 seconds). It handles any errors that occur while listing or deleting pods.

## Troubleshooting

- **Error: Forbidden when deleting pods**:

Author
Paras Kamboj
  This error may occur if the Role or RoleBinding does not have sufficient permissions. Ensure that the Role has `list` and `delete` permissions on `pods` in `second-namespace`.

- **Python script not running in the pod**:
  Check the logs of the `pod-cleanup` pod for any errors related to the Python script execution:
  ```bash
  kubectl logs pod-cleanup -n first-namespace
