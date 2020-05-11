# Pod Security Policy on EKS

![CARTOON](/_temp/pod-security-policy.jpeg)
```
Kubernetes is a platform for building platforms. It's a better place to start; not the endgame.
- Kelsey Hightower ( @kelseyhightower)
```
## What is EKS?
Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service running on AWS. It takes away the bulk of the pain of managing a Kubernetes service by running the master tier for you. As with all AWS services, security is a Shared Responsibility Model. Amazon ensure the security of the master tier, but what you run inside the cluster – that’s up to you.


## What is Pod Security Policy?
In Kubernetes, workloads are deployed as Pods, which expose a lot of the functionality of running Docker containers.
Pod Security Policies are cluster-wide resources that control security sensitive aspects of pod specification. PSP objects define a set of conditions that a pod must run with in order to be accepted into the system, as well as defaults for their related fields. PodSecurityPolicy is an optional admission controller that is enabled by default through the API, thus policies can be deployed without the PSP admission plugin enabled. This functions as a validating and mutating controller simultaneously.

**Pod Security Policies allow you to control:**

- The running of privileged containers
- Usage of host namespaces
- Usage of host networking and ports
- Usage of volume types
- Usage of the host filesystem
- A white list of Flexvolume drivers
- The allocation of an FSGroup that owns the pod’s volumes
- Requirements for use of a read only root file system
- The user and group IDs of the container
- Escalations of root privileges
- Linux capabilities, SELinux context, AppArmor, seccomp, sysctl profile

Lets approach this subject from a application standpoint. Most applications are deployed into EKS in form of deployments running pods. Or sample deployment will be such:
Assuming we have agreen-field EKS with no special security controls on cluster/namespaces :
In the manifest **alpine-restricted.yml** , we are defining a few security contexts at the pod and container level.
**runAsUser: 1000** means all containers in the pod will run as user UID 1000
**fsGroup: 2000** means the owner for mounted volumes and any files created in that volume will be GID 2000
**allowPrivilegeEscalation: false** means the container cannot escalate privileges
**readOnlyRootFilesystem: true** means the container can only read the root filesystem

![psp](/_temp/PSP.png)
To enable these PSPs we need to delete a PSP. You see, by default pods are in fact validated against a PSP by default – it’s just that it allows everything and is accessible to everyone. it’s called **eks.privileged**

Note that by doing this, you can bork your cluster – make sure you’re ready to [recreate it.](https://docs.aws.amazon.com/eks/latest/userguide/pod-security-policy.html)

```
kubectl delete psp eks.privileged
```

```
---
apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: alpine-restricted
        labels:
            app: alpine-restricted
    spec:
        replicas: 1
        selector:
        matchLabels:
            app: alpine-restricted
        template:
        metadata:
            labels:
            app: alpine-restricted
        spec:
            securityContext:
            runAsUser: 1000
            fsGroup: 2000
            volumes:
                - name: sec-ctx-vol
            emptyDir: {}
            containers:
                - name: alpine-restricted
            image: alpine:3.9
            command: ["sleep", "3600"]
            volumeMounts:
                - name: sec-ctx-vol
                    mountPath: /data/demo
            securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
```
Upon deployment it will fail to launch with the following error:
```
> kubectl apply -f alpine-restricted.yaml
> kubectl get deploy,rs,pod
# output

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/alpine-test   0/1     0            0           67s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.extensions/alpine-test-85c976cdd   1         0         0       67s

> kubectl describe replicaset.extensions/alpine-test-85c976cdd | tail -n3
# output

 Type     Reason        Age                    From                   Message
  ----     ------        ----                   ----                   -------
  Warning  FailedCreate  114s (x16 over 4m38s)  replicaset-controller  Error creating: pods "alpine-test-85c976cdd-" is forbidden: unable to validate against any pod security policy: []
```
Without Pod Security Policies, the replication controller cannot create the pod. Let's deploy a restricted PSP and create a ClusterRole and ClusterRolebinding, which will allow the replication-controller to use the PSP defined above.

```
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
    name: psp.restricted
    annotations:
        seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
        seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
spec:
    readOnlyRootFilesystem: true
    privileged: false
    allowPrivilegeEscalation: false
    runAsUser:
        rule: 'MustRunAsNonRoot'
    supplementalGroups:
        rule: 'MustRunAs'
        ranges:
            - min: 1
            max: 65535
    fsGroup:
        rule: 'MustRunAs'
    ranges:
        - min: 1
        max: 65535
    seLinux:
        rule: 'RunAsAny'
    volumes:
        - configMap
        - emptyDir
        - secret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
    name: psp:restricted
rules:
- apiGroups:
  - policy
  resourceNames:
  - psp.restricted
  resources:
  - podsecuritypolicies
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: psp:restricted:binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: psp:restricted
subjects:
  # Apply PSP to service account of the controllers in kube-system that bring up pods
  # DaemonSet is excluded since they are used to interact with the host
  - kind: ServiceAccount
    namespace: kube-system
    name: replication-controller
  - kind: ServiceAccount
    namespace: kube-system
    name: replicaset-controller
  - kind: ServiceAccount
    namespace: kube-system
    name: job-controller
```
the PSP **psp.restricted** have some restrictions for pods :

- the root filesystem must be **readonly**
- it doesn't allow for privileged containers
- privilege escalations are by forbidden **readOnlyRootFilesystem: true**
- the user must be nonroot **runAsUser: rule: 'MustRunAsNonRoot'**
- fsGroup and supplementalGroups cannot be root **defined allowed range**
- the pod can only use the volumes specified: **configMap, emptyDir and secret**
- in the annotations section there is a seccomp profile, which is used by containers. Due to these annotations, the PSP will mutate the podspec before deploying it. If we check the pod manifest in Kubernetes, we will see **seccomp** annotations defined there as well.

Re-deploy the **alpine-restricted.yml** and inspect the pod annotation
```
kubectl get pod/alpine-test-85c976cdd-k69cn -o jsonpath='{.metadata.annotations}'
# output
map[kubernetes.io/psp:psp.restricted seccomp.security.alpha.kubernetes.io/pod:docker/default]
```

We can see that there are two annotations set:
- kubernetes.io/psp:psp.restricted
- seccomp.security.alpha.kubernetes.io/pod:docker/default

The first is the PodSecurityPolicy(**psp.restricted**) used by the pod. 
The second is the seccomp profile used by the pod. Seccomp (secure computing mode) is a Linux kernel feature used to restrict the actions available inside a container.

Lets edit our deployment a bit and deploy a more privileged version: **alpine-privileged.yml**
```
 apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: alpine-privileged
   namespace: privileged
   labels:
     app: alpine-privileged
 spec:
   replicas: 1
  selector:
    matchLabels:
      app: alpine-privileged
  template:
    metadata:
      labels:
        app: alpine-privileged
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 2000
      volumes:
      - name: sec-ctx-vol
        emptyDir: {}
      containers:
      - name: alpine-privileged
        image: alpine:3.9
        command: ["sleep", "1800"]
        volumeMounts:
        - name: sec-ctx-vol
          mountPath: /data/demo
        securityContext:
          allowPrivilegeEscalation: true
          readOnlyRootFilesystem: false
```
This deployment want to start a pod with write permissions on the root filesystem, and to enable privilege escalation.

```
kubectl describe replicaset.extensions/alpine-privileged-7bdb64b569 -n privileged | tail -n3
# output
  Type     Reason        Age                   From                   Message
  ----     ------        ----                  ----                   -------
  Warning  FailedCreate  29s (x17 over 5m56s)  replicaset-controller  Error creating: pods "alpine-privileged-7bdb64b569-" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.readOnlyRootFilesystem: Invalid value: false: ReadOnlyRootFilesystem must be set to true spec.containers[0].securityContext.allowPrivilegeEscalation: Invalid value: true: Allowing privilege escalation for containers is not allowed]
```
A long string of error, but basically it states the following:  
The pod is created with the **replicaset-controller** service account, which allows use of the **psp.restricted** policy. Pod creation failed because podspec contains the **allowPrivilegeEscalation: true** and **readOnlyRootFilesystem: false** securityContexts - at this time, these are invalid values as far as **psp.restricted** is concerned.

To deploy this, let's make the deployment go through a service account that can allow it do get deployed. Making the following modification in the deployment manifest file, making it use the **privileged-sa** service account 
```
...
  template:
    metadata:
      labels:
        app: alpine-privileged
    spec:
      serviceAccountName: privileged-sa
      securityContext:
        runAsUser: 1
        fsGroup: 1
...
```
deploy a psp.privileged policy and create a Role and a Rolebinding to allow the privileged-sa to use the defined PSP
Create a separate namespace and a service account in it:
```
kubectl create sa privileged-sa -n privileged
```
next create the PSP and the role and role-binding to go with it:
```
apiVersion: policy/v1beta1
 kind: PodSecurityPolicy
 metadata:
   creationTimestamp: null
   name: psp.privileged
 spec:
   readOnlyRootFilesystem: false
   privileged: true
   allowPrivilegeEscalation: true
  runAsUser:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: psp:privileged
  namespace: privileged
rules:
- apiGroups:
  - policy
  resourceNames:
  - psp.privileged
  resources:
  - podsecuritypolicies
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp:privileged:binding
  namespace: privileged
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: psp:privileged
subjects:
  - kind: ServiceAccount
    name: privileged-sa
    namespace: privileged
```
The psp.privileged policy contains **readOnlyRootFilesystem: false** and **allowPrivilegeEscalation: true.** The **privileged-sa** service account in the privileged namespace allows us to use the **psp.privileged** policy, so, if we deploy the modified **alpine-privileged.yml**, the pod should start.
Deploy the pod and inspect the pod annotation:
```
kubectl get pod/alpine-privileged-667dc5c859-q7mf6 -n privileged -o jsonpath='{.metadata.annotations}'
# output
map[kubernetes.io/psp:psp.privileged]
```

## What if we have multiple PSP

When multiple policies are available, the pod security policy controller selects policies in the following order:

- If policies successfully validate the pod without altering it, they are used.
- In the event of a pod creation request, the first valid policy in alphabetical order is used.
- Otherwise, if there's a pod update request an error is returned, because pod mutations are disallowed during update operations.

we'll create two Pod Security Policies with RBAC:
```
apiVersion: policy/v1beta1
 kind: PodSecurityPolicy
 metadata:
   name: psp.1
   annotations:
     seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
     seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
 spec:
   readOnlyRootFilesystem: false
  privileged: false
  allowPrivilegeEscalation: true
  runAsUser:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.2
  annotations:
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
spec:
  readOnlyRootFilesystem: false
  privileged: false
  allowPrivilegeEscalation: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  volumes:
  - configMap
  - emptyDir
  - secret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: psp:multy
  namespace: multy
rules:
- apiGroups:
  - policy
  resourceNames:
  - psp.1
  - psp.2
  resources:
  - podsecuritypolicies
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp:multy:binding
  namespace: multy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: psp:multy
subjects:
  - kind: ServiceAccount
    name: replicaset-controller
    namespace: kube-system
```

there are two PSPs in psp-multiple.yaml, which are almost identical. Now, Role and RoleBinding in the multy namespace will allow us to use a predefined **psp.1** and **psp.2** for the replicaset-controller

```
kubectl get psp

# output
NAME             PRIV    CAPS   SELINUX    RUNASUSER          FSGROUP     SUPGROUP    READONLYROOTFS   VOLUMES
...
psp.1            false          RunAsAny   RunAsAny           RunAsAny    RunAsAny    true             configMap,emptyDir,secret
psp.2            false          RunAsAny   MustRunAsNonRoot   RunAsAny    RunAsAny    true             configMap,emptyDir,secret
...
```

Creating a deployment 

```
apiVersion: apps/v1
 kind: Deployment
 metadata:
   name: alpine-multy
   namespace: multy
   labels:
     app: alpine-multy
 spec:
   replicas: 1
  selector:
    matchLabels:
      app: alpine-multy
  template:
    metadata:
      labels:
        app: alpine-multy
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: alpine-multy
        image: alpine:3.9
        command: ["sleep", "2400"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
```
Checking the annotation again:
```
kubectl get pod alpine-multy-5864df9bc8-mgg68 -n multy -o jsonpath='{.metadata.annotations}'
# output
map[kubernetes.io/psp:psp.1 seccomp.security.alpha.kubernetes.io/pod:docker/default]
```
As you can see, this pod uses the psp.1 PodSecurityPolicy, which is the first valid policy in alphabetical order, though it is the second rule in the PSP selection.  

Let's modify **psp.2** by deleting seccomp's annotations:

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.2
spec:
  readOnlyRootFilesystem: false
```

delete and deploy the pod again after modifying the **psp.2** check the annotation again:
```
kubectl get pod alpine-multy-5864df9bc8-lj2fz -n multy -o jsonpath='{.metadata.annotations}'
# output
map[kubernetes.io/psp:psp.2]
```

Now, the pod uses **psp.2** due to the annotations being deleted, since, in this case, the policy successfully validated the pod without altering it; it's the 1st rule in PSP selection.

## Conclusion
The security model around nodes is well thought out and you cannot escalate cluster admin just by compromising a node. However, without PSPs, anything already running on that node is fair game. After configuring appropriate PSPs these vulnerabilities cannot be exploited.
