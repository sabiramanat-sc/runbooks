# Runbook: NKP Bastion Host and Air-Gapped Nutanix Deployment & RX Lab Configuration

This runbook starts with bastion preparation and continues through NKP cluster creation, registry-sync recovery, self-managed pivot, Kommander verification, and UI access.

## RX Lab Configuration

## 1. Reserve RX Lab
- Reserve for 2 weeks.
- Configure workload:
  - AHV Version
  - AOS 7
  - CVM (64 GB RAM)
  - Rocky Image
- Assign workload to a cluster with **512+ GB RAM**.

## 2. Configure Networking
- Access Prism Element using the **CVM IP**.
- Configure **Primary** and **Secondary VLANs**.
- Use cluster configuration details for network settings.
- Configure **DHCP only on the Secondary VLAN**.
- Set the DHCP range according to the VLAN.

## 3. Deploy Prism Central
- Deploy through Prism Element.
- Access via:
  - **IP:** Prism Central IP

## 4. Default Credentials Prism Central
- **Username:** Admin
- **Password:** Nutanix/4u

## Required version pairing — do not mix versions

This runbook uses the exact pairing below:

```bash
export BUNDLE_VERSION="<Bundle_Version>"
export KUBERNETES_VERSION="NKP_Version"
export NKP_IMAGE="nkp-rocky-image-version"
```

Do not use an NKP `v2.17.1` bundle with this `v1.34.4` image. If changing `BUNDLE_VERSION`, replace the VM image with the image supported by that NKP release and verify the installed `kubeadm`, `kubelet`, and `kubectl` versions before deployment.

## 1. Prepare the bastion image

1. Download the NKP Rocky image from the [Nutanix Support Portal](https://portal.nutanix.com/page/downloads/?product=nkp&bit=NKP%20Operating%20System%20Images). Select the image for the same NKP release as `BUNDLE_VERSION`.
2. Upload the image to Prism Central under **Compute & Storage > Images**.
3. Create the bastion VM:
   * 2 vCPU and 8 GB RAM.
   * Clone the image disk and expand it to at least 128 GB.
   * Attach a network that reaches Prism Central, the internet, and the NKP node subnet.
   * Use Cloud-Init Linux / Custom Script for the script below.

## 2. Bastion Cloud-Init

Replace `<YOUR_SSH_PUBLIC_KEY>` with the bastion administrator's ED25519 public key.

```yaml
#cloud-config
users:
  - name: nutanix
    lock_passwd: false
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: [adm, sudo, wheel]
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 <YOUR_SSH_PUBLIC_KEY>
chpasswd:
  list: |
    nutanix:nutanix/4u
  expire: false
ssh_pwauth: true
yum_repos:
  docker-ce-stable:
    name: Docker CE Stable - $basearch
    baseurl: https://download.docker.com/linux/centos/9/$basearch/stable
    enabled: true
    gpgcheck: true
    gpgkey: https://download.docker.com/linux/centos/gpg
packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin
  - vim
  - bash-completion
runcmd:
  - systemctl enable --now docker
  - usermod -aG docker nutanix
  - curl wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/local/bin/yq
  - chmod +x /usr/local/bin/yq
```

## 3. Verify the bastion

```bash
ssh nutanix@<BASTION_IP>
ping -c 3 8.8.8.8
nslookup google.com
docker ps
yq --version      #you have to give permission first
kubectl version --client
```

Manual Docker Installation:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo usermod -aG docker nutanix
newgrp docker
```

## 4. Download and prepare the NKP air-gapped bundle

Run all commands from the bastion. `BUNDLE_VERSION` is the only version variable used in paths and commands.

```bash
cd ~
curl -fLO "https://downloads.d2iq.com/dkp/${BUNDLE_VERSION}/nkp-air-gapped-bundle_${BUNDLE_VERSION}_linux_amd64.tar.gz"
tar -xzf "nkp-air-gapped-bundle_${BUNDLE_VERSION}_linux_amd64.tar.gz"
cd "${HOME}/nkp-${BUNDLE_VERSION}/cli"
echo 'export PATH="$PATH:'"$(pwd)'"' >> ~/.bashrc
source ~/.bashrc
chmod +x ./nkp ./kubectl
sudo cp -p ./kubectl /usr/local/bin/kubectl
docker load -i "../konvoy-bootstrap-image-${BUNDLE_VERSION}.tar"
```

Confirm the CLI and bootstrap image are the expected release:

```bash
./nkp version
docker images | grep "konvoy-bootstrap.*${BUNDLE_VERSION}"
```

Before deployment, make both bundle tar files world-readable. This prevents the registry-sync container from failing with a permissions error.

```bash
chmod 644 ../container-images/*.tar
stat -c '%a %U:%G %s %n' ../container-images/*.tar
```

## 5. Configure the cluster

Replace only the placeholders below. Do not paste passwords into shared logs.

```bash
cd "${HOME}/nkp-${BUNDLE_VERSION}/cli"
export NUTANIX_USER="<<PC service account>>"
export NUTANIX_PASSWORD="<<PC service account password>>"
export NKP_CLUSTER_NAME="<<desired cluster name>>"
export CONTROLPLANE_VIP="<<available IP in cluster subnet>>"
export NUTANIX_CLUSTER="<<desired Prism Element cluster>>"
export NUTANIX_ENDPOINT="<<Prism Central IP>>"
export NUTANIX_SUBNET="<<Nutanix subnet name>>"
export SSH_PUBLIC_KEY="<<path to public key file>>"
export STORAGE_CONTAINER="<<storage container>>"
export METALLB_IP_RANGE="<<start-ip>>-<<end-ip>>"
export SERVICES_CIDR="10.96.0.0/12"
export PODS_CIDR="10.95.0.0/16"
export CP_COUNT=3; export CP_CPU=4; export CP_MEM=16; export CP_DISK=80
export WRK_COUNT=4; export WRK_CPU=8; export WRK_MEM=32; export WRK_DISK=100
```

The control-plane VIP and MetalLB addresses must be unused, reserved, and on the correct node subnet. Do not overlap MetalLB addresses with DHCP/IPAM allocations.

## 6. Create the cluster

```bash
./nkp create cluster nutanix -c "${NKP_CLUSTER_NAME}" \
  --bundle '../container-images/*.tar' \
  --control-plane-endpoint-ip "${CONTROLPLANE_VIP}" \
  --control-plane-prism-element-cluster "${NUTANIX_CLUSTER}" \
  --control-plane-subnets "${NUTANIX_SUBNET}" \
  --control-plane-vm-image "${NKP_IMAGE}" \
  --csi-storage-container "${STORAGE_CONTAINER}" \
  --endpoint "https://${NUTANIX_ENDPOINT}:9440" \
  --kubernetes-service-load-balancer-ip-range "${METALLB_IP_RANGE}" \
  --worker-prism-element-cluster "${NUTANIX_CLUSTER}" \
  --control-plane-replicas "${CP_COUNT}" --control-plane-memory "${CP_MEM}" \
  --control-plane-vcpus "${CP_CPU}" --control-plane-disk-size "${CP_DISK}" \
  --worker-replicas "${WRK_COUNT}" --worker-memory "${WRK_MEM}" \
  --worker-vcpus "${WRK_CPU}" --worker-disk-size "${WRK_DISK}" \
  --worker-subnets "${NUTANIX_SUBNET}" --worker-vm-image "${NKP_IMAGE}" \
  --ssh-public-key-file "${SSH_PUBLIC_KEY}" \
  --kubernetes-pod-network-cidr "${PODS_CIDR}" \
  --kubernetes-service-cidr "${SERVICES_CIDR}" \
  --insecure --airgapped --self-managed --timeout 60m
```

## 7. Monitor from a second terminal

```bash
cd "${HOME}/nkp-${BUNDLE_VERSION}/cli"
export BOOT="$PWD/${NKP_CLUSTER_NAME}-bootstrap.conf"
export WORKLOAD="$PWD/${NKP_CLUSTER_NAME}.conf"
export KUBECONFIG="$BOOT"
watch ./nkp describe cluster -c "${NKP_CLUSTER_NAME}"
kubectl get events -A --sort-by=.lastTimestamp
kubectl logs -n capx-system deploy/capx-controller-manager --since=30m
```

## 8. Registry-sync failure recovery

If NKP reports `Waiting for bundles to be pushed to internal registry` and `Job has reached the specified backoff limit`, first fix permissions and verify the mounted files.

```bash
FAILED_JOB="$(kubectl get jobs -n default -o name | grep "${NKP_CLUSTER_NAME}-registry-sync" | tail -1 | cut -d/ -f2)"
chmod 644 ../container-images/*.tar
BOOTNODE=konvoy-capi-bootstrapper-control-plane
docker exec "$BOOTNODE" sh -c 'ls -lh /opt/bundles/*.tar'
kubectl get pods -n registry-system
```

The original failed pod may already be deleted. Create a retry Job from the failed Job:

```bash
export RETRY="${NKP_CLUSTER_NAME}-registry-sync-retry-$(date +%H%M%S)"
kubectl get job "$FAILED_JOB" -n default -o json > /tmp/registry-sync-job.json
python3 - <<'PY'
import json, os
p='/tmp/registry-sync-job.json'; new=os.environ['RETRY']
with open(p) as f: j=json.load(f)
for k in ['uid','resourceVersion','creationTimestamp','generation','managedFields','ownerReferences','finalizers']: j['metadata'].pop(k,None)
j['metadata']['name']=new; j.pop('status',None); s=j['spec']; s.pop('selector',None); s.pop('manualSelector',None); s['backoffLimit']=0
for k in ['controller-uid','batch.kubernetes.io/controller-uid']: j['spec']['template']['metadata'].get('labels',{}).pop(k,None)
j['spec']['template']['metadata']['labels'].update({'job-name':new,'batch.kubernetes.io/job-name':new})
with open('/tmp/registry-sync-retry.json','w') as f: json.dump(j,f)
PY
kubectl create -f /tmp/registry-sync-retry.json
kubectl get pods -n default -l "batch.kubernetes.io/job-name=${RETRY}" -w
```

After a pod appears:

```bash
POD="$(kubectl get pods -n default -l "batch.kubernetes.io/job-name=${RETRY}" -o jsonpath='{.items[0].metadata.name}')"
kubectl logs "$POD" -n default --all-containers --prefix --tail=300
kubectl wait --for=condition=complete "job/${RETRY}" -n default --timeout=60m
```

Continue only when the retry Job shows `SUCCEEDED=1`. Do not rerun the original create command against an already-created cluster.

## 9. Self-managed pivot recovery

If the target cluster is healthy but Kommander waits for self-management, check for CAPI CRDs:

```bash
export KUBECONFIG="$WORKLOAD"
kubectl get nodes
kubectl get crd clusters.cluster.x-k8s.io
```

If the CRD is missing, install CAPI components and let the command finish:

```bash
./nkp create capi-components --kubeconfig "$WORKLOAD" --timeout 30m --verbose 6
kubectl --kubeconfig "$WORKLOAD" get crd clusters.cluster.x-k8s.io
```

Then move CAPI resources from bootstrap to the target cluster:

```bash
./nkp move capi-resources --from-kubeconfig "$BOOT" --to-kubeconfig "$WORKLOAD" \
  --namespace default --to-namespace default --verbose 6
```

## 10. Verify Kommander and access the UI

```bash
export KUBECONFIG="$WORKLOAD"
kubectl get nodes
kubectl get pods -A
kubectl -n kommander get job kommander-bootstrap
kubectl -n kommander get secret dkp-credentials
kubectl get helmreleases -A
```

Wait for `dkp-credentials` and the Kommander resources to exist, then run:

```bash
./nkp get dashboard --kubeconfig "$WORKLOAD"
```

Open the returned URL. If the URL uses a MetalLB IP and the browser cannot reach it, test from the browser host:

```bash
curl -vk --connect-timeout 5 https://<METALLB_IP>/dkp/kommander/dashboard/
```

For local testing:

```bash
kubectl -n kommander port-forward svc/kommander-traefik 8443:443
```

Open `https://127.0.0.1:8443/dkp/kommander/dashboard/`.
