# kubernetes-learning
When i started working on Kubernetes almost 2.5 years back, Ingress was one of the nightmare for me (Yes, I had a list of nightmare like RBAC, ServiceAccount etc). Eventually i learned it, And i wanted to make a brief default full fledge working ingress introduction here.

kubectl (1.11.4), ingress-nginx (https://github.com/kubernetes/ingress-nginx)

For those who do first then learn. You can create ingress-nginx by executing below 5 commands on a running cluster. And verify it and skip the whole article. Or you can simply create every manifest file and make it your own by going through it.
```
To Create:
kubectl apply -f https://raw.githubusercontent.com/vikash32/kubernetes-learning/master/IngressSetUp/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/vikash32/kubernetes-learning/master/IngressSetUp/service-nodeport.yaml
kubectl create deploy httpd --image=httpd ; kubectl expose deployment httpd --port=80 --target-port=80
kubectl create deploy nginx --image=nginx ; kubectl expose deployment nginx --port=80 --target-port=80
kubectl apply -f https://raw.githubusercontent.com/vikash32/kubernetes-learning/master/IngressSetUp/ingress-rules.yaml

To Verify:
curl -k https://<apiserver-ip>:<ingress-nginx-svc-443-nodeport>/   ### return 404
curl -k https://<apiserver-ip>:<ingress-nginx-svc-443-nodeport>/nginx  ### return nginx container home page
curl -k https://<apiserver-ip>:<ingress-nginx-svc-443-nodeport>/httpd  ### return httpd container home page
```
Note: Basic knowledge of Kubernetes required to perform this task.

Below the steps we will execute one by one.
1. Create Namespace "ingress-nginx" 
2. Create ConfigMap "nginx-configuration"
3. Create ServiceAccout "nginx-ingress-serviceaccount"
4. Create ClusterRole "nginx-ingress-clusterrole"
5. Create ClusterRoleBinding "nginx-ingress-clusterrole-nisa-binding"
6. Create Ingress Controller Deployment "nginx-ingress-controller"
7. Create Ingress Nginx Service "ingress-nginx"
8. Create deployment httpd/nginx to verify nginx-controller if it is working fine 
9. Create Ingress Rules 

Step1 : Create Namespace "ingress-nginx"
```
### ing-nginx-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```    
Step2 :  Create ConfigMap "nginx-configuration"
```
### ing-nginx-cm-config.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```    
There are two ways one can configure the nginx. Either using annotation or using ConfigMap. I am using ConfigMap here. For More Info

Step3 : Create ServiceAccout "nginx-ingress-serviceaccount"
```
### ing-nginx-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```    
Step4 : Create ClusterRole "nginx-ingress-clusterrole"
```
### ing-nginx-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
- apiGroups: [""]
  resources: [configmaps,endpoints,nodes,pods,secrets]
  verbs: [list,watch]

- apiGroups: [""]
  resources: [nodes]
  verbs: [get]

- apiGroups: [""]
  resources: [services]
  verbs: [get,list,watch]

- apiGroups: ["extensions"]
  resources: [ingresses]
  verbs: [get,list,watch]

- apiGroups: [""]
  resources: [events]
  verbs: [create,patch]

- apiGroups: ["extensions"]
  resources: [ingresses/status]
  verbs: [update]

- apiGroups: [""]
  resources: [configmaps]
  verbs: [create]

- apiGroups: [""]
  resources: [configmaps]
  resourceNames: [ingress-controller-leader-nginx]
```  
Step5 : Create ClusterRoleBinding "nginx-ingress-clusterrole-binding"
```
### ing-nginx-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
```
So far, we have create service account, cluster role and mapped the service account with cluster role through cluster role binding. For More Info

Step6 : Create Ingress Controller Deployment "nginx-ingress-controller"
```
### ing-nginx-controller-deploy.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
```              
The only thing which one need to understand the argument part in deployment. first argument tell to load the config from nginx-configuration configmap which we have created in step2, second arguments is the service ingress-nginx (We will create it in step 7) for the nginx-ingress-controller deployment which expose the endpoint of nginx controller. 3rd argument is the annotation which kubernetes provide by default. We will be using this annotation when we create ingress rules in step 8. For More Info 

Step7 : Create Ingress Nginx Service "ingress-nginx"
```
### ing-nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```    
Now we are all set with our nginx ingress controller.

Step8 : Create deployment and service httpd/nginx to test our routing on these httpd and nginx containers from ingress-nginx-controller.
```
kubectl create deploy httpd --image=httpd 
kubectl expose deployment httpd --port=80 --target-port=80
kubectl create deploy nginx --image=nginx
kubectl expose deployment nginx --port=80 --target-port=80
```
Step9 : Create Ingress Rules
```
### ing-nginx-rules.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /httpd
        backend: 
          serviceName: httpd
          servicePort: 80
      - path: /nginx
        backend:
          serviceName: nginx
          servicePort: 80
```          
Ingress rules help to route traffic on the basis of path. These are bascally syntax otherwise it is self explanatory that /nginx will go to nginx service which is pointing to nginx deployment and /httpd will go to httpd service which is pointing to httpd deployment. Here my api server IP is 172.24.48.64 and my ingress-nginx service (Step 7 ) node port for ssl(443) is 32718. And i am going to use it.
```
curl -k https://172.24.48.64:32718  
    <html>
    <head><title>404 Not Found</title></head>
    <body>
    <center><h1>404 Not Found</h1></center>
    <hr><center>nginx/1.15.6</center>
    </body>
    </html>



curl -k https://172.24.48.64:32718/httpd
    <html><body><h1>It works!</h1></body></html>



curl -k https://172.24.48.64:32718/nginx
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
```    
This Document is already long enough to go to sleep mode. But trust me once you get it, it will be matter of minute to set up nginx ingress. I highly recommend to go through the official documentation.

Ingress nginx configuration guide (https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/)

Kubernetes RBAC (https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

Kubernetes Ingress (https://kubernetes.io/docs/concepts/services-networking/ingress/)

Thanks - Vikash
