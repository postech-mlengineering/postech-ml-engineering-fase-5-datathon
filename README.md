# Repositório para o Datathon da Fase 5 da Pós-Graduação em Machine Learning Engineering da FIAP

Este repositório contém a solução desenvolvida para o Datathon da Fase 5, com foco na tomada de decisão automatizada para campanhas de marketing bancário. O projeto implementa um pipeline ponta a ponta com **Multi-Armed Bandit Contextual**, ingestão de variáveis via **Feature Store (Feast)**, rastreamento de ciclo de vida com **MLflow**, orquestração via **Docker Compose**, **API de inferência com FastAPI** e uma interface em **React**.

A base utilizada é a *Bank Marketing* (Moro et al., 2014). A variável `duration` foi deliberadamente removida do pipeline para mitigar vazamento de dados (*data leakage*), uma vez que sua medição só ocorre após o término do contato.

---

### Modelagem

#### Multi-Armed Bandit Contextual (Epsilon-Greedy)

Em substituição a abordagens estáticas ou contato indiscriminado com toda a base, implementou-se uma política de exploração e explotação (*epsilon-greedy*) com aprendizado incremental contínuo. 

* **Estimador Base:** `SGDClassifier` do *scikit-learn*, atualizado a cada nova observação via `partial_fit`.
* **Pipeline de Pré-processamento:** Composto por `StandardScaler` para variáveis numéricas e `OneHotEncoder` para variáveis categóricas.
* **Matriz de Recompensas (Payoff):**
  * **Aceite da oferta:** $+2,0$
  * **Recusa da oferta:** $-0,5$ (associado a custos de telefonia, tempo de operação e desgaste de relacionamento com o cliente)
  * **Não ofertar:** $0,0$ (impacto neutro)
* **Regra de Decisão:** O modelo calcula a probabilidade de aceite do cliente e aciona a recomendação de oferta exclusivamente quando o valor esperado do retorno for positivo.

#### Linha de Base (Baseline)

Utilizado como referência comparativa heurística, aplicando a política de ofertar estritamente para perfis historicamente mais propensos à conversão: clientes com conversões em campanhas anteriores, estudantes e aposentados.

#### Oráculo e Regret

Para mensurar a eficácia da política contextual frente ao cenário teórico ideal, foi implementado um oráculo com conhecimento a posteriori da decisão ótima para cada cliente. Essa comparação viabiliza o cálculo analítico do *regret* acumulado da estratégia ao longo do tempo.

#### Feature Store (Feast)

O pipeline de dados opera com o Feast para servir variáveis tanto no treino quanto na inferência:
* **Armazenamento Offline:** Arquivos Parquet locais (`data/`).
* **Armazenamento Online:** SQLite (`data/online_store.db`).
* **Versões de Feature Views:**
  * `bank_features` (v1): Contempla o conjunto integral de atributos.
  * `bank_features_v2` (v2): Exclui as variáveis temporais `month` e `day_of_week`. A API utiliza esta versão como padrão.

---

### Pré-requisitos

Certifique-se de possuir as seguintes ferramentas instaladas em seu ambiente:

* **Python:** Versão `>=3.11` e `<3.13`
* **Poetry:** Gerenciador de ambientes e dependências Python
* **Docker e Docker Compose:** Para execução dos serviços conteinerizados
* **Node.js:** Versão `18+` (para execução local do frontend)

---

### Instalação

Clone o repositório e configure as dependências com o Poetry:

```bash
git clone https://github.com/postech-mlengineering/postech-ml-engineering-fase-5-datathon.git
cd postech-ml-engineering-fase-5-datathon
poetry install
```

---

### Como Executar

#### 1. Treinamento e Simulação

Executa a leitura dos dados brutos, geração das bases analíticas, treino do modelo incremental e persistência dos experimentos no MLflow:

```bash
poetry run python main.py
```

O script realiza as seguintes etapas:
1. Leitura de `data/bank-additional-full.csv`.
2. Geração do arquivo `data/bank-additional-full.parquet` com campos de governança (`client_id` e `event_timestamp`).
3. Execução da simulação para as versões v1 e v2 das *feature views*, registrando métricas no MLflow.
4. Registro do modelo como `bank_marketing_model` com definição do alias `champion`.

#### 2. Configuração e Materialização da Feature Store

Registra as definições das entidades e carrega os dados no repositório online:

```bash
cd feature_repo
poetry run feast apply
poetry run feast materialize-incremental $(date -u +"%Y-%m-%dT%H:%M:%S")
cd ..
```

#### 3. Execução dos Serviços via Docker Compose

Inicia os containers da API de inferência, serving do MLflow e UI de monitoramento:

```bash
docker compose up --build
```

Serviços inicializados:

| Serviço | Porta | Descrição |
|---|---|---|
| `fastapi_app` | 8000 | API RESTful de inferência e recomendações |
| `mlflow_serve` | 5002 | Servidor de predição do MLflow (`bank_marketing_model@champion`) |
| `mlflow_ui` | 5000 | Interface visual do MLflow para consulta de runs, traces e métricas |

> **Nota:** Certifique-se de executar as etapas 1 e 2 antes do build do Docker, pois os artefatos de dados e repositório de features são copiados para a imagem.

---

### Interface Web (Frontend)

O projeto disponibiliza uma interface visual ("Motor de Campanha") construída em React, Vite e Tailwind CSS.

```bash
cd frontend
npm install
npm run dev
```

* O Vite está pré-configurado na porta `80`. Em ambientes Linux sem privilégios de root, execute em porta alternativa:
```bash
npm run dev -- --port 5173
```
* **Credenciais de acesso para demonstração:** Usuário `admin` | Senha `password123`.

---

### Endereços de Acesso

* **Frontend:** `http://localhost` (ou `http://localhost:5173`)
* **Documentação da API (Swagger):** `http://localhost:8000/docs`
* **MLflow UI:** `http://localhost:5000`

---

### Testando a API de Inferência

#### 1. Obtenção do Token de Acesso

```bash
curl -X POST http://localhost:8000/token \
  -d "username=admin&password=password123"
```

#### 2. Inferência por Cliente

Retorna a probabilidade de aceite, valor esperado e decisão de oferta para um cliente específico:

```bash
curl -X POST "http://localhost:8000/predict?client_id=10" \
  -H "Authorization: Bearer admin"
```

#### 3. Recomendações em Lote

Retorna os clientes recomendados, ordenados por valor esperado decrescente (`top_n: 0` processa todos):

```bash
curl -X POST http://localhost:8000/recommendations \
  -H "Authorization: Bearer admin" \
  -H "Content-Type: application/json" \
  -d '{"top_n": 20}'
```

> Cada chamada realizada à API registra um *run* no experimento `bank_marketing_bandit` do MLflow contendo latência, probabilidades e métricas operacionais.

---

### Arquitetura Alvo em Nuvem (AWS)

Para implantação em ambiente produtivo na AWS, adota-se a seguinte divisão de componentes e serviços gerenciados:

* **Computação e Orquestração:** As imagens Docker dos serviços (`fastapi_app`, `mlflow_serve` e `mlflow_ui`) armazenadas no **Amazon ECR** e executadas via **Amazon ECS** sob instâncias gerenciadas, com tráfego distribuído por um **Application Load Balancer (ALB)** com terminação HTTPS.
* **Frontend:** Build estático do React armazenado em bucket **Amazon S3** e distribuído globalmente via **Amazon CloudFront**.
* **Armazenamento de Metadados e Modelos:** Backend do MLflow migrado de SQLite para **Amazon RDS (PostgreSQL)**; artefatos de modelos e datasets Parquet persistidos no **Amazon S3**.
* **Feature Store Online:** Migração do online store local para **Amazon DynamoDB** (ou **Amazon ElastiCache / Redis**) para garantir baixa latência nas leituras online.
* **Segurança e Governança:** Credenciais e tokens centralizados no **AWS Secrets Manager**, com logs e métricas estruturadas no **Amazon CloudWatch**.
* **Rotinas Agendadas e CI/CD:** Atualizações periódicas do modelo (`main.py`) e materialização do Feast disparadas via **Amazon EventBridge** em tarefas ECS; pipeline de integração contínua implementado via **GitHub Actions**.

---

### Notebooks

O notebook com a análise exploratória dos dados e testes preliminares localiza-se em `notebooks/eda.ipynb`. Para executá-lo:

```bash
poetry run jupyter notebook
```

---

### Tecnologias

| Componente | Tecnologia | Versão | Descrição |
|---|---|---|---|
| Linguagem | Python | `>=3.11,<3.13` | Linguagem para desenvolvimento de scripts e modelos |
| Gerenciamento | Poetry | `>=2.0.0` | Gerenciador de dependências e ambientes virtuais |
| Manipulação de Dados | Polars | `>=1.0.0` | Biblioteca de alta performance para manipulação de dados |
| Manipulação de Dados | Pandas | `>=2.0.0` | Biblioteca para manipulação e análise de dados tabulares |
| Machine Learning | Scikit-learn | `>=1.3.0` | Biblioteca para pré-processamento e treinamento de ML |
| MLOps / Rastreamento | MLflow | `>=2.14.0` | Rastreamento de experimentos, métricas, artefatos e serving |
| Feature Store | Feast | `==0.38.0` | Armazenamento e disponibilização de features offline e online |
| API / Framework | FastAPI | `>=0.100.0` | Framework assíncrono para desenvolvimento da API |
| Servidor ASGI | Uvicorn | `>=0.23.0` | Servidor ASGI para execução da aplicação FastAPI |
| Segurança / Autenticação | Python-Jose | `>=3.3.0` | Biblioteca para implementação de tokens de autenticação |
| Segurança / Hashing | Passlib | `>=1.7.4` | Biblioteca para controle e verificação de senhas |
| Visualização | Matplotlib | `>=3.7.0` | Biblioteca para criação de gráficos estáticos |
| Monitoramento | psutil | `>=5.9.0` | Biblioteca para coleta de métricas de uso de hardware e sistema |
| Serialização | Protobuf | `==4.25.3` | Protocolo para serialização de dados estruturados |
| Frontend | React + Vite + Tailwind | `-` | Interface web para visualização e operação da campanha |
| Conteinerização | Docker / Compose | `-` | Padronização e orquestração de containers de serviço |

---

### Estrutura do Projeto

```
.
├── backend/
│   └── api/                # Ponto de entrada da API FastAPI e Dockerfile
├── data/                   # Arquivos CSV brutos, base Parquet e SQLite online
├── docs/                   # Enunciado do projeto e documentações de apoio
├── feature_repo/           # Definições das entidades e feature views do Feast
├── frontend/               # Código-fonte da aplicação React (Vite + Tailwind)
├── notebooks/              # Análise exploratória dos dados (eda.ipynb)
├── Dockerfile.mlflow       # Imagem Docker dedicada ao serving de modelos no MLflow
├── docker-compose.yml      # Definição e orquestração dos serviços
├── main.py                 # Pipeline de treino, simulação do bandit e tracking
├── pyproject.toml          # Configuração do Poetry e dependências do projeto
└── README.md
```

---

### Observações

* **Seleção de Feature Views:** Para alternar a versão utilizada pela API, configure a variável de ambiente `FEATURE_VIEW_NAME` (`bank_features` ou `bank_features_v2`) no container `fastapi_app`.
* **Persistência de Dados Locais:** Os arquivos de banco de dados SQLite (`.db`) estão incluídos no `.gitignore`. Por esse motivo, é mandatório executar as etapas de treino e materialização do Feast ao clonar o projeto em um novo ambiente.
* **Autenticação:** O mecanismo de autenticação presente na API possui fins exclusivamente demonstrativos, com credenciais estáticas de desenvolvimento.

---

### Referência da Base

S. Moro, P. Cortez e P. Rita. *A Data-Driven Approach to Predict the Success of Bank Telemarketing*. Decision Support Systems, 2014. http://dx.doi.org/10.1016/j.dss.2014.03.001

## Colaboradores

[Jorge Platero](https://github.com/jorgeplatero)

[Leandro Delisposti](https://github.com/LeandroDelisposti)

[Hugo Rodrigues](https://github.com/Nokard)
