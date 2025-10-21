## Practical Guide: Managing DNS Configuration in an OpenShift Cluster with NMState and Automated Removal via ArgoCD Hooks (Updated with App of Apps)

## Introduction

Managing network configurations on Kubernetes cluster nodes, such as adding custom DNS resolvers, can be a challenging task. The **Red Hat NMState Operator (Node Network Configuration Policy)** simplifies this by allowing declarative network configuration management via Kubernetes Custom Resources (CRs).

This article demonstrates how to use NMState to inject custom DNS resolvers into OpenShift cluster nodes, leveraging **ArgoCD's App of Apps pattern** for the GitOps workflow. This pattern allows us to install the Operator, its instance, and the configuration in a single step. We address a crucial point: the necessity of ensuring the **cleanup** of these configurations after the ArgoCD Application is removed, as NMState does not perform this automatically. To solve this, we implement a **Cleanup Job** triggered via an ArgoCD PostDelete Hook, ensuring a complete rollback.

## Prerequisites

1.  Running OpenShift Cluster.
2.  ArgoCD installed and configured (to manage the Applications).
3.  NMState Operator (initial setup is done via the App of Apps).
4.  Git Repository structured as shown in Step 3.

## Step 1: NMState Configuration Structure (The Repository Layout)

To manage the installation and configuration life cycle separately, we will use three distinct ArgoCD Applications, all orchestrated by a fourth **Application of Applications (App of Apps)**.

The repository structure should reflect this separation:

```bash
└── openshift-nmstate
    ├── 01-operator
    │   ├── 01-application-operator.yaml  # ArgoCD App for Operator Installation
    │   └── manifests
    │       ├── clean-nncp-sa-cr-crb.yaml # ServiceAccount and RBAC for Cleanup Hook
    │       ├── namespace.yaml
    │       ├── operator-group.yaml
    │       └── subscription.yaml
    ├── 02-instance
    │   ├── 02-application-instance.yaml  # ArgoCD App for NMState Instance
    │   └── manifests
    │       └── instance.yaml             # NMState CR
    ├── 03-dns-custom
    │   ├── 03-application-dns-custom.yaml # ArgoCD App for DNS Configuration
    │   └── manifests
    │       ├── clean-nncp-job.yaml        # The PostDelete Hook Job (Cleanup)
    │       ├── master-dns-custom.yaml     # NodeNetworkConfigurationPolicy (Master)
    │       └── worker-dns-custom.yaml     # NodeNetworkConfigurationPolicy (Worker)
    └── extras
        ├── 00-app-of-apps-dns-custom.yaml      # Main App that deploys 01, 02, and 03
```

## Step 2: Configuring DNS and Cleanup Resources

This step involves creating the specific YAML files for the NNCP and the Cleanup Job.

### 2.1. DNS-Resolver Configuration (NNCP)

These files (`master-dns-custom.yaml` and `worker-dns-custom.yaml`) contain the desired DNS configuration using the `NodeNetworkConfigurationPolicy` (NNCP).

**(NNCP Example - `master-dns-custom.yaml`)**

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: master-dns-custom
spec:
  desiredState:
    dns-resolver:
      config:
        search:
          - cluster.local
        server:
          - 192.168.10.10
          - 192.168.10.11
  nodeSelector:
    node-role.kubernetes.io/master: ''
        
```

**(NNCP Example - `worker-dns-custom.yaml`)**

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: worker-dns-custom
spec:
  desiredState:
    dns-resolver:
      config:
        search:
          - cluster.local
        server:
          - 192.168.10.10
          - 192.168.10.11
  nodeSelector:
    node-role.kubernetes.io/worker: ''
        
```

-----

### 2.1.1. The Challenge of Configuration Removal

According to the official Red Hat documentation (e.g., [Red Hat Docs Link](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/kubernetes_nmstate/k8s-nmstate-updating-node-network-config#virt-example-nmstate-IP-management-dns_k8s-nmstate-updating-node-network-config)), simply deleting the NNCP that added the custom DNS configuration **does not remove the configuration from the node.**

To effectively revert the DNS changes, a subsequent NNCP must be applied with an identical structure but with empty values, explicitly telling NMState to remove the settings:

```yaml
# ...
    dns-resolver:
      config: {}
    interfaces: []
# ...
```

In an automated GitOps environment, forcing a manual step to apply this "cleanup" configuration after the ArgoCD Application is deleted is undesirable and error-prone. To maintain a fully automated lifecycle, we need a mechanism to execute this rollback step automatically.

This is where the **Cleanup Job** comes into play, as detailed below.

-----

### 2.2. The Cleanup Job and RBAC (PostDelete Hook)

The cleanup logic is separated into two files:

1.  **`clean-nncp-sa-cr-crb.yaml` (RBAC):** Contains the `ServiceAccount` (`nmstate-clean-job`) and its permissions, which are deployed by the **`01-operator`** Application.
2.  **`clean-nncp-job.yaml` (Job Hook):** Contains the `Job` definition, deployed by the **`03-dns-custom`** Application. The crucial ArgoCD annotations (`argocd.argoproj.io/hook: PostDelete` and `serviceAccountName: nmstate-clean-job`) ensure it runs after deletion with the necessary permissions.

**(Cleanup Job Snippet - `clean-nncp-job.yaml`)**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nmstate-cleanup
  namespace: openshift-nmstate 
  annotations:
    argocd.argoproj.io/hook: PostDelete  
    argocd.argoproj.io/hook-delete-policy: HookFailed
spec:
  template:
    spec:
      serviceAccountName: nmstate-clean-job # The ServiceAccount we created
      restartPolicy: Never # Ensures the Job does not restart the pod if it fails
      containers:
      - name: yaml-generator-executor
        image: registry.redhat.io/openshift4/ose-cli:latest # Image with the OpenShift CLI
        command: ["/bin/bash", "-c"]
        args:
          - | 

            # 1. Write the YAML content to the file
            cat <<EOF > /tmp/dns-custom-cleanup.yaml
            apiVersion: nmstate.io/v1
            kind: NodeNetworkConfigurationPolicy
            metadata:
              name: dns-custom-cleanup
              namespace: openshift-nmstate
            spec: 
              desiredState: 
                dns-resolver:
                  config: {}  
            EOF
            oc apply -f /tmp/dns-custom-cleanup.yaml
            sleep 30
            oc delete nncp dns-custom-cleanup 
            oc delete --namespace openshift-nmstate subscriptions.operators.coreos.com kubernetes-nmstate-operator
            echo "Removing NMState ClusterServiceVersion..."
            # You will need to get the exact name of the CSV, which changes with versions (e.g.: nmstate-operator.1.x.x)
            # A robust way is to list and filter:
            CSV_NAME=$(oc get csv -n openshift-nmstate -o=jsonpath='{.items[?(@.spec.displayName=="NMState")].metadata.name}' || true)
            if [ -n "$CSV_NAME" ]; then
                echo "Found CSV: $CSV_NAME. Deleting..."
                oc delete clusterserviceversion "$CSV_NAME" -n openshift-nmstate --wait=true --timeout=2m
            else
                echo "NMState CSV not found or already deleted."
            fi
            echo "Removing NMState CRDs..."
            oc -n openshift-nmstate delete nmstate nmstate
            oc delete --all deployments --namespace=openshift-nmstate          
            oc delete crd nmstates.nmstate.io
            oc delete crd nodenetworkconfigurationenactments.nmstate.io
            oc delete crd nodenetworkstates.nmstate.io
            oc delete crd nodenetworkconfigurationpolicies.nmstate.io
            oc delete namespace openshift-nmstate --wait=false
            # Add any other CRD that NMState might have installed            
            echo "NMState Operator removal complete."                    
```


## Step 3: Orchestration with App of Apps

Instead of synchronizing the three Applications (`01-operator`, `02-instance`, `03-dns-custom`) individually, we use the App of Apps pattern. The file `00-app-of-apps-dns-custom.yaml` is the **single Application** that the user will deploy.

**(App of Apps - `00-apps-of-app-dns.yaml`)**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: 00-app-of-apps-dns-custom
  namespace: openshift-gitops
spec:
  destination:
    namespace: openshift-gitops
    name: in-cluster  
  sources:
    - path: openshift-nmstate/01-operator/
      repoURL: https://github.com/marcosraposo/gitops-dns-clean
      targetRevision: HEAD
    - path: openshift-nmstate/02-instance/
      repoURL: https://github.com/marcosraposo/gitops-dns-clean
      targetRevision: HEAD   
    - path: openshift-nmstate/03-dns-custom/
      repoURL: https://github.com/marcosraposo/gitops-dns-clean
      targetRevision: HEAD          
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true   
```

> **Note:** The individual Application manifests (`01-application-operator.yaml`, etc.) must be configured to ensure dependency order, often achieved by referencing the Git repository root and the correct path to their respective `manifests` folder.

## Step 4: Deployment and Removal

### Deployment

1.  In ArgoCD, create a new Application pointing only to the `00-apps-of-app-dns.yaml` file (or the directory `/extras`).
2.  Synchronize the main Application (`nmstate-all-in-one`).
3.  **Result:** ArgoCD will recursively deploy the Operator (01), the Instance (02), and the DNS Configuration (03), ensuring the NMState environment is fully ready in one go.

### Removal and Automated Cleanup

1.  In ArgoCD, execute the **Delete** operation on the main Application (`nmstate-all-in-one`).
2.  ArgoCD will attempt to delete the child Applications (01, 02, 03).
3.  When deleting the **`03-dns-custom`** Application:
      * The NNCPs (`master-dns-custom.yaml`, `worker-dns-custom.yaml`) are deleted.
      * The **`nmstate-cleanup` Job** is triggered by the `PostDelete` hook.
      * The Job runs, applies a temporary NNCP with an empty DNS config (`config: {}`), forcing the nodes to revert the custom DNS settings to their default state.
      * The Job then deletes the temporary cleanup NNCP.

**Verification:**

After the deletion process is complete and the `nmstate-cleanup` Job finishes successfully:

1.  Check the Job logs to confirm the temporary NNCP was applied and deleted.
2.  Verify the `/etc/resolv.conf` on a cluster node. The custom DNS entries should be gone, confirming the successful automated rollback.

## Conclusion

By leveraging the **App of Apps pattern** in ArgoCD, we achieve streamlined, single-point deployment for complex installations like the NMState Operator. Crucially, by integrating the **PostDelete Hook Job**, we overcome the native limitations of the NMState operator concerning automated configuration cleanup, ensuring a **fully GitOps-compliant, safe, and automated lifecycle** for custom DNS configurations.