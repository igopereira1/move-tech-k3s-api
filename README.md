# ☸️ API REST na Magalu Cloud com K3s (Kubernetes) & CI/CD

![Kubernetes](https://img.shields.io/badge/Kubernetes_K3s-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Magalu Cloud](https://img.shields.io/badge/Magalu_Cloud-0086FF?style=for-the-badge&logo=icloud&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)

Este repositório guia o fluxo completo de publicação e orquestração de uma **API REST com FastAPI** em nuvem: do versionamento no GitHub, passando pelo build e push automatizado via **GitHub Actions** para o **Magalu Cloud Container Registry**, até o deploy em um cluster **K3s (Kubernetes)** exposto pelo roteador **Traefik Ingress**.

---

## 📌 Visão Geral da Arquitetura

A esteira de entrega foi estruturada para separar claramente as responsabilidades de **Build (CI)** e **Deploy/Orquestração (CD/Kubernetes)**:

```text
[ Git Push / Fork ] ──> [ GitHub Actions ] ──( Build & Push )──> [ Magalu Cloud Registry ]
                                                                             │
                                                                             │ (Image Pull)
                                                                             ▼
[ Usuário / Client ] ──( HTTP :80 )──> [ Traefik Ingress ] ──> [ Service ClusterIP ]
                                                                        │
                                                               ┌────────┴────────┐
                                                               ▼                 ▼
                                                          [ Pod 1 ]         [ Pod 2 ]
                                                      (FastAPI :8000)   (FastAPI :8000)
```

---

## 🛠️ Tecnologias Utilizadas

- **Linguagem & Framework:** Python 3.12 + FastAPI (Uvicorn)
- **Containerização:** Docker
- **CI/CD:** GitHub Actions
- **Registry Privado:** Magalu Cloud Container Registry
- **Orquestração Cloud:** K3s (Lightweight Kubernetes Single-Node VM na Magalu Cloud)
- **Exposição Externa:** Traefik Ingress Controller + Kubernetes ClusterIP Service

---

## 📂 Estrutura do Projeto

```text
move_api/
├── .github/
│   └── workflows/
│       └── deploy.yml            # Pipeline do GitHub Actions (CI)
├── app/                          # Código fonte da API FastAPI
├── k8s/
│   ├── namespace.yml             # Isolamento de ambiente no K8s (move-tech-api)
│   ├── deployment.yml            # Réplicas, Probes, Resources e Secrets
│   ├── service.yml               # Comunicação interna ClusterIP (Porta 80 -> 8000)
│   └── ingress.yml               # Exposição HTTP pública via Traefik (:80)
├── Dockerfile                    # Instruções de empacotamento da API
├── docker-compose.yml            # Ambiente de desenvolvimento local
├── pyproject.toml                # Gerenciamento de dependências
└── README.md                     # Documentação do laboratório
```

---

## 🔒 Configuração de Segredos e Variáveis (GitHub)

Para que o GitHub Actions consiga autenticar e publicar no Container Registry da Magalu Cloud, configure em **Settings > Secrets and variables > Actions**:

### 🗝️ Repository Secrets

| Nome do Secret | Descrição |
| :--- | :--- |
| `MAGALU_REGISTRY_USERNAME` | Usuário retornado por `mgc container-registry credentials get` |
| `MAGALU_REGISTRY_PASSWORD` | Senha retornada por `mgc container-registry credentials get` |

---

## ⚡ Guia Rápido de Execução das Missões (Passo a Passo)

### 1️⃣ Desenvolvimento & Validação Local
```bash
# Build da imagem local
docker build -t move-tech-orders-api:local .

# Execução em background
docker run -d --rm -p 8000:8000 --name api-local move-tech-orders-api:local

# Teste de saúde
curl -i http://127.0.0.1:8000/health
```
> Acesse no navegador: `http://127.0.0.1:8000/docs`

---

### 2️⃣ Publicação no GitHub Actions (CI)
Crie a branch de trabalho e envie a alteração para rodar a esteira automatizada:
```bash
git switch -c feature/container_registry
git add .
git commit -m "ci: adiciona pipeline de build e publicação"
git push -u origin feature/container_registry
```

---

### 3️⃣ Preparando o Cluster Kubernetes (K3s + Magalu Cloud)
Com a MGC CLI autenticada e o script `k3s.sh` disponível:
```bash
# Criar cluster single-node na VM Magalu Cloud
./k3s.sh kubernetes cluster create --name move-tech-api

# Criar Namespace no Kubernetes
kubectl apply -f k8s/namespace.yml

# Criar Credenciais de Acesso ao Registry Privado no Namespace correto
kubectl create secret docker-registry mgc-registry-secret \
  --namespace move-tech-api \
  --docker-server=container-registry.br-se1.magalu.cloud \
  --docker-username='SEU_USUARIO_REGISTRY' \
  --docker-password='SUA_SENHA_REGISTRY'
```

---

### 4️⃣ Deploy dos Manifestos K8s no Cluster
```bash
# 1. Aplicação dos Pods (Deployment)
kubectl apply -f k8s/deployment.yml

# 2. Exposição Interna (Service ClusterIP)
kubectl apply -f k8s/service.yml

# 3. Exposição Externa HTTP (Ingress Traefik)
kubectl apply -f k8s/ingress.yml

# Verificar status da implantação
kubectl rollout status deployment/move-tech-orders-api -n move-tech-api
```

---

## 🧪 Validação e Consumo da API na Nuvem

Obtenha o IP público da sua VM Magalu Cloud e defina no seu terminal:
```bash
export API_IP="IP_PUBLICO_DA_VM"

# Validar Healthcheck
curl -i "http://${API_IP}/health"

# Teste de Endpoint GET
curl -i "http://${API_IP}/items"
```

📍 **Endpoints de Produção:**
- **Documentação Interativa (Swagger):** `http://IP_PUBLICO_DA_VM/docs`
- **Health Check Probe:** `http://IP_PUBLICO_DA_VM/health`

---

## 🛠️ Diagnóstico Rápido de Falhas (Troubleshooting)

| Sintoma / Erro | Causa Provável | Solução |
| :--- | :--- | :--- |
| `ErrImagePull` / `ImagePullBackOff` | Secret ausente ou em namespace diferente | Verifique `kubectl get secret -n move-tech-api` |
| `CrashLoopBackOff` | Health probe apontando para caminho errado | Confirme se as probes no YAML usam `/health` na porta `8000` |
| Endpoints do Service `<none>` | Selector do Service não bate com os labels dos Pods | Garanta que ambos usam `app: move-tech-orders-api` |
| `FailedScheduling` (Port 80) | Tentativa de usar Service LoadBalancer | O Traefik já usa a porta 80. Use `ClusterIP` + `Ingress` |
| Navegador não abre a página | Navegador forçando HTTPS automático | Acesse explicitamente com `http://` (sem TLS configurado) |

---

## 🧹 Limpeza de Recursos

Para evitar custos desnecessários após o término do laboratório:
```bash
# Remover os recursos do Kubernetes
kubectl delete namespace move-tech-api

# Destruir a VM e Cluster na Magalu Cloud
./k3s.sh kubernetes cluster delete --cluster-id ID_DO_CLUSTER
```

---

## 👨‍💻 Autor

Desenvolvido como parte do laboratório prático de **Publicação e Consumo de API REST na Magalu Cloud com K3s (Kubernetes)**.