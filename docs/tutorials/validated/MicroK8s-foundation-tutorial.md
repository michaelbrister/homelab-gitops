# MicroK8s Homelab Foundation with MetalLB and Traefik

This tutorial builds the Kubernetes foundation used by the rest of the homelab series. The finished system is a single-node MicroK8s cluster with persistent hostpath storage, a MetalLB address on the LAN, Traefik ingress, and local DNS names that route to Traefik.

The completed homelab used:

```text
Kubernetes distribution: MicroK8s 1.35.x
StorageClass:            microk8s-hostpath
Volume binding:          WaitForFirstConsumer
Ingress controller:      Traefik
IngressClass:            traefik
Traefik LAN address:     192.168.4.240
Final LAN zone:          *.home.mikebrister.com
```

> This is a single-node design. `microk8s-hostpath` stores data on that node; it does not protect workloads from disk or node loss.

## 1. Prerequisites

Prepare a Linux server with:

- a stable LAN address;
- enough disk for Kubernetes images and persistent volumes;
- a hostname resolvable on the LAN;
- outbound Internet access for pulling images and Helm charts;
- an unused LAN address for Traefik, `192.168.4.240` in this setup;
- `sudo` access.

The workstation used to manage the cluster should have `kubectl`, Helm, Git, and DNS access to the homelab zone.

## 2. Install MicroK8s

Install the same Kubernetes minor line used by the completed cluster:

```bash
sudo snap install microk8s --classic --channel=1.35/stable
```

Allow the current user to run MicroK8s commands:

```bash
sudo usermod -a -G microk8s "$USER"
mkdir -p ~/.kube
chmod 700 ~/.kube
```

Start a new login session so the new group membership is active. Then wait for the cluster:

```bash
microk8s status --wait-ready
microk8s kubectl get nodes -o wide
```

The node should report `Ready`.

## 3. Configure kubectl access

Export MicroK8s' kubeconfig:

```bash
microk8s config > ~/.kube/config
chmod 600 ~/.kube/config
```

Verify the standard client:

```bash
kubectl cluster-info
kubectl get nodes
```

All later tutorials use `kubectl` directly.

## 4. Enable DNS and persistent hostpath storage

Enable the required core add-ons:

```bash
microk8s enable dns
microk8s enable hostpath-storage
```

Verify DNS pods and the StorageClass:

```bash
kubectl get pods -n kube-system
kubectl get storageclass
```

Expected storage information:

```text
NAME                          PROVISIONER            VOLUMEBINDINGMODE
microk8s-hostpath (default)   microk8s.io/hostpath   WaitForFirstConsumer
```

`WaitForFirstConsumer` means a new PVC may remain `Pending` until a pod that uses it is scheduled. This is normal.

## 5. Enable MetalLB

Choose an unused address range on the same LAN as the cluster node. The completed setup assigned Traefik `192.168.4.240`; reserve a small range containing that address.

For example:

```bash
microk8s enable metallb:192.168.4.240-192.168.4.250
```

Verify MetalLB:

```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspools,l2advertisements -n metallb-system
```

All MetalLB pods should be running and the address pool should contain the configured LAN range.

## 6. Enable Traefik

Enable the MicroK8s community add-ons repository and Traefik:

```bash
microk8s enable community
microk8s enable traefik
```

Verify its workload, service, and ingress classes:

```bash
kubectl get pods -A | grep -i traefik
kubectl get service -A | grep -i traefik
kubectl get ingressclass
```

The later application manifests use:

```yaml
spec:
  ingressClassName: traefik
```

Wait for the Traefik LoadBalancer service to receive `192.168.4.240`:

```bash
kubectl get service -A -w
```

## 7. Configure LAN DNS

Create a wildcard record in the DNS server used by LAN clients:

```text
*.home.mikebrister.com -> 192.168.4.240
```

Use the `home.mikebrister.com` subdomain from the beginning so the later certificate tutorial can issue publicly trusted certificates while LAN DNS continues to resolve services privately.

Verify from the management workstation:

```bash
dig +short test.home.mikebrister.com
```

Expected:

```text
192.168.4.240
```

This is split DNS: LAN clients resolve application names to the private MetalLB address. The applications do not need public inbound exposure.

## 8. Validate ingress with a test workload

Create a temporary namespace and web server:

```bash
kubectl create namespace ingress-test

kubectl create deployment echo \
  -n ingress-test \
  --image=nginx:alpine

kubectl expose deployment echo \
  -n ingress-test \
  --port=80
```

Create `/tmp/ingress-test.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: ingress-test
spec:
  ingressClassName: traefik
  rules:
    - host: test.home.mikebrister.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f /tmp/ingress-test.yaml
kubectl get pods,service,ingress -n ingress-test
```

Test from the workstation:

```bash
curl -I http://test.home.mikebrister.com
```

An HTTP response confirms the complete route:

```text
LAN DNS
   |
   v
192.168.4.240
   |
   v
MetalLB -> Traefik -> Ingress -> Service -> Pod
```

## 9. Validate persistent storage

Create `/tmp/storage-test.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-test
  namespace: ingress-test
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: microk8s-hostpath
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-test
  namespace: ingress-test
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: busybox:1.37
      command: ["sh", "-c", "echo persistent > /data/check && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: storage-test
```

Apply and verify:

```bash
kubectl apply -f /tmp/storage-test.yaml
kubectl get pvc,pod -n ingress-test -w
kubectl exec -n ingress-test storage-test -- cat /data/check
```

Expected value:

```text
persistent
```

Delete and recreate only the pod from the same manifest, then verify the file still exists. This confirms persistence across pod replacement.

## 10. Cleanup the test resources

```bash
kubectl delete namespace ingress-test
```

The PVC and its dynamically provisioned test volume are removed with the namespace. Do not use this cleanup pattern for namespaces containing real data.

## 11. Final validation

```bash
microk8s status
kubectl get nodes
kubectl get pods -A
kubectl get storageclass
kubectl get ingressclass
kubectl get service -A | grep -i LoadBalancer
dig +short argocd.home.mikebrister.com
```

The foundation is ready when:

- the node is `Ready`;
- DNS and storage pods are healthy;
- `microk8s-hostpath` is available and default;
- Traefik is reachable at `192.168.4.240`;
- `traefik` is a valid IngressClass;
- the wildcard homelab DNS record resolves to the Traefik address.

## 12. Recovery notes

After a normal server restart:

```bash
microk8s status --wait-ready
kubectl get nodes
kubectl get pods -A
kubectl get service -A | grep -i traefik
```

The Kubernetes objects persist in MicroK8s and hostpath volumes remount from the node. A failure of the node or its disk can still destroy those volumes, so later application tutorials should not describe hostpath as highly available storage.

## 13. Final architecture

```text
Management workstation
        |
        | *.home.mikebrister.com
        v
LAN DNS -> 192.168.4.240
                    |
                    v
                 MetalLB
                    |
                    v
                 Traefik
                    |
                    v
              Kubernetes Ingress
                    |
                    v
            Services and workloads

Persistent workloads
        |
        v
microk8s-hostpath -> local node disk
```
