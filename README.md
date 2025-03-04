# Manual Cluster API Server Auditing

This document was created to outline effective an deployment template for logging activity within a manual provisioned Kubernetes cluster. As highlighted in the OWASP Top 10, establishing a robust and efficient logging system is essential for auditing events and communications across containerized deployments.

In the following sections, I will walk through the process of configuring custom logs tailored to specific kubectl API resources. By designing and implementing a structured logging strategy, we can capture various types of API requests, responses, and resource metadata—allowing for improved monitoring, security analysis, and vulnerability detection.

However do note this solution is not necessary when using clusters provisioned in managed Kubernetes environments. Each major cloud provider has custom solutions for native logging

For a free, easy-to-use lab environment to test this configuration, you can follow the link below to access Kubernetes playgrounds that simulate manually provisioned clusters, similar to those used in official Kubernetes certification exams:
- https://killercoda.com/playgrounds

*All Examples Used are stored in the repository*

---
### Creating our Policy YAML

After logging into killercoda. I used the CKA lab environment to stand up this configuration.

Use `ls` and `cd` into the control plane File system

We begin by creating a policy manifest yaml to be stored in the file system of our control plane node under the following path. For my example, I named mine `simple-policy.yaml`

*Directly creating the file in path using VIM*
```shell
sudo vim /etc/kubernetes/simple-policy.yaml
```

The below Yaml defines the parameters and the nature of the events to be recorded as logs under the following specifications:
- Excludes logging of received requests for efficiency.
- Logs responses from all pods
- Logs **metadata only** for pod logs, statuses, authenticated users, and non-resource API URLs.
- Logs requests **to ConfigMaps in `kube-system`** but not their responses.
- Logs metadata **for Secrets and ConfigMaps** (without exposing contents).

My Policy YAML is only an example. You can define the logging strategy needed per your organizations needs for this implementation.

*Courtesy of the Linux Foundation*
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods"]
  - level: Metadata
    resources:
      - group: ""
        resources: ["pods/log", "pods/status"]
  - level: Metadata
    userGroups: ["system:authenticated"]
    nonResourceURLs:
      - "/api*"
      - "/version"
  - level: Request
    resources:
      - group: ""
        resources: ["configmaps"]
    namespaces: ["kube-system"]
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
omitStages:
  - "RequestReceived"
```

---
### Updating the kube-apiserver container manifest

Next we need to update the API server Manifest to use our policy file to record logs and configure settings regarding the storage of our logs. 

Editing this YAML will temporarily reset our ability to use kubectl within our cluster while the configuration updates.

*The API server YAML is located in the following path:*
```yaml
/etc/kubernetes/manifests/kube-apiserver.yaml
```
##### Installing logging logic

Open the file; and within the block of `spec.containers.command` in alphabetical order compared to the rest of the default values we declare the following *Only add the commented lines*:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.30.1.2
    - --allow-privileged=true
    - --audit-log-maxage=7 #<-- Retain age in days
    - --audit-log-maxbackup=2 #<-- Max number to retain
    - --audit-log-maxsize=50 #<-- Meg size when to rotate
    - --audit-log-path=/var/log/audit.log #<-- path to logging file
    - --audit-policy-file=/etc/kubernetes/simple-policy.yaml #< Our policy manifest
```

This enables the API server to enforce the rules defined in our audit policy YAML and log recorded events to the specified file.

As Defined:
1. The **API server** is enforcing the policy.
2. The **audit policy YAML** defines the rules.
3. The **log file** is where events are recorded.
##### Mounting the policy manifest and log-file from host to the api container

Next, we define **volume mounts** and **host paths** to link our **audit policy YAML** and the **audit log file** from the control plane node. These mounts make the files accessible inside the `kube-apiserver` container.

**This setup ensures that the API server can:**  
- **Read** the audit policy directly from the host.  
- **Write** audit logs to a persistent location on the control plane node.

*Be sure to add theses mounts and paths in line with the already existing code*
```yaml
volumeMounts:
  - mountPath: /etc/kubernetes/simple-policy.yaml #<-- Use same file names here
    name: audit
    readOnly: true
  - mountPath: /var/log/audit.log #<-- And here
    name: audit-log
    readOnly: false

volumes:
  - hostPath: #<-- Add these eight lines
      path: /etc/kubernetes/simple-policy.yaml
      type: File
    name: audit
  - hostPath:
      path: /var/log/audit.log
      type: FileOrCreate #<-- Tells the server te create if file does not exist
    name: audit-log
```

---
##### Full Example 

See the below complete `kube-apiserver.yaml` manifest as placement guide for the resources we provisioned and changed. After Saving any updates if you encounter any errors be sure to check for alignment mis configurations, typos and spellcheck is your kubectl commands are no longer working. 

Once you've taken any corrective action or if you had a successful deployment, you'll be able to make calls to kubectl shortly after updating. 

*Complete `kube-apiserver.yaml`*
```yaml
controlplane:/etc/kubernetes/manifests$ cat kube-apiserver.yaml 
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 172.30.1.2:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.30.1.2
    - --allow-privileged=true
    - --audit-log-maxage=7 #<-- Retain age in days
    - --audit-log-maxbackup=2 #<-- Max number to retain
    - --audit-log-maxsize=50 #<-- Meg size when to rotate
    - --audit-log-path=/var/log/audit.log #<-- Where to log
    - --audit-policy-file=/etc/kubernetes/simple-policy.yaml #< Our policy manifest
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: registry.k8s.io/kube-apiserver:v1.32.1
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 172.30.1.2
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-apiserver
    readinessProbe:
      failureThreshold: 3
      httpGet:
        host: 172.30.1.2
        path: /readyz
        port: 6443
        scheme: HTTPS
      periodSeconds: 1
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 50m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 172.30.1.2
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
    - mountPath: /etc/kubernetes/simple-policy.yaml
      name: audit
      readOnly: true
    - mountPath: /var/log/audit.log
      name: audit-log
      readOnly: false
  hostNetwork: true
  priority: 2000001000
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
  - hostPath:
      path: /etc/kubernetes/simple-policy.yaml
      type: File
    name: audit
  - hostPath:
      path: /var/log/audit.log
      type: FileOrCreate
    name: audit-log
status: {}
```

##### To check your work

*Use the Tail Command we can view live writes from our api-server to the logging file*
```shell
tail -f /var/log/audit.log
```

---

And that’s it! Feel free to use this as a resource and a template to build **custom logging solutions** for your manually provisioned or bare-metal Kubernetes clusters. This guide provides a solid foundation, but there’s plenty of room for expansion based on your specific security, compliance, and observability needs.

This setup is just the beginning—tailor it to your infrastructure and security strategy to create a **resilient and scalable** logging framework.

---
Within the pursuit of happiness we must derive pleasure
![Reward](./demon/demon.png)
