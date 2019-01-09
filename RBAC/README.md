How to Provide access to kubernetes cluster.

openssl genrsa -out devuser.key 2048

openssl req -new -key devuser.key -out devuser.csr -subj "/CN=devuser/O=readonly"

copy ca.crt and ca.key from master to current location.

openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -days 730 -in devuser.csr -out devuser.crt

kubectl config set-cluster kubernetes --embed-certs=true --server=https://172.29.46.114:6443 --certificate-authority=ca.crt

kubectl config set-credentials devuser --client-certificate=/Users/vikash.singh3/staging_kubernetes_infra/kubernetes_yaml/users/devuser.crt --client-key=/Users/vikash.singh3/staging_kubernetes_infra/kubernetes_yaml/users/devuser.key

kubectl config set-context dev-kubernetes --cluster=kubernetes --user=devuser

kubectl config  use-context dev-kubernetes


cat >> read-only-role.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: readonly-role
  namespace: staging-ns
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
EOF

cat >> read-only-rolebinding.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-only-rb
  namespace: staging-ns
subjects:
- kind: Group
  name: readonly
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: readonly-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f read-only-role.yaml
kubectl apply -f read-only-rolebinding.yaml


