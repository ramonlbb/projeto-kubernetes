---

# Projeto de Deploy Fullstack com Kubernetes

## 1. Visão Geral do Projeto

Este projeto demonstra o deploy completo de uma aplicação web fullstack (React + Flask + PostgreSQL) em um cluster Kubernetes. O objetivo é aplicar conceitos essenciais de orquestração, como alta disponibilidade, gerenciamento de configurações, persistência de dados e exposição de serviços para o mundo externo através de um Ingress Controller.

### 1.1. Arquitetura da Solução

A aplicação é dividida em dois namespaces para melhor organização e segurança:

*   `app-ns`: Contém os componentes da aplicação (frontend e backend).
*   `db-ns`: Contém os componentes do banco de dados (PostgreSQL).

O fluxo de tráfego é gerenciado pelo **NGINX Ingress Controller**, que direciona as requisições externas para os serviços apropriados:
*   Requisições para a raiz (`/`) são direcionadas para o **Frontend (React)**.
*   Requisições para o caminho `/api/` são direcionadas para o **Backend (Flask)**.

![Arquitetura da Aplicação](https://raw.githubusercontent.com/pedrofilhojp/kube-students-projects/main/assets/image.png)

### 1.2. Integrantes da Equipe

Ramon Lucas e Mateus Eduardo

---

## 2. Pré-requisitos

Antes de começar, garanta que você tenha as seguintes ferramentas instaladas e configuradas:

1.  **Docker**: Para construir as imagens da aplicação.
2.  **kubectl**: A ferramenta de linha de comando para interagir com o cluster Kubernetes.
3.  **Kind**: Para criar um cluster Kubernetes localmente.
4.  **Docker Hub (Opcional)**: Uma conta para publicar as imagens Docker construídas (necessário se o deploy for em um cluster remoto).

---

## 3. Passos para Execução do Deploy

Siga os passos abaixo para configurar o ambiente e realizar o deploy da aplicação.

### 3.1. Construir e Publicar as Imagens Docker

Antes de aplicar os manifestos do Kubernetes, é necessário construir as imagens Docker para o frontend e o backend e publicá-las em um registro de contêineres, como o Docker Hub.

1.  Navegue até o diretório de cada aplicação (`frontend/` e `backend/`).
2.  Construa e publique as imagens. Substitua `seu-usuario-dockerhub` pelo seu nome de usuário.
    ```bash
    # Construindo a imagem do backend
    docker build -t seu-usuario-dockerhub/backend-flask:latest ./backend
    docker push seu-usuario-dockerhub/backend-flask:latest

    # Construindo a imagem do frontend
    docker build -t seu-usuario-dockerhub/frontend-react:latest ./frontend
    docker push seu-usuario-dockerhub/frontend-react:latest
    ```
3.  **Importante:** Após publicar as imagens, atualize os arquivos `backend/deployment.yaml` e `frontend/deployment.yaml` com os nomes corretos das suas imagens.

### 3.2. Configurar o Cluster Kubernetes com Kind

Para que o Ingress funcione corretamente em um ambiente local, o cluster Kind precisa ser criado com um mapeamento de portas.

1.  Crie um arquivo de configuração `kind-config.yaml`:
    ```yaml
    # kind-config.yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
    ```

2.  Crie o cluster Kind usando o arquivo de configuração:
    ```bash
    kind create cluster --name meu-projeto-k8s --config kind-config.yaml
    ```

### 3.3. Instalar o NGINX Ingress Controller

O cluster precisa de um Ingress Controller para gerenciar o acesso externo.
```bash
# Aplicar o manifesto de instalação do NGINX Ingress para Kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Aguardar o Ingress Controller ficar pronto antes de continuar
echo "Aguardando o Ingress Controller ficar pronto..."
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
echo "Ingress Controller pronto!"
```

### 3.4. Aplicar os Manifestos da Aplicação

Com o cluster e o Ingress prontos, aplique os manifestos do projeto na ordem correta para garantir que as dependências (como os Namespaces) sejam criadas primeiro.

```bash
# 1. Crie os Namespaces
echo "Criando os namespaces..."
kubectl apply -f namespace.yaml

# Dê um segundo para os namespaces serem registrados no cluster
sleep 2

# 2. Crie os recursos do Banco de Dados
echo "Aplicando recursos do banco de dados..."
kubectl apply -f database/

# 3. Crie os recursos do Backend
echo "Aplicando recursos do backend..."
kubectl apply -f backend/

# 4. Crie os recursos do Frontend
echo "Aplicando recursos do frontend..."
kubectl apply -f frontend/

# 5. Crie a regra de Ingress
echo "Aplicando a regra de Ingress..."
kubectl apply -f ingress/

echo "Deploy concluído!"
```

### 3.5. Configurar o DNS Local

Para acessar a aplicação usando um domínio amigável, adicione a seguinte linha ao seu arquivo de hosts:

*   **Linux/macOS:** `/etc/hosts`
*   **Windows:** `C:\Windows\System32\drivers\etc\hosts`

```
127.0.0.1   meu-app.com
```
*É necessário privilégios de administrador para editar este arquivo.*

---

## 4. Verificação e Acesso

Após a conclusão do deploy, você pode verificar o status dos recursos e acessar a aplicação.

### 4.1. Verificando o Status dos Pods

Para verificar se todos os componentes (pods) da aplicação e da infraestrutura estão rodando corretamente, use o seguinte comando:

```bash
kubectl get pods --all-namespaces
# ou, de forma mais organizada:
kubectl get pods -A
```

O resultado esperado é ver todos os pods nos namespaces `app-ns`, `db-ns` e `ingress-nginx` com o status `Running`.

### 4.2. Acessando a Aplicação

Abra seu navegador e acesse o seguinte endereço:

> **http://meu-app.com**

A interface da aplicação de mensagens deve ser carregada. Você pode adicionar novas mensagens para testar a comunicação completa entre o frontend, o backend e a persistência no banco de dados.

---
