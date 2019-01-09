#How to Provide access to kubernetes cluster. 
We are going to use authentication through certificates. use "CN" as user name and "O" as group, hence in this case username is "devuser" and the group is "readonly"

```
openssl genrsa -out devuser.key 2048

openssl req -new -key devuser.key -out devuser.csr -subj "/CN=devuser/O=readonly"
```

Now we have to signed the devuser.csr file through our certificate Authority generated on api server. You can get the path of it. 
```
Certificate : ca.crt (kubeadm) or ca.key (kubespray)
Pricate Key : ca.key (kubeadm) or ca-key.pem (kubespray)
```
You can copy the certificate from master to workspace then signed devuser signing request. 
```
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -days 730 -in devuser.csr -out devuser.crt
```
Now we have devuser.crt (Certificate file).  We are going to create kubernetes config file.
```
kubectl config set-cluster kubernetes --embed-certs=true --server=https://<api-server-ip>:6443 --certificate-authority=ca.crt

kubectl config set-credentials devuser --client-certificate=devuser.crt --client-key=devuser.key

kubectl config set-context dev-kubernetes --cluster=kubernetes --user=devuser

kubectl config  use-context dev-kubernetes
```

We are done with creation and setting up of user "devuser" to use kubernetes cluster. Lastly we need to create cluster role to have "get/list/watch" access on across all namespace. Also Cluster Role Binding to map the role with the "readonly" group.

Please note, we are using ClusterRole and ClusterRoleBinding to get/list/watch across all namespace. If you want to make for specific namespace, you can create Role and RoleBinding.

To create ClusterRole , excute below command from api server 
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

To create ClusterRoleBinding , execute below command from api servers.
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



