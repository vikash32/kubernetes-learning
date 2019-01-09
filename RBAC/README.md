# Provide user access to kubernetes cluster. 

We are going to use authentication/authorization through certificate using openssl, We can create certificate for user specific and assign that user to specific group. 
For example: Use "CN" as user name and "O" as group. So, in this case username is "devuser" and the group is "readonly".

These are the steps we are going to execute to provide a user say "devuser" of access get/list/watch of all the resources.
### Step1. : Create Private Key "devuser.key" and then create certifiate signing request file "devuser.csr"
```
openssl genrsa -out devuser.key 2048

openssl req -new -key devuser.key -out devuser.csr -subj "/CN=devuser/O=readonly"
```
Now we have to signed the devuser.csr file through our certificate Authority generated on api server. You can get path of ca.crt/ca.key as given.
`/etc/kubernetes/pki (kubeadm)`
`/etc/kubernetes/ssl (kubespray)`

```
Certificate : ca.crt (kubeadm) or ca.key (kubespray)
Pricate Key : ca.key (kubeadm) or ca-key.pem (kubespray)
```
You can copy ca.key,ca.crt,devuser.csr,devuser.key at a common place on your local system so that you can generate a signed certificate.

### Step2. : Signed the "devuser.csr" file with ca.crt and ca.key file and create "devuser.crt" certificate file. 

```
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -days 730 -in devuser.csr -out devuser.crt
```
Now we have devuser.crt (Certificate file).  We are going to create kubernetes config file.

### Step3. Create kubernetes config file. Execute these commands on local system. But first make sure your local system has kubectl installed.
```
kubectl config set-cluster kubernetes --embed-certs=true --server=https://<api-server-ip>:6443 --certificate-authority=ca.crt

kubectl config set-credentials devuser --client-certificate=devuser.crt --client-key=devuser.key

kubectl config set-context dev-kubernetes --cluster=kubernetes --user=devuser

kubectl config  use-context dev-kubernetes
```

We are done with creation and setting up of user "devuser" to use kubernetes cluster. Lastly we need to create cluster role to have "get/list/watch" access on across all namespace. Also Cluster Role Binding to map the role with the "readonly" group.

Please note, we are using ClusterRole and ClusterRoleBinding to get/list/watch across all namespace. If you want to make for specific namespace, you can create Role and RoleBinding.

### Step4. Create Role/ClusterRole to specify the resources which you want to get accessible by this user or group. Execute these command from apiserver/master node.
```
cat >> read-only-role.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
EOF
```
### Step5. Create RoleBinding/ClusterRoleBinding to map the Role/ClusterRole with the user/group.Execute these command from apiserver/master node.
```
cat >> read-only-rolebinding.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only-rb
subjects:
- kind: Group
  name: readonly
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: readonly-role
  apiGroup: rbac.authorization.k8s.io
EOF
```
Execute below manifest file from api servers.
```
kubectl apply -f read-only-role.yaml
kubectl apply -f read-only-rolebinding.yaml
```

Now if from your workstation, we go for listing pods,svc,deployment. it will show us the list of them. But if we try to create deployment. it will throw forbidden.

### Verify from your local system
```
sh-3.2# kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-65v85                1/1     Running   0          3d
kube-system   coredns-78fcdf6894-s5tj2                1/1     Running   0          3d
kube-system   etcd-etinsz2ap4864                      1/1     Running   0          3d
kube-system   kube-apiserver-etinsz2ap4864            1/1     Running   0          3d
kube-system   kube-controller-manager-etinsz2ap4864   1/1     Running   0          3d
kube-system   kube-proxy-bdmlf                        1/1     Running   0          3d
kube-system   kube-proxy-m6qqp                        1/1     Running   0          3d
kube-system   kube-scheduler-etinsz2ap4864            1/1     Running   0          3d
kube-system   weave-net-njllh                         2/2     Running   0          3d
kube-system   weave-net-wvd4p                         2/2     Running   0          3d
sh-3.2#
sh-3.2#
sh-3.2# kubectl create deployment nginx --image=nginx
Error from server (Forbidden): deployments.apps is forbidden: User "devuser" cannot create deployments.apps in the namespace "default"
```

For further reading : https://kubernetes.io/docs/reference/access-authn-authz/rbac/

Thanks - Vikash

