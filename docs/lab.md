# Build a Play with Kubernetes (PWK) cluster

## Pre-reqs

You will need a Docker Hub or GitHub account to use PWK.

## Build the cluster

1. Point your browser to [Play with Kubernetes](https://workshop.play-with-k8s.com/)
2. Login with Docker Hub or GitHub account
3. Click `+ ADD NEW INSTANCE` from the left-side panel
4. Complete step 1 and 2 shown in the terminal window on the right
  - **Step 1** Initialize the master node (this step may take a minute or two)
  - **Step 2** Initialize the Pod network
5. Copy the long `kubeadm join...` command that is displayed after the `kubeadm init` command completes
6. Add three more PWK nodes using the `+ ADD NEW INSTANCE` button on the left
7. Paste the `kubeadm join...` command into the three new nodes
8. Run `kubectl get nodes` from **node1** and verify that all 4 nodes are listed and showing as `Ready`
