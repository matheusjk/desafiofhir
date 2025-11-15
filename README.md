# desafiofhir
HL7 FHIR


Claro. Este é um exemplo de documentação (o entregável da Parte 1) que você pode salvar como `README.md` na raiz do seu projeto.

-----

# Projeto: Servidor FHIR Local e Pipeline de Dados

Este documento descreve o processo de instalação, configuração e uso de um servidor FHIR local (HAPI FHIR) utilizando Docker Compose, bem como as instruções para a execução de um pipeline de carga de dados (ETL) para popular o servidor com dados de pacientes.

## Parte 1: Configuração do Servidor FHIR (Arquitetura)

O objetivo desta parte é implantar um servidor FHIR Open Source (HAPI FHIR) em um ambiente local, sem depender de provedores de nuvem. A solução utiliza Docker e Docker Compose para garantir que o ambiente seja portátil, replicável e fácil de gerenciar.

### 1.1. Tecnologias e Arquitetura

  * **Docker & Docker Compose:** Utilizados para orquestrar os containers.
  * **HAPI FHIR:** Imagem `hapiproject/hapi-fhir-jpaserver-starter:latest`. Este é o nosso servidor de aplicação FHIR.
  * **PostgreSQL:** Imagem `postgres:14`. Este é o banco de dados de persistência para o HAPI FHIR.

O arquivo `docker-compose.yaml` define três componentes principais:

1.  **`fhir-db` (Serviço de Banco de Dados):**

      * Container PostgreSQL que armazena todos os recursos FHIR.
      * Utiliza um volume nomeado (`postgres_data`) para persistir os dados mesmo se o container for recriado.
      * Possui um `healthcheck` para garantir que o servidor HAPI só inicie após o banco estar pronto.

2.  **`hapi-fhir` (Serviço do Servidor FHIR):**

      * O servidor HAPI FHIR.
      * Expõe a porta `8080` do container na porta `8080` do host local.
      * É configurado via variáveis de ambiente para se conectar ao serviço `fhir-db` (ex: `spring.datasource.url=jdbc:postgresql://fhir-db:5432/hapi`).
      * A versão do FHIR é definida como `R4` e a segurança está desabilitada (`fhir.rest.security.enabled=false`) para facilitar o desenvolvimento local.

3.  **`etl-service` (Serviço de Carga de Dados):**

      * Um container de serviço (baseado em `jupyter/pyspark-notebook`) que contém Python, Pyspark e outras bibliotecas necessárias para a Parte 2.
      * Ele "dorme" (`command: sleep infinity`) para que possamos executar comandos de ETL dentro dele.

### 1.2. Artefato Principal: `docker-compose.yaml`

O arquivo `docker-compose.yaml` é o único artefato de configuração necessário para esta parte. Todas as configurações do servidor (como a conexão com o banco) são injetadas por meio de variáveis de ambiente definidas nele.

*(Neste ponto, você pode colar o conteúdo do `docker-compose.yaml` fornecido anteriormente dentro de um bloco de código, se desejar, ou apenas referenciar o arquivo).*

### 1.3. Passo a Passo para Instalação e Execução

Siga estes passos para configurar e executar o servidor FHIR.

#### 1\. Pré-requisitos

  * Docker Desktop (ou Docker Engine) instalado e em execução.
  * Docker Compose (geralmente incluído no Docker Desktop) instalado.

#### 2\. Estrutura de Pastas

Antes de começar, crie a seguinte estrutura de pastas para o seu projeto:

```
projeto-fhir/
├── docker-compose.yaml   <-- Cole o conteúdo fornecido aqui
├── data/
│   └── patients.csv      <-- Coloque seu arquivo CSV aqui
├── scripts/
│   └── load_data.py      <-- Você criará seu script de ETL aqui
└── README.md             <-- Este arquivo
```

#### 3\. Iniciando os Serviços

1.  Abra um terminal na pasta raiz do seu projeto (ex: `projeto-fhir/`).

2.  Execute o seguinte comando para construir e iniciar todos os serviços em segundo plano:

    ```bash
    docker-compose up -d
    ```

3.  O Docker irá baixar as imagens (Postgres, HAPI FHIR, Pyspark) e iniciar os containers. O HAPI FHIR pode levar de 1 a 2 minutos para iniciar completamente, pois ele precisa inicializar o banco de dados.

#### 4\. Verificando a Instalação

Você pode verificar se tudo está funcionando de duas maneiras:

1.  **Via Terminal:**
    Execute `docker-compose ps`. Você deve ver os três serviços (`hapi-fhir`, `fhir-db`, `etl-service`) com o status `Up` ou `running`.

    ```
    NAME           IMAGE                                    COMMAND                  SERVICE       CREATED         STATUS                   PORTS
    etl-service    jupyter/pyspark-notebook:latest          "tini -g -- start-no…"   etl-service   2 minutes ago   Up 2 minutes (healthy)   8888/tcp
    hapi-fhir      hapiproject/hapi-fhir-jpaserver-starter:latest   "/hapi-fhir-jpaser…"   hapi-fhir     2 minutes ago   Up 2 minutes (healthy)   0.0.0.0:8080->8080/tcp
    fhir-db        postgres:14                              "docker-entrypoint.s…"   fhir-db       2 minutes ago   Up 2 minutes (healthy)   5432/tcp
    ```

2.  **Via Navegador:**
    Abra seu navegador e acesse a URL base do servidor FHIR, solicitando seu "Capability Statement" (a descrição de tudo que o servidor pode fazer):

    > **http://localhost:8080/fhir/metadata**

    Se a instalação foi bem-sucedida, você verá um documento JSON (ou XML, dependendo do seu navegador) começando com `{"resourceType": "CapabilityStatement", ...}`.

-----

## Parte 2: Carga de Dados (Pipeline de Integração)

Com o servidor FHIR em execução (Parte 1), o próximo passo é carregar os dados do arquivo `data/patients.csv`.

### 2.1. Ambiente de ETL

O serviço `etl-service` no `docker-compose.yaml` foi criado para isso.

  * Ele já tem **Pyspark** e a biblioteca **requests** (para fazer chamadas HTTP) instalados.
  * Sua pasta local `./scripts` está mapeada para `/home/jovyan/scripts` dentro do container.
  * Sua pasta local `./data` está mapeada para `/home/jovyan/data` dentro do container.

### 2.2. Executando o Pipeline (Seus Próximos Passos)

1.  **Crie seu Script de Carga:** Crie o arquivo `scripts/load_data.py`.

2.  **Lógica do Script:** Dentro deste script, você deve:

      * Usar Pyspark (ou Pandas, se preferir) para ler o arquivo `/home/jovyan/data/patients.csv`.
      * Iterar sobre cada linha do CSV.
      * **Ponto Crítico:** Para se comunicar com o servidor FHIR de *dentro* do container `etl-service`, você **NÃO** deve usar `localhost`. Use o nome do serviço do Docker:
          * **URL Correta:** `http://hapi-fhir:8080/fhir`
      * Para cada linha, monte o JSON do `Resource Patient` e faça um `POST` para `http://hapi-fhir:8080/fhir/Patient`.
      * Se a coluna "observação" existir, pegue o ID do paciente recém-criado e monte o JSON do `Resource Observation`, fazendo um `POST` para `http://hapi-fhir:8080/fhir/Observation`.

3.  **Execute o Script:**
    Quando seu script `load_data.py` estiver pronto, execute o seguinte comando no seu terminal (na mesma pasta do `docker-compose.yaml`):

    ```bash
    # Se estiver usando Pyspark
    docker-compose exec etl-service spark-submit /home/jovyan/scripts/load_data.py

    # Se for um script Python simples (sem Pyspark, talvez usando Pandas + Requests)
    docker-compose exec etl-service python3 /home/jovyan/scripts/load_data.py
    ```

### 2.3. Verificando os Dados Carregados

Após executar o script, você pode verificar se os dados foram carregados acessando as seguintes URLs no seu navegador:

  * **Para ver todos os Pacientes:** `http://localhost:8080/fhir/Patient`
  * **Para ver todas as Observações:** `http://localhost:8080/fhir/Observation`

-----

## Gerenciamento do Ambiente

Para parar todos os serviços, execute:

```bash
docker-compose down
```

Se você quiser parar os serviços E remover o volume do banco de dados (para começar do zero), execute:

```bash
docker-compose down -v
```
