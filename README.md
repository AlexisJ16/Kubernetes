# Services E Ingress en Minikube

Descripción: Implementación de 4 servicios con imagenes diferentes, usando Minikube

Estado: Listo

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

<img width="647" height="349" alt="image" src="https://github.com/user-attachments/assets/71211317-c638-4290-a431-a9e78c1b0b2f" />


## 3) Arranque limpio y add-ons

```bash
minikube start --driver=docker
minikube status
kubectl cluster-info

minikube addons enable metrics-server
minikube addons enable ingress
kubectl -n ingress-nginx get pods

```

<img width="791" height="603" alt="image 1" src="https://github.com/user-attachments/assets/107a93e2-ef1d-4a9e-a62c-9d7914783a63" />

<img width="588" height="71" alt="image 2" src="https://github.com/user-attachments/assets/fa8224eb-dc11-40a0-a797-d03fbaf89e49" />


## 4) Namespace

```bash
kubectl create ns lab
kubectl config set-context --current --namespace=lab
kubectl get ns

```

<img width="520" height="213" alt="image 3" src="https://github.com/user-attachments/assets/b64e11a1-f36d-445c-aaf7-1c33a4ad8269" />


## 5) Puertos locales reservados (evitar choques)

- ClusterIP → 18081
- NodePort → 18082 (NodePort real: 31080)
- Ingress → 18083
- LoadBalancer → 18084

```bash
lsof -i :18081 -i :18082 -i :18083 -i :18084 || true

```

<img width="761" height="39" alt="image 4" src="https://github.com/user-attachments/assets/931a3fbe-c420-4444-b902-e79a5facde67" />

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

<img width="925" height="402" alt="image 5" src="https://github.com/user-attachments/assets/a3450687-4418-439e-9206-3642523cb7aa" />


<img width="1096" height="314" alt="image 6" src="https://github.com/user-attachments/assets/79e3e829-779e-48ae-a07c-610498c4301d" />

## 8) Pruebas con puertos independientes

### 8.1 ClusterIP → 18081

```bash
kubectl port-forward -n lab svc/web-cip 18081:80 &
curl -I http://127.0.0.1:18081

```

En Codespaces, marca 18081 Public.

<img width="718" height="156" alt="image 7" src="https://github.com/user-attachments/assets/6df6d1a5-ebd9-4e4e-a321-8675d0073ead" />

<img width="1302" height="331" alt="image 8" src="https://github.com/user-attachments/assets/6e3209c4-1e8d-4730-931e-5b7698e112a7" />


### 8.2 NodePort → 31080 y 18082

```bash
kubectl get -n lab svc web-np
minikube ip
curl -I http://$(minikube ip):31080

kubectl port-forward -n lab svc/web-np 18082:80 &
curl -I http://127.0.0.1:18082

```

Marca 18082 Public.

<img width="672" height="431" alt="image 9" src="https://github.com/user-attachments/assets/6bc4c26e-9186-4585-8c72-5d27a52d9298" />

<img width="1301" height="167" alt="image 10" src="https://github.com/user-attachments/assets/1e1bfaba-338c-4aa5-9c60-5e9c505aea39" />

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

<img width="724" height="553" alt="image 11" src="https://github.com/user-attachments/assets/7b41356b-a716-404e-845e-1aba1a95072c" />

<img width="1919" height="1032" alt="image 12" src="https://github.com/user-attachments/assets/3714710e-755c-460d-8e04-376ebcd77ebc" />

### 8.4 Ingress → 18083

```bash
kubectl get -n lab ingress web-ing
kubectl -n ingress-nginx get svc ingress-nginx-controller

kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 18083:80 &
curl -I http://127.0.0.1:18083

```

Marca 18083 Public. Respuesta 200 de `nginxdemos/hello`.

<img width="752" height="280" alt="image 13" src="https://github.com/user-attachments/assets/11166de3-3ad2-4d97-92a4-d6754ebf0223" />

<img width="1271" height="210" alt="image 14" src="https://github.com/user-attachments/assets/7e3bbd08-3c7d-497c-a06b-30b8c3e28175" />

## 9) Verificación

```bash
kubectl get all -n lab -o wide
kubectl top pods -n lab
kubectl describe -n lab svc web-cip
kubectl describe -n lab svc web-np
kubectl describe -n lab svc web-lb
kubectl describe -n lab ingress web-ing

```

<img width="956" height="478" alt="image 15" src="https://github.com/user-attachments/assets/70d6d4f2-3791-4d8a-93af-563625afb48c" />

<img width="1121" height="729" alt="image 16" src="https://github.com/user-attachments/assets/be0da7b7-1ac0-4a04-bc2e-e89a0336eeb2" />

<img width="644" height="705" alt="image 17" src="https://github.com/user-attachments/assets/c3d916c0-6a9d-494b-a699-f893878ec3de" />

## 10) Gestión y cierre de puertos

```bash
lsof -i :18081 -i :18082 -i :18083 -i :18084 || true
pkill -f "port-forward.*18081" 2>/dev/null || true
pkill -f "port-forward.*18082" 2>/dev/null || true
pkill -f "port-forward.*18083" 2>/dev/null || true
pkill -f "port-forward.*18084" 2>/dev/null || true

```

<img width="787" height="327" alt="image 18" src="https://github.com/user-attachments/assets/fa4632b4-b47b-4327-80ac-319b8d9281f7" />


## 11) Limpieza

```bash
kubectl delete -f manifests/ingress.yaml
kubectl delete -f manifests/loadbalancer.yaml
kubectl delete -f manifests/nodeport.yaml
kubectl delete -f manifests/clusterip.yaml
kubectl delete ns lab
sudo pkill -f "minikube tunnel" 2>/dev/null || true

```

<img width="642" height="293" alt="image 19" src="https://github.com/user-attachments/assets/551c3e12-92a0-4dea-a4cc-741f69bcf3ea" />
