# DevOps Essentials Lab — 523

Projeto/laboratório desenvolvido durante o curso **DevOps Essentials (523)** da [4Linux](https://4linux.com.br).

Ambiente completo de **GitOps 100% local**, simulando na prática um fluxo de entrega contínua com Kubernetes, deploy automático via Git e infraestrutura como código — tudo rodando localmente, sem precisar de conta em cloud.

---

## O que este projeto demonstra

- **Cluster Kubernetes local** provisionado com **Kind**
- **Servidor Git self-hosted** com **Gitea**
- **Deploy contínuo (GitOps)** orquestrado por **ArgoCD**, incluindo self-heal e rollback automático
- **Aplicação web** em **Flask**, containerizada com **Docker** e publicada no cluster
- **Infraestrutura como código** com **OpenTofu** (fork open source do Terraform)
- **Emulação de serviços AWS** (S3, SQS, DynamoDB, Lambda) via **Floci**, sem custos de cloud
- Automação completa do ambiente com **Vagrant** e scripts shell (`setup.sh` / `teardown.sh`)

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                    VM Vagrant                           │
│                 (192.168.56.10)                         │
│                                                         │
│  Browser / Terminal (do host)                           │
│  192.168.56.10:3000  →  Gitea  (Git server)             │
│  192.168.56.10:8080  →  ArgoCD (GitOps operator)        │
│  192.168.56.10:9090  →  Aplicação Flask                 │
│  192.168.56.10:4566  →  Floci  (AWS local)              │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Cluster Kind (Docker)               │   │
│  │                                                  │   │
│  │  namespace: gitea                                │   │
│  │  └─ gitea-http (Git server via Helm)             │   │
│  │                                                  │   │
│  │  namespace: argocd                               │   │
│  │  └─ argocd-server                                │   │
│  │     └─ observa gitea/devops-lab.git              │   │
│  │        └─ aplica k8s/ no cluster                 │   │
│  │                                                  │   │
│  │  namespace: devops-lab                           │   │
│  │  └─ devops-app (Flask — DevOpsLab HelloWorld)    │   │
│  │                                                  │   │
│  │  namespace: tools                                │   │
│  │  └─ floci (S3, SQS, DynamoDB, Lambda...)         │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Fluxo GitOps implementado

```
git push → Gitea → ArgoCD detecta (~3 min) → kubectl apply → app atualizada
```

Rollback é feito com `git revert` — o cluster nunca é modificado diretamente, apenas via Git.

---

## Stack utilizada

| Ferramenta | Papel no projeto |
|---|---|
| **Kind** | Cluster Kubernetes local dentro do Docker |
| **Gitea** | Servidor Git self-hosted |
| **ArgoCD** | Operador GitOps — sincroniza o cluster automaticamente com o Git |
| **Floci** | Emulador local de serviços AWS (S3, SQS, DynamoDB, Lambda) |
| **OpenTofu** | Provisionamento de infraestrutura como código |
| **Flask** | Aplicação web de demonstração |
| **Docker** | Containerização da aplicação |
| **Helm** | Gerenciador de pacotes Kubernetes |
| **Vagrant** | Provisionamento automatizado do ambiente completo |

---

## Estrutura do repositório

```
523-devops-essentials-lab/
│
├── setup.sh          # Sobe cluster Kind + instala Gitea, ArgoCD, Floci
├── teardown.sh       # Destrói o cluster completamente
├── lab.sh            # Helpers de automação
│
├── app/              # Aplicação Flask (DevOpsLab HelloWorld)
│   ├── app.py        # Rotas: / e /health
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── static/       # CSS e imagens
│   └── templates/    # HTML (index.html)
│
├── k8s/              # Manifests Kubernetes (gerenciados pelo ArgoCD)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
│
├── argocd/
│   └── application.yaml   # ArgoCD Application apontando para o Gitea
│
└── aws-lab/               # Infraestrutura AWS emulada
    ├── test_aws.py        # Testes com boto3
    └── terraform/
        ├── main.tf        # Provider AWS → Floci (SQS via OpenTofu)
        ├── variables.tf
        └── outputs.tf
```

---

## Como rodar

### Pré-requisitos

| Ferramenta | Versão mínima | Instalação |
|---|---|---|
| Docker | 24+ | [docs.docker.com](https://docs.docker.com/engine/install/) |
| kind | 0.27+ | [kind.sigs.k8s.io](https://kind.sigs.k8s.io) |
| kubectl | 1.32+ | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools/) |
| helm | 3.16+ | [helm.sh](https://helm.sh/docs/intro/install/) |

**Hardware recomendado:** 8 GB RAM, 4 CPUs, 20 GB de disco livre.

### Com Vagrant (ambiente completo automatizado)

```bash
vagrant up
vagrant ssh
cd ~/523
bash setup.sh
```

Acesse pelo **host** (sua máquina física):

| Serviço | Endereço | Credenciais |
|---|---|---|
| **Gitea** | http://192.168.56.10:3000 | `gitadmin` / `gitadmin123` |
| **ArgoCD** | http://192.168.56.10:8080 | `admin` / (ver abaixo) |
| **Aplicação** | http://192.168.56.10:9090 | — |
| **Floci** | http://192.168.56.10:4566 | `test` / `test` |

### Sem Vagrant (máquina com Docker + ferramentas já instaladas)

```bash
git clone https://github.com/SEU-USUARIO/SEU-REPO.git
cd SEU-REPO
bash setup.sh
```

| Serviço | Endereço | Credenciais |
|---|---|---|
| **Gitea** | http://localhost:3000 | `gitadmin` / `gitadmin123` |
| **ArgoCD** | http://localhost:8080 | `admin` / (ver abaixo) |
| **Aplicação** | http://localhost:9090 | — |
| **Floci** | http://localhost:4566 | `test` / `test` |

Senha inicial do ArgoCD:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Para destruir o ambiente:

```bash
bash teardown.sh
```

---

## Cenários demonstrados

Alguns fluxos que o projeto implementa e que podem ser reproduzidos:

**Deploy contínuo via Git** — qualquer alteração commitada e enviada ao Gitea é detectada pelo ArgoCD e aplicada automaticamente no cluster, sem intervenção manual.

```bash
docker build -t devops-app:v2 app/
kind load docker-image devops-app:v2 --name devops-lab
sed -i 's|devops-app:.*|devops-app:v2|' k8s/deployment.yaml
git add . && git commit -m "feat: nova versão da aplicação" && git push
```

**Escalabilidade declarativa** — número de réplicas controlado via manifest e Git.

```bash
sed -i 's/replicas: 1/replicas: 3/' k8s/deployment.yaml
git add k8s/deployment.yaml && git commit -m "scale: aumenta réplicas" && git push
```

**Rollback via Git** — reversão de estado sem tocar diretamente no cluster.

```bash
git revert HEAD --no-edit && git push
```

**Self-heal do ArgoCD** — o Git é sempre a fonte da verdade; mudanças manuais no cluster são revertidas automaticamente.

```bash
kubectl scale deployment devops-app -n devops-lab --replicas=5
# ArgoCD reverte para o estado definido no Git em ~3 minutos
```

**Infraestrutura como código com OpenTofu** — criação de fila SQS emulada via Floci.

```bash
cd aws-lab/terraform
tofu init && tofu apply -auto-approve
```

---

## Troubleshooting

### Pod em Pending ou ImagePullBackOff

```bash
kubectl describe pod -n devops-lab <nome-do-pod>
kind load docker-image devops-app:latest --name devops-lab
```

### ArgoCD não conecta no Gitea

```bash
kubectl exec -n argocd deploy/argocd-server -- \
  curl -sf http://gitea-http.gitea.svc.cluster.local:3000
kubectl get pods -n gitea
```

### Floci não responde

```bash
kubectl get pods -n tools
kubectl logs -n tools deploy/floci
curl http://localhost:4566/_localstack/health   # de dentro da VM
```

### Porta já em uso (sem Vagrant)

```bash
lsof -i :3000
lsof -i :8080
bash teardown.sh
```

### ArgoCD não sincroniza automaticamente

```bash
kubectl -n argocd patch app devops-app \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

---

## Créditos

Projeto desenvolvido a partir do laboratório do curso **DevOps Essentials (523)** da [4Linux](https://4linux.com.br).
