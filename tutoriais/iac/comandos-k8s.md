O **kubectl** é a ferramenta de linha de comando usada para interagir com um cluster do Kubernetes.  
Abaixo estão os principais comandos e o que cada um faz:

---

## 🔎 Comandos de Consulta (Visualização)

### `kubectl get`

Lista recursos do cluster.

```bash
kubectl get pods
kubectl get services
kubectl get nodes
kubectl get deployments
```

Mostra informações resumidas dos recursos.

---

### `kubectl describe`

Mostra detalhes completos de um recurso específico.

```bash
kubectl describe pod nome-do-pod
```

Exibe eventos, status, IP, erros, etc.

---

### `kubectl logs`

Mostra os logs de um container dentro de um Pod.

```bash
kubectl logs nome-do-pod
```

Para múltiplos containers:

```bash
kubectl logs nome-do-pod -c nome-do-container
```

---

### `kubectl top`

Mostra uso de CPU e memória (requer Metrics Server).

```bash
kubectl top pods
kubectl top nodes
```

---

## 🚀 Comandos de Criação e Atualização

### `kubectl apply`

Cria ou atualiza recursos a partir de um arquivo YAML/JSON.

```bash
kubectl apply -f deployment.yaml
```

É o comando mais usado para deploy.

---

### `kubectl create`

Cria um recurso.

```bash
kubectl create -f pod.yaml
```

Também pode criar recursos rapidamente:

```bash
kubectl create deployment nginx --image=nginx
```

---

### `kubectl edit`

Abre o recurso no editor para edição direta.

```bash
kubectl edit deployment nginx
```

---

### `kubectl scale`

Altera o número de réplicas.

```bash
kubectl scale deployment nginx --replicas=3
```

---

### `kubectl autoscale`

Cria um autoscaler (HPA).

```bash
kubectl autoscale deployment nginx --min=2 --max=5 --cpu-percent=80
```

---

## ❌ Comandos de Remoção

### `kubectl delete`

Remove recursos.

```bash
kubectl delete pod nome-do-pod
kubectl delete -f deployment.yaml
```

---

## 🛠️ Comandos de Execução e Debug

### `kubectl exec`

Executa comando dentro de um container.

```bash
kubectl exec -it nome-do-pod -- /bin/bash
```

---

### `kubectl port-forward`

Encaminha uma porta local para um Pod ou Service.

```bash
kubectl port-forward pod/nome-do-pod 8080:80
```

---

### `kubectl cp`

Copia arquivos entre máquina local e Pod.

```bash
kubectl cp arquivo.txt nome-do-pod:/tmp/
```

---

## ⚙️ Comandos de Configuração

### `kubectl config`

Gerencia contexto e configurações.

```bash
kubectl config get-contexts
kubectl config use-context nome-contexto
```

---

### `kubectl cluster-info`

Mostra informações do cluster.

```bash
kubectl cluster-info
```

---

### `kubectl version`

Mostra versão do cliente e do servidor.

```bash
kubectl version
```

---

## 📦 Trabalhando com Namespaces

```bash
kubectl get pods -n nome-do-namespace
kubectl create namespace dev
kubectl config set-context --current --namespace=dev
```

---