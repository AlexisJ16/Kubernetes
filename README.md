# Services E Ingress en Minikube

Cursos: Plataformas ll (https://www.notion.so/Plataformas-ll-257bc4ac62d6806f8641f7c80d2e2015?pvs=21)
Descripción: Implementación de 4 servicios con imagenes diferentes, usando Minikube
Estado: Listo
Hecho: Yes
Plazo: 29 de septiembre de 2025
Responsable: Alexis Jaramillo Martinez

---

# Laboratorio: Servicios e Ingress en Kubernetes con Minikube (Codespaces)

## 1) Fundamento mínimo

- ClusterIP: acceso interno L4 dentro del clúster.
- NodePort: expone L4 en cada nodo en 30000–32767.
- LoadBalancer: IP externa L4 (simulada con `minikube tunnel`).
- Ingress: enrutamiento L7 por host/path mediante NGINX Ingress Controller.

## 2) Reset duro

```bash
pkill -f "kubectl proxy" 2>/dev/null || true
sudo pkill -f "minikube tunnel" 2>/dev/null || true
pkill -f "port-forward" 2>/dev/null || true

minikube delete --all --purge

cp ~/.kube/config ~/.kube/config.bak 2>/dev/null || true
kubectl config delete-context minikube 2>/dev/null || true
kubectl config delete-cluster minikube 2>/dev/null || true
kubectl config unset users.minikube 2>/dev/null || true
kubectl config unset current-context 2>/dev/null || true

kubectl config get-contexts

```

![image.png](image.png)

## 3) Arranque limpio y add-ons

```bash
minikube start --driver=docker
minikube status
kubectl cluster-info

minikube addons enable metrics-server
minikube addons enable ingress
kubectl -n ingress-nginx get pods

```

![image.png](image%201.png)

![image.png](image%202.png)

## 4) Namespace

```bash
kubectl create ns lab
kubectl config set-context --current --namespace=lab
kubectl get ns

```

![image.png](image%203.png)

## 5) Puertos locales reservados (evitar choques)

- ClusterIP → 18081
- NodePort → 18082 (NodePort real: 31080)
- Ingress → 18083
- LoadBalancer → 18084

```bash
lsof -i :18081 -i :18082 -i :18083 -i :18084 || true

```

![image.png](image%204.png)

## 6) Manifiestos

### 6.1 `manifests/clusterip.yaml` (nginx, ClusterIP)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-cip
  namespace: lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-cip
  template:
    metadata:
      labels:
        app: web-cip
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-cip
  namespace: lab
spec:
  type: ClusterIP
  selector:
    app: web-cip
  ports:
  - port: 80
    targetPort: 80

```

### 6.2 `manifests/nodeport.yaml` (httpd, NodePort=31080)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-np
  namespace: lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-np
  template:
    metadata:
      labels:
        app: web-np
    spec:
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-np
  namespace: lab
spec:
  type: NodePort
  selector:
    app: web-np
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31080

```

### 6.3 `manifests/loadbalancer.yaml` (ealen/echo-server, LoadBalancer)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-lb
  namespace: lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-lb
  template:
    metadata:
      labels:
        app: web-lb
    spec:
      containers:
      - name: echo
        image: ealen/echo-server:latest
        env:
        - name: PORT
          value: "80"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: lab
spec:
  type: LoadBalancer
  selector:
    app: web-lb
  ports:
  - port: 80
    targetPort: 80

```

### 6.4 `manifests/ingress.yaml` (nginxdemos/hello, Ingress NGINX)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-ing
  namespace: lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-ing
  template:
    metadata:
      labels:
        app: web-ing
    spec:
      containers:
      - name: hello
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-ing
  namespace: lab
spec:
  type: ClusterIP
  selector:
    app: web-ing
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
  namespace: lab
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-ing
            port:
              number: 80

```

## 7) Despliegue

```bash
kubectl apply -f manifests/clusterip.yaml
kubectl apply -f manifests/nodeport.yaml
kubectl apply -f manifests/loadbalancer.yaml
kubectl apply -f manifests/ingress.yaml
kubectl get all -n lab -o wide

```

![image.png](image%205.png)

![image.png](image%206.png)

## 8) Pruebas con puertos independientes

### 8.1 ClusterIP → 18081

```bash
kubectl port-forward -n lab svc/web-cip 18081:80 &
curl -I http://127.0.0.1:18081

```

En Codespaces, marca 18081 Public.

![image.png](image%207.png)

![image.png](image%208.png)

### 8.2 NodePort → 31080 y 18082

```bash
kubectl get -n lab svc web-np
minikube ip
curl -I http://$(minikube ip):31080

kubectl port-forward -n lab svc/web-np 18082:80 &
curl -I http://127.0.0.1:18082

```

Marca 18082 Public.

![image.png](image%209.png)

![image.png](image%2010.png)

### 8.3 LoadBalancer → 18084 o túnel

```bash
kubectl get -n lab svc web-lb
# Opción A: EXTERNAL-IP con túnel
sudo -E minikube tunnel     # en otra terminal
kubectl get -n lab svc web-lb -w
export ELB=$(kubectl -n lab get svc web-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -I http://$ELB

# Opción B: port-forward estable
kubectl port-forward -n lab svc/web-lb 18084:80 &
curl -I http://127.0.0.1:18084

```

`ealen/echo-server` devuelve JSON del request. Esa salida es la evidencia.

![image.png](image%2011.png)

![image.png](image%2012.png)

### 8.4 Ingress → 18083

```bash
kubectl get -n lab ingress web-ing
kubectl -n ingress-nginx get svc ingress-nginx-controller

kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 18083:80 &
curl -I http://127.0.0.1:18083

```

Marca 18083 Public. Respuesta 200 de `nginxdemos/hello`.

![image.png](image%2013.png)

![image.png](image%2014.png)

## 9) Verificación

```bash
kubectl get all -n lab -o wide
kubectl top pods -n lab
kubectl describe -n lab svc web-cip
kubectl describe -n lab svc web-np
kubectl describe -n lab svc web-lb
kubectl describe -n lab ingress web-ing

```

![image.png](image%2015.png)

![image.png](image%2016.png)

![image.png](image%2017.png)

## 10) Gestión y cierre de puertos

```bash
lsof -i :18081 -i :18082 -i :18083 -i :18084 || true
pkill -f "port-forward.*18081" 2>/dev/null || true
pkill -f "port-forward.*18082" 2>/dev/null || true
pkill -f "port-forward.*18083" 2>/dev/null || true
pkill -f "port-forward.*18084" 2>/dev/null || true

```

![image.png](image%2018.png)

## 11) Limpieza

```bash
kubectl delete -f manifests/ingress.yaml
kubectl delete -f manifests/loadbalancer.yaml
kubectl delete -f manifests/nodeport.yaml
kubectl delete -f manifests/clusterip.yaml
kubectl delete ns lab
sudo pkill -f "minikube tunnel" 2>/dev/null || true

```

![image.png](image%2019.png)
