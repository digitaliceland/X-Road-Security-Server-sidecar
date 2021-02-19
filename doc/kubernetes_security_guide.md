# Kubernetes Security Server Sidecar Security User Guide <!-- omit in toc -->

## Version history <!-- omit in toc -->

 Date       | Version | Description                                                     | Author
 ---------- | ------- | --------------------------------------------------------------- | --------------------
 25.01.2021 | 1.0     | Initial version                                                 | Alberto Fernandez Lorenzo

# Table of contents
* [1 Introduction](#1-introduction)
   * [1.1 Target Audience](#11-target-audience)
* [2 Reference Data](#2-reference-data)
* [3 Handling passwords and secrets](#3-handling-passwords-and-secrets)
   * [3.1 Secrets as volume](#31-secrets-as-volume)
   * [3.2 Secrets as environment variables](#32-secrets-as-environment-variables)
* [4 User accounts](#4-user-accounts)
   * [4.1 Create a kubeconfig](#41-create-a-kubeconfig)
   * [4.2 Grant Cluster access to users](#42-grant-cluster-access-to-users)
   * [4.3 Restrict namespace access to a cluster](#43-restrict-namespace-access-to-a-cluster)
   * [4.4 Kubernetes Dashboard](#44-kubernetes-dashboard)
* [5 Network policies](#5-network-policies)
   * [5.1 Create Network policies](#51-create-network-policies)
* [6 Pod Security Policies](#6-pod-security-policies)
   * [6.1 Pod Security Polices in AWS EKS](#61-pod-security-policias-in-aws-eks)
   * [6.2 Creating a Pod Security Policy](#62-creating-a-pod-security-policy)
* [7 Monitoring](#7-monitoring)
* [8 Backups](#8-backups)
* [9 Message logs](#9-message-logs)


# 1 Introduction
## 1.1 Target Audience
The intended audience of this Security User Guide are X-Road Security Server system administrators responsible for installing and using X-Road Security Server Sidecar.
This document will discuss how to secure the installation of a Security Server Sidecar cluster explained in the [Kubernetes User Guide](https://github.com/nordic-institute/X-Road-Security-Server-sidecar/blob/master/doc/kubernetes_security_server_sidecar_user_guide.md).
The document is intended for readers with a moderate knowledge of Linux server management, computer networks, Docker, Kubernetes, AWS EKS and X-Road.

# 2 Reference Data

Please check the Reference data in the [Kubernetes User Guide](https://github.com/nordic-institute/X-Road-Security-Server-sidecar/blob/XRDDEV-1419/doc/kubernetes_security_server_sidecar_user_guide.md#44-reference-data).


**Ref** | **Value**                                | **Explanation**
------- | ----------------------------------------- | ----------------------------------------------------------
4.1    | &lt;role arn&gt;                    | The ARN of the IAM role to add.
4.2    | &lt;kubernetes groups&gt;            | A list of groups within Kubernetes to which the role is mapped. Typically "system:masters" and "system:nodes".
4.3    | &lt;user arn&gt;                    | The ARN of the IAM user to add.
4.4    | &lt;user name&gt;                    | The user name within Kubernetes to map to the IAM user.
4.5    | &lt;networkpolicy name&gt;                    | Unique name that identifies a NetworkPolicy inside a namespace.

# 3 Handling passwords and secrets
Kubernetes Secrets let you store and manage sensitive information in a safer way than putting it verbatim in a Pod definition or container image. For example, for the scenario [Multiple Pods using a Load Balancer](https://github.com/nordic-institute/X-Road-Security-Server-sidecar/blob/XRDDEV-1419/doc/kubernetes_security_server_sidecar_user_guide.md#23-multiple-pods-using-a-load-balancer) it is recommended to use secrets to store the environmental variables for the database password and the SSH keys using for the communication between the Primary and Secondary pods.
The Secrets can be used with a Pod in three ways:
- As files in a volume mounted on one or more containers.
- As container environment variable.
- By the kubectl when pulling images for the Pod.

For the Security Server Sidecar container, the first option is used to store the SSH keys and the second option is used to store the database password. The third option is only required if the Sidecar image would be in a private repository, which is not the case.

## 3.1 Secrets as volume
If you don't have an existing SSH key, you can create one by running:
```
ssh-keygen -f /path/to/.ssh/
```
Then create a Kubernetes Secret to store the SSH key by running (**Reference Data: 3.1, 3.14**);
```
kubectl create secret generic <secret name> --from-file=private-key=/path/to/.ssh/id_rsa --from-file=public-key=/path/to/.ssh/id_rsa.pub --namespace=<namespace name>
```

We can then consume these secrets as a volume in Kubernetes, i.e. by including the Secret under the definition of volumes in the Pod deployment manifest. We should define the key name, path and permissions, and then mount the volume on a path inside the container.

Below is an example of using Secrets in a Pod deployment manifest file (**Reference Data: 3.13, 3.14**):
``` yaml
[...]
volumes:
- name: <manifest volume name>
  secret:
    secretName: <secret name>
    items:
    - key: public-key
      path: id_rsa.pub
      mode: 0644
    - key: private-key
      path: id_rsa.pub
      mode: 0644
[...]
   volumeMounts:
   - name: <manifest volume name>
     mountPath: "/etc/.ssh/"
[...]
```

## 3.2 Secrets as environment variables

This example shows how to create a secret for the Security Server Sidecar as environment variables with sensitive data.
- Create a manifest file called for example 'secret-env-variables.yaml' and fill it with the desired values of the environment variables ( **Reference Data: 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 1.10**):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-sidecar-variables
  namespace: <namespace_name>
type: Opaque
stringData:
  XROAD_TOKEN_PIN: "<token pin>"
  XROAD_ADMIN_USER: "<admin user>"
  XROAD_ADMIN_PASSWORD: "<admin password>"
  XROAD_DB_HOST: "<database host>"
  XROAD_DB_PWD: "<database password>"
  XROAD_DB_PORT: "<database port>"
  XROAD_LOG_LEVEL: "<xroad log level>"
```
- Apply the manifest:
```bash
$ kubectl apply -f secret-env-variables.yaml
```


Then we can consume the Secrets as environment variables by modifying the deployment Pod manifest of each container that requires the secret as follows (the Secret key becomes the environment variable name in the Pod):
``` yaml
[...]
containers:
 - name: security-server-sidecar
   image: niis/xroad-security-server-sidecar:latest
   imagePullPolicy: "Always"
   envFrom:
   - secretRef:
     name: secret-sidecar-variables
[...]
```

Alternatively, if we don't want to include all the environment variables in a single Secret, we can reference it by key in the environment variable definition. Below is an example of how to do that with the variable "XROAD_DB_PWD":

``` yaml
[...]
containers:
 - name: security-server-sidecar
   image: niis/xroad-security-server-sidecar:latest
   imagePullPolicy: "Always"
   env:
   - name: XROAD_TOKEN_PIN
     value: "<token pin>"
   - name: XROAD_ADMIN_USER
     value: "<admin user>"
   - name: XROAD_ADMIN_PASSWORD
     value: "<admin password>"
   - name: XROAD_LOG_LEVEL
     value: "<xroad log level>"
   - name: XROAD_DB_HOST
     value: "<database host>"
   - name: XROAD_DB_PORT
     value: "<database port>"
   - name: XROAD_DB_PWD
     valueFrom:
       secretKeyRef:
         name: secret-sidecar-variables
         key: XROAD_DB_PWD
   - name: XROAD_DATABASE_NAME
     value: "<database name>"
[...]
```

# 4 User accounts
Amazon EKS uses IAM to provide authentication to your Kubernetes cluster through the `aws eks get-token` command (available in AWS CLI version 1.16.156 or later, or on the AWS IAM Authenticator for Kubernetes). However, for authorization, it still relies on native Kubernetes Role Based Access Control (RBAC). This means that IAM is only used for authentication of valid IAM entities. All the permissions required for interacting with the Kubernetes API of your Amazon EKS cluster are managed through the native Kubernetes RBAC system.

## 4.1 Create a kubeconfig
A kubeconfig file is a file used to configure access to Kubernetes when used in conjunction with the kubectl command-line tool.
To create a kubeconfig file, we should first be authenticated through the AWS CLI. More information on how to authenticate with AWS CLI can be found [here](https://aws.amazon.com/premiumsupport/knowledge-center/authenticate-mfa-cli/).
Then, open a terminal and run the following command (**Reference Data: 3.21, 3.22**):
```
aws eks --region <cluster region> update-kubeconfig --name <cluster name>
```

We can test the configuration by running the following command (you should see one Kubernetes service):
```
kubectl get svc

NAME                               TYPE       AGE     CLUSTER-IP      EXTERNAL-IP                                                                  PORT(S)                                                                                                                                          
kubernetes                         ClusterIP  190d    10.100.0.1      <none>                                                                          443/TCP            
```

Verify that the kubeconfig file exists by running:
```
cat /root/.kube/config
```

## 4.2 Grant cluster access to users
When we create an Amazon EKS cluster, the IAM user or role entity, such as a federated user that creates the cluster, is automatically granted system:masters permissions in the cluster's RBAC configuration on the control plane. This IAM entity does not appear in the ConfigMap or any other visible configuration, so make sure to keep track of which IAM entity originally created the cluster. To grant additional AWS users or roles the ability to interact with your cluster, we must edit the aws-auth ConfigMap within Kubernetes.
Below are the steps to add IAM users or roles to your cluster:

- Check if you have already applied the 'aws-auth' ConfigMap:
```
kubectl describe configmap -n kube-system aws-auth
```
- If you don't have any ConfigMap you can download it by running:
```
curl -o aws-auth-cm.yaml https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/aws-auth-cm.yaml
```
- Edit the ConfigMap file to introduce the InstanceNodeARN and then apply the changes by running:
```
kubectl apply -f aws-auth-cm.yaml
```

- To add an IAM user or role to an Amazon EKS cluster, edit the 'aws-auth' ConfigMap by running:
```
kubectl edit -n kube-system configmap/aws-auth
```

- Add your IAM users, roles, or AWS accounts to the ConfigMap file (**Reference Data: 4.1, 4.2, 4.3, 4.4**):
``` yaml
apiVersion: v1
data:
  mapRoles: |
    - rolearn: <role arn>
      username: <system:node:{{EC2PrivateDNSName}}>
      groups:
        - <kubernetes group>
        - <kubernetes group>
  mapUsers: |
    - userarn: <user arn>
      username: <user name>
      groups:
        - <kubernetes group>
        - <kubernetes group>
    - userarn: <user arn>
      username: <user name>
      groups:
        - <kubernetes group>
```
To add an IAM role: add the role details to the mapRoles section of the ConfigMap under data or add the whole section if it does not already exist in the file, as shown above.
To add an IAM user: add the user details to the mapUsers section of the ConfigMap under data or add the whole section if it does not already exist in the file, as shown above.

- Save the ConfigMap file and exit the text editor.

## 4.3 Restrict namespace access to a cluster
If the same Cluster is shared by different developers and teams, it is advisable to create isolated environments by creating namespaces and restrict access within each namespace. More information about how to assign permissions to a namespace in an AWS EKS Cluster on the [AWS Documentation](https://aws.amazon.com/premiumsupport/knowledge-center/eks-iam-permissions-namespaces/).

## 4.4 Kubernetes Dashboard
Dashboard is a web-based Kubernetes user interface. You can use Dashboard to deploy containerized applications to a Kubernetes cluster, troubleshoot your containerized application, and manage the cluster resources.

The Dashboard UI is not deployed by default. To deploy it, run the following command:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

You can access Dashboard using the kubectl command-line tool by running the following command:
```
kubectl proxy
```

Kubectl will make Dashboard available at http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/.

In the login view, you will be required to enter a token, as described in section [4 User Accounts](#4-user-accounts). AWS EKS provides authentication to the Kubernetes Cluster through the `aws eks get-token` command, so in order to get the token you can run (**Reference Data: 3.21**):
```
aws eks get-token --cluster-name <cluster name>
```

Installing the Kubernetes Dashboard could have potential security risks:
- The installation recommended in the EKS docs tells users to authenticate when connecting to the dashboard by fetching the authentication token for the dashboard’s cluster service account, which, again, may have cluster-admin privileges. That means a service account token with full cluster privileges and whose use cannot be traced to a human is now floating around outside the cluster.
- Everyone who can access the dashboard can make any queries or changes permitted by the service’s RBAC role.
- The Kubernetes Dashboard has been the subject of a number of CVEs. Because of its access to the cluster’s Kubernetes API and its lack of internal controls, vulnerabilities can be extremely dangerous.

# 5 Network policies
Network policies are objects which allow to explicitly state which traffic is permitted between groups of pods and other network endpoints in a Kubernetes cluster so that all non-conforming traffic will be blocked. It is a Kubernetes equivalent of a cluster-level firewall.

Each network policy specifies a list of allowed (ingress and egress) connections for all the group of pods specified using pod selectors and labels. If at least one network policy is applicable to a pod then the pod is considered isolated and are allowed to make or accept the connections listed in the network policy. On the contrary, if no network policies are applicable to a pod, then all network connections to and from it are allowed. Therefore, it's strongly recommended to implement network policies to restrict the attack surface if there are services exposed to the Internet.

The most relevant network policies to enforce are:
- Explicitly allow internet access for pods that need it: We can do that by setting the flag `allow-internet-access` to true.
- Explicitly allow necessary pod-to-pod communication: If we don't know on beforehand which pods need to communicate, we can allow traffic between all pods which belong to the same namespace or which have a specific label. We can also allow pods from a deployment A to communicate with pods from a deployment B or from a namespace A to a namespace B.

Network Policies are similar to AWS security groups in the sense that allows creating network ingress and egress rules. The difference with Network Policies is that it allows assigning them to pods using pod selectors and labels instead of instances of an AWS security group.

Network Policies can be configured in AWS with a network provider with network policy support, such as Calico, Cilium, Kube-router, Romana, Weave Net. In this case, we will be using Calico.

To install Calico, apply the Calico manifest by running:
```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.6/config/v1.6/calico.yaml
```

Verify the installation by running:
```
kubectl get daemonset calico-node --namespace kube-system
```

## 5.1 Create Network policies

In this example, it will be shown how to isolate the Primary Pod described in [Multiple Pods scenario](https://github.com/nordic-institute/X-Road-Security-Server-sidecar/blob/master/doc/kubernetes_security_server_sidecar_user_guide.md#426-multiple-pods-using-a-load-balancer-deployment) so that it only allows traffic from the Secondary Pods through port 22.

- Modify the Primary Pod manifest adding the label "role:primary" to identify it:
``` yaml
[...]
metadata:
  name: <service name>
  labels:
    run:  <service selector>
    role: primary
[...]
```

- Modify the Secondary Pod manifest adding the label "role:secondary" to identify it:
``` yaml
[...]
spec:
  selector:
    matchLabels:
      run: <service selector>
  replicas: <number replicas>
  template:
    metadata:
      labels:
        run: <pod name>
        role: secondary
[...]
```
- Apply the changes in the manifests.

- Create a NetworkPolicy to deny all the ingress traffic for the Primary Pod (**Reference Data: 3.1, 4.5**):
``` yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: <namespace name>
  name: <network policy name>
spec:
  podSelector:
    matchLabels:
      role: primary
  policyTypes:
  - Ingress
```

- Apply the manifest:
```
kubectl apply -f /path/to/file.yaml

```

- After all the ingress traffic is denied, create a network policy that allows the traffic from the Secondary Pods to the Primary Pod through the port 22 (**Reference Data: 3.1, 4.5**):
``` yaml

kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: <namespace name>
  name:  <network policy name>
spec:
  podSelector:
    matchLabels:
      role: primary
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: secondary
     ports:
       - protocol: TCP
         port: 22
```

- Apply the manifest:
```
kubectl apply -f /path/to/file.yaml

```

- List the NetworkPolicies by running (**Reference Data: 3.1):
```
kubectl get networkpolicies -n <namespace name>

```

- Delete the NetworkPolicies by running:
```
kubectl delete -f /path/to/file.yaml

```

# 6 Pod Security Policies

Pod Security Policies are a cluster-level resource which holds a collection of flags that control security aspects of the pod specification. These flags define the conditions with which a pod should be run within the cluster when created. If a request for creating a pod does not meet these conditions, the request is rejected and the pod is not created.

The most relevant pod security policies to enforce are:
- Disable privileged containers: We can disable them for every installation by setting the flag `privileged: false`.
- Prevent privilege escalation: We can disable it by setting the flag `allowPrivilegeEscalation: false`. This flag is recommended to set even if the container is run as a non-root user.
- Enforce non-root users: Prevent privilege escalation: We can disable it by setting the flag `allowPrivilegeEscalation: false`. This flag is recommended to set even if the container is run as a non-root user.We can enforce to run the container as non-root user by setting the rule MustRunAsNonRoot under the flag runAsUser. However, running the sidecar requires the container to run as root so we cannot use MustRunAsNonRoot for now.
- Prevent hostPath volumes: Mounting volumes using hostPath poses a serious risk of accessing the host file system by attackers. Nevertheless, if we need to use them, we can allow only with specified directories and permissions with the flag allowedHostPaths.

## 6.1 Pod Security Polices in AWS EKS
Kubernetes does not enable pod security policies by default. To enforce them every time a new pod is created in the cluster, you need to enable the Pod Security Policy at the cluster level via an admission controller. AWS EKS from version 1.13 supports the Kubernetes admission controller with a default privileged policy named eks.privileged. We can verify that the default policy exist by running:
```
kubectl describe psp eks.privileged
```
The eks.privileged default Pod Security Policy is equivalent to run Kubernetes with the PodSecurityPolicy controller disabled. Also, any authenticated users will be able to create any pods on the cluster with the eks.privileged policy. We can verify that by running the following command:
```
kubectl describe clusterrolebindings eks:podsecuritypolicy:authenticated
```

## 6.2 Creating a Pod Security Policy

Prior to creating a Pod Security Policy, we should first create a dedicated namespace, as well as a service account and role binding to authorize a non-admin user to use that policy. For example, we can create a namespace called sidecar-psp-restrictive and a service account called sidecar-eks-user and its respective role binding under that namespace with the following commands:
```
kubectl create ns sidecar-psp-restrictive

kubectl -n sidecar-psp-restrictive create sa sidecar-eks-user
```

Then we should proceed to create a Pod Security Policy with the recommended flags. At the same time, it's also important to make sure you authorize users and service accounts to use the Pod Security Policy. We can do that by creating a Role/ClusterRole that allows a user, in this case named sidecar-eks-user, to use the policy, in this case named eks.restrictive, and a RoleBinding/ClusterRoleBinding to bind to the role in the namespace, in this case named sidecar-psp-restrictive.

Below is an example of the Pod Security Policy Role and RoleBinding configuration in yaml file named sidecar-restrictive-psp.yaml:

``` yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: eks.restrictive
  annotations:
    kubernetes.io/description: 'eks.restrictive policy in sidecar-psp-restrictive namespace'
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
  labels:
    kubernetes.io/cluster-service: "true"
    eks.amazonaws.com/component: pod-security-policy
spec:
  privileged: false
  allowPrivilegeEscalation: false
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
    - 'awsElasticBlockStore'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  hostPorts:
      - min: 1025
        max: 65535
  runAsUser:
    # Running the sidecar requires the container to run as root so we cannot use MustRunAsNonRoot.
    rule: 'RunAsAny'
  seLinux:
    # Assuming the nodes are using AppArmor rather than SELinux
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group
      - min: 1
        max: 65535
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sidecar:psp:unprivileged
  namespace: sidecar-psp-restrictive
  labels:
      kubernetes.io/cluster-service: "true"
      eks.amazonaws.com/component: pod-security-policy
rules:
- apiGroups:
  - policy
  resourceNames:
  - eks.restrictive
  resources:
  - podsecuritypolicy
  verbs:
  - use
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sidecar-user:psp:unprivileged
  namespace: sidecar-psp-restrictive
  annotations:
    kubernetes.io/description: 'Allow service account sidecar-eks-user to use eks.restrictive psp in sidecar-psp-restrictive namespace.'
  labels:
    kubernetes.io/cluster-service: "true"
    eks.amazonaws.com/component: pod-security-policy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sidecar:psp:unprivileged
  subjects:
  - kind: ServiceAccount
    name: sidecar-eks-user
    namespace: sidecar-psp-restrictive
```

Then, you can apply the Pod Security Policy and its corresponding Role and RoleBinding configuration:
```
kubectl -n sidecar-psp-restrictive apply -f sidecar-restrictive-psp.yaml
```

It's worth noting that once we enable a Pod Security Policy, the admission controller will enforce it when creating or updating pods, otherwise the pod won't be created. However, if you modify the Pod Security Policy after the pods are already running, those that violate the policy as a consequence of this modification will not be shut down. In case of multiple Pod Security Policies available, the admission controller will select the first one alphabetically sorted by their name.

After the Pod Security Policy is created, you can verify that the sidecar-eks-user under the namespace sidecar-psp-restrictive can use the eks.restrictive Pod Security Policy with the following command:
```
kubectl --as=system:serviceaccount:sidecar-psp-restrictive:sidecar-eks-user -n sidecar-psp-restrictive auth can-i use podsecuritypolicy/eks.restrictive
```

Now the user sidecar-eks-user should be able to create pods that match the conditions on the Pod Security Policy eks.restrictive.


# 7 Monitoring
The following steps are recommended for monitoring using AWS CloudWatch so that we can detect potential security risks in your Cluster.

- **Collect control panel logs**: The control plane logs capture Kubernetes audit events and requests to the Kubernetes API server, among other components. Analysis of these logs will help detect some types of attacks against the cluster, and security auditors will want to know that you collect and retain this data.
AWS EKS Clusters can be configured to send control panel logs to Amazon CloudWatch. At a minimum, you will want to collect the following logs:
 * api - the Kubernetes API server log.
 * audit - the Kubernetes audit log.
 * authenticator - the EKS component used to authenticate AWS IAM entities to the Kubernetes API.

- **Monitor container and cluster performance for anomalies**:  Irregular spikes in application load or node usage can be a signal that an application may need programmatic troubleshooting, but they can also signal unauthorized activity in the cluster.  Monitoring key metrics provides critical visibility into your workload’s functional health and that it may need performance tuning or that it may require further investigation. For collecting these metrics, it is required to set up Amazon CloudWatch Container Insights for your cluster.

- **Monitor Node (EC2 Instance) Health and Security**: EKS provides no automated detection of node issues. Changes in node CPU, memory, or network metrics that do not correlate with the cluster workload activity can be signs of security events or other issues.

# 8 Backups
The restoration of backups is a process that is executed with root permission and therefore it can lead to potential security risks. Please ensure the backup files are coming from trusted sources before restoring them.

# 9 Message logs
The backup of the log messages may contain sensitive information. Therefore, it is recommended to save the automatic backups in an AWS EFS type volume and periodically send the backups to an AWS S3 Bucket with encryption both in transit and rest. More information can be found in [the Kubernetes User Guide](https://github.com/nordic-institute/X-Road-Security-Server-sidecar/blob/master/doc/kubernetes_security_server_sidecar_user_guide.md#8-message-logs-and-disk-space).