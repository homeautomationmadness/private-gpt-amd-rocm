# PrivateGPT on AMD Radeon GPU on Docker


![privategpt](https://cdn-images-1.medium.com/max/1000/1*N2bhoET7nx26HfwWvyjyBw.jpeg)

**PrivateGPT is a production-ready AI project that allows you to ask questions about your documents using the power of Large Language Models (LLMs), even in scenarios without an Internet connection. 100% private, no data leaves your execution environment at any point.**


![Docker Image Size (latest by date)](https://img.shields.io/docker/image-size/homeautomationmadness/private-gpt-amd-rocm)
![Docker Pulls](https://img.shields.io/docker/pulls/homeautomationmadness/private-gpt-amd-rocm)


`GitHub` - homeautomationmadness/private-gpt-amd-rocm - https://github.com/homeautomationmadness/private-gpt-amd-rocm  
`DockerHub` - homeautomationmadness/private-gpt-amd-rocm - https://hub.docker.com/r/homeautomationmadness/private-gpt-amd-rocm

## Documentation
`GitHub` - zylon-ai/private-gpt - https://github.com/zylon-ai/private-gpt  
`Docs` - docs.privategpt.dev - https://docs.privategpt.dev/

## Index

1. [Usage](#usage)
   1.1 [docker build](#dockerbuild)  
   1.2 [docker-compose.yaml use Ollama](#docker-compose)  
   1.3 [docker-compose.yaml without Ollama](#docker-compose-custom)
2. [Environment Variables](#environment-variables)
3. [Volumes](#volumes)
4. [Ports](#ports)
5. [Running](#running-it)
6. [Find Me](#findme)
7. [License](#license)

## 1 Usage <a name="usage"></a>

### 1.1 docker build <a name="dockerbuild"></a>
So if you have the desire or expertise you can build this container image yourself by running the command below. 

You will need a Huggingface token: without it, the build will fail.

Pull the project from GitHub:

```shell
git pull https://github.com/homeautomationmadness/private-gpt-amd-rocm.git
cd private-gpt-amd-rocm/build
```

When this is built, it will be stored in your local docker instances as a usable container image.

```shell
docker build --no-cache --progress=plain --rm --build-arg \ 
HUGGINGFACE_TOKEN=hf_<your TOKEN from huggingface> \
-f Dockerfile \
-t homeautomationmadness/private-gpt-amd-rocm:latest \
-t homeautomationmadness/private-gpt-amd-rocm:V1.0 .
```

If the build is successful, proceed to [1.2 docker-compose.yml](#docker-compose).

If the build happens to fail, look at the following section [1.1.1 Troubleshoot the build](#trobleshootbuild).

#### 1.1.1 Troubleshoot the build <a name="trobleshootbuild"></a>

The following command will provide a more verbose output to a log file in the root directory of the project folder.

```shell
docker build --no-cache --progress=plain --rm --build-arg  \ 
HUGGINGFACE_TOKEN=hf_<your TOKEN from huggingface> \
-f Dockerfile \
-t homeautomationmadness/private-gpt-amd-rocm:latest \
-t homeautomationmadness/private-gpt-amd-rocm:V1.0 . \
 2>&1 | tee ../build.log
```

### 1.2 docker-compose.yml use Ollama <a name="docker-compose"></a>
Ollama will be the core and the workhorse of this setup the image selected is tuned and built to allow the use of selected AMD Radeon GPUs. This provides the benefits of it being ready to run on AMD Radeon GPUs, centralised and local control over the LLMs (Large Language Models) that you choose to use.

The Private GPT image that you can build using the provided docker file or the the already compiled image allows you to use it independently or link to Ollama and external systems.

At the time of putting this together, I used Ollama for the queries and the chats and Private GPT to do the Embeddings To RAG.


Configuration will be done in the .env file 

```yaml
###############################################################
# Services
###############################################################
services:
  ################################################################  
  # Database systems
  ################################################################        
  # Google alloydbomni Sql
  alloydbomni:
    image: google/alloydbomni 
    container_name: alloydbomni
    hostname: alloydbomni
    restart: unless-stopped
    networks:
      ai_net:
    volumes:
      - "alloydbomni_data:/var/lib/postgresql/data"
    environment:
      - PGDEBUG
      - POSTGRES_DB=$PG_RAG
      - POSTGRES_USER=$priv_gpt_user
      - POSTGRES_PASSWORD=$priv_gpt_pwd 

  ###############################################################
  # AI
  ###############################################################
  ollama:
    image: ollama/ollama:rocm
    ports:
      - 11434:11434
    volumes:
      - ollama_config:/root/.ollama
    container_name: ollama
    security_opt:
      - "seccomp=unconfined"
    tty: true
    restart: always
    devices:
      - /dev/kfd
      - /dev/dri 
    environment:
      - OLLAMA_KEEP_ALIVE=24h
      - OLLAMA_HOST=0.0.0.0
      - HSA_OVERRIDE_GFX_VERSION=9.0.0
      - HSA_ENABLE_SDMA=0
    networks:
      ai_net:

  ollama-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: ollama-webui
    volumes:
      - ollama_webui_config:/app/backend/data
    depends_on:
      - ollama
    ports:
      - 8080:8080
    environment: # https://docs.openwebui.com/getting-started/env-configuration#default_models
      - OLLAMA_BASE_URLS=http://ollama:7869 #comma separated ollama hosts
      - ENV=dev
      - WEBUI_AUTH=False
      - WEBUI_NAME="Home Automation Madness AI"
      - WEBUI_URL=http://localhost:8080
      - WEBUI_SECRET_KEY=t0p-s3cr3t
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: unless-stopped
    networks:
      ai_net:

  privategpt:
    image: homeautomationmadness/private-gpt-amd-rocm:latest
    container_name: privategpt
    devices:
      - /dev/kfd
      - /dev/dri 
    environment:
      ### Local Processing
      # LLAMACPP_LLM_HF_REPO_ID: $llamacpp_llm_hf_repo_id
      # LLAMACPP_LLM_HF_MODEL_FILE: $llamacpp_llm_hf_model_file
      # LLAMACPP_EMBEDDING_HF_MODEL_NAME: $llamacpp_embedding_hf_model_name
      # LLM_TOKENIZER: $llm_tokenizer
      # EMBEDDING_INGEST_MODE: $embedding_ingest_mode
      # EMBEDDING_COUNT_WORKERS: $embedding_count_workers
      # EMBEDDING_MODE: $embedding_mode
      # HUGGINGFACE_TOKEN: $huggingface_token
      PGPT_PROFILES: $pgpt_profiles
      RUN_SETUP: $run_setup
      ### Ollama settings
      LLM_MODE: $llm_mode
      OLLAMA_API_BASE: $ollama_api_base
      OLLAMA_EMBEDDING_API_BASE: $ollama_embedding_api_base
      OLLAMA_LLM_MODEL: $ollama_llm_model
      OLLAMA_EMBEDDING_MODEL: $ollama_embedding_model
      EMBEDDING_MODE: $embedding_mode
      EMBEDDING_INGEST_MODE: $embedding_ingest_mode
      ### Related to Postgres & Ollama
      EMBEDDING_EMBED_DIM: $embedding_embed_dim
      # ### Postgres Rag
      NODESTORE_DATABASE: $nodelistore_database
      VECTORSTORE_DATABASE: $vectorstore_database
      POSTGRES_HOST: $postgress_host
      POSTGRES_PORT: $postgres_port
      POSTGRES_DATABASE: $pg_rag
      POSTGRES_USER: $priv_gpt_user
      POSTGRES_PASSWORD: $priv_gpt_pwd
      POSTGRES_SCHEMA_NAME: $postgres_schema_name # - the postgres schema name - Default: private_gpt
      ## RAG extraction
      RAG_SIMILARITY_TOP_K: $rag_similarity_top_k
      RAG_RERANK_ENABLED: $rag_rerank_enabled
      RAG_RERANK_MODEL: $rag_rerank_model
      RAG_RERANK_TOP_N: $rag_rerank_top_n
      ### sYSTEM STUFF
      PUID: $PUID
      PGID: $PGID
      TZ: $TZ
    volumes:
      - privategpt_data:/home/worker/app/local_data
      - privategpt_models:/home/worker/app/models
    ports:
      - 8197:8080/tcp
    networks:
      ai_net:

###############################################################
# Networks
###############################################################
  ai_net:
    name: ai_net
    driver: bridge

###############################################################
# Volumes
###############################################################
volumes:
  alloydbomni_data:
    driver: local
    name: alloydbomni_data
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '$CONTAINERCONFIG/private-gpt-rocm-docker/docker-data/ollama/config'

  ollama_config:
    driver: local
    name: ollama_config
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '$CONTAINERCONFIG/private-gpt-rocm-docker/docker-data/ollama/config'
  
  ollama_webui_config:
    driver: local
    name: ollama_webui_config
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '$CONTAINERCONFIG/ollama_webui_config/config/'

  privategpt_models:
    driver: local
    name: privategpt_models
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '$CONTAINERCONFIG/privategpt/models/'

  privategpt_data:
    driver: local
    name: privategpt_data
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '$CONTAINERCONFIG/privategpt/data/'

```

See [2. Environment Variables](#environment-variables) below to update accordingly. 

### 1.2 docker-compose.yml with custom model <a name="docker-compose-custom"></a>

### 2 Environment Variables <a name="environment-variables"></a>

#### 2.1 System variables
Environment variables are used in this setup to control most settings, the default .env file looks like the one below, it is located in the ./docker folder with the docker-compose.yml file above.

Following this we will look at initial values to update and how to get some of them.


```yaml
################################################################  
# Container system settings
################################################################ 
COMPOSE_PROJECT_NAME=Home_Auto_AI
CONTAINERCONFIG="<Docker Container Path>"
PUID=<get from running id on linux command line>
PGID=<get from running id on linux command line>
TZ=Europe/London

################################################################  
# Container image versions
################################################################ 
OLLAMA_SW_VER=rocm              # Ollama AMD compiled Version
PRIVATEGPT=latest               # Private gpt AMD ROCM version

################################################################  
# Container services ports
################################################################  
PORTAINER_PORT=9002

################################################################  
# AlloyDB Omni 
################################################################  
PG_RAG=private_ai
priv_gpt_user=postgres
priv_gpt_pwd=<aplhanumeric password>

################################################################  
# Private GPT
################################################################  
### local processing
llamacpp_llm_hf_repo_id="Cyborg-AI/mystral-hf-7b-v0.1.gguf"
llamacpp_llm_hf_model_file="mystral-hf-7b-v0.1.gguf"
llamacpp_embedding_hf_model_name="BAAI/bge-small-en-v1.5"
llm_tokenizer=mistralai/Mistral-7B-Instruct-v0.2
embedding_ingest_mode=parallel
embedding_count_workers=4
embedding_mode=huggingface
huggingface_token=<hugginface token>
pgpt_profiles=local
run_setup=true

### Ollama settings
llm_mode=ollama
ollama_api_base=http://ollama:11434
ollama_embedding_api_base=http://ollama:11434
ollama_llm_model=mistral # llama3:8b
ollama_embedding_model=BAAI/bge-small-en-v1.5 # nomic-embed-text
embedding_mode=ollama
embedding_ingest_mode=simple

### related to Postgres & Ollama
embedding_embed_dim=384

### Postgres rag
nodelistore_database=postgres
vectorstore_database=postgres
postgres_host=alloydbomni # - the postgres host address - Default=postgres
postgres_port=5432 # - the postgres port - Default=5432
postgres_database=$pg_rag # - the postgres database name - Default=postgres
postgres_user=$priv_gpt_user # - the postgres username - Default=postgres
postgres_password=$priv_gpt_pwd # - the postgres usernames password - Default=admin
postgres_schema_name=<schema name> # - the postgres schema name - Default=private_gpt

### rag extraction
rag_similarity_top_k=10
rag_rerank_enabled=true
rag_rerank_model=cross-encoder/ms-marco-MiniLM-L-2-v2
rag_rerank_top_n=5

```
##### Container User settings

To update the PUID and PGID in the **Container system settings** section of the .env file.

```yaml
################################################################  
# Container system settings
################################################################ 
COMPOSE_PROJECT_NAME=Home_Auto_AI
CONTAINERCONFIG="<Docker Container Path>"
PUID=(User id number)
PGID=(Docker Group id number)
TZ=Europe/London

                  ............
```


To get the User ID (**PUID**) run:
```bash
id -u
```

To get the docker Group id (**PGID**)  run the following bash command:
```bash
getent group docker | cut -d: -f3
```

##### AlloyDB/Postgres SQL root user settings

```yaml
                  ............
################################################################  
# AlloyDB Omni 
################################################################  
PG_RAG=private_ai
priv_gpt_user=postgres
priv_gpt_pwd=(aplhanumeric password)
                  ...........
```

To create a random password for AlloyDB Omni, you can run the following command shown below. I would run it three times and then splice the mix together to form a 12 - 16 password as this will be the master user with God rights.



```bash
openssl rand -base64 18 | tr -dc 'a-z0-9' | head -c12; echo
```

Once you have this replace **(alphanumeric password)** with the code. This now makes Alloydb Omni or PostgreSQL ready for the next steps


##### Creating PrivateGPT RAG database & user
PrivateGPT supports many different backend databases in this use case Postgres SQL in the Form of [Googles AlloyDB Omni](https://cloud.google.com/alloydb/docs/omni) which is a Postgres SQL compliant engine written by Google for Generative AI and runs faster than Postgres native server.



For this lab, I have not used the best practices of using a different user and password but you should. once you are comfortable with the deployment.


```yaml

                  ............
                  
################################################################  
# AlloyDB Omni 
################################################################  
                  ............
### postgres rag
                  ............
postgres_host=alloydbomni # - the postgres host address - Default=postgres
postgres_port=5432 # - the postgres port - Default=5432
postgres_database=$pg_rag # - the postgres database name - Default=postgres
postgres_user=$priv_gpt_user # - the postgres username - Default=postgres
postgres_password=$priv_gpt_pwd # - the postgres usernames password - Default=admin
postgres_schema_name=(schema name) # - the postgres schema name - Default=private_gpt

```
##### Better Practice Create PrivateGPT Database and User
If you want to do initial deployment to follow best practice, you need to update the following variables: <a name="rootuserpwdbp"></a> 
```yaml
postgres_user=$priv_gpt_user
postgres_password=$priv_gpt_pwd
```

To do this, run the following commands:
```bash
cd Path_of_project 
cd docker
docker compose -f docker-compose.yml --env-file .env up -d alloydbomni
```

Execute the following commands to create the AlloyDB/postgres user and password.
```bash
docker compose -f docker-compose.yml --env-file .env exec -it alloydbomni psql -h localhost -U postgres

```
The terminal window displays psql login text that ends with a postgres=# prompt.

The following commands will then allow you to create a new User, Database, and set the user password and permissions to crud the database.

Make sure that you update the **'PASSWORD'** in the first line keeping it inside single quotes. The method used to create the Postgres password [above](#rootuserpwdbp) could be used.

```psql
CREATE USER private_gpt WITH PASSWORD 'PASSWORD';
CREATEDB private_gpt_db;
GRANT SELECT,INSERT,UPDATE,DELETE ON ALL TABLES IN SCHEMA public TO private_gpt;
GRANT SELECT,USAGE ON ALL SEQUENCES IN SCHEMA public TO private_gpt;
\q # This will quit psql client and exit back to your user bash prompt.
```
##### Using Ollama as LLM engine
This is the default for this lab setup which offloads most of the process to the Ollama engine and uses the models that you download there. so looking at the setting 
`ollama_embedding_model=BAAI/bge-small-en-v1.5 `

The LLMs ***BAAI/bge-small-en-v1.5*** & ***mistral*** need to exist on the Ollama engine otherwise you will get failure messages.

```yaml

                  ............
### Ollama settings
llm_mode=ollama
ollama_api_base=http://ollama:11434
ollama_embedding_api_base=http://ollama:11434
ollama_llm_model=mistral 
ollama_embedding_model=BAAI/bge-small-en-v1.5 
embedding_mode=ollama

                  ............

### postgres rag
                  ............
postgres_host=alloydbomni # - the postgres host address - Default=postgres
postgres_port=5432 # - the postgres port - Default=5432
postgres_database=$pg_rag # - the postgres database name - Default=postgres
postgres_user=$priv_gpt_user # - the postgres username - Default=postgres
postgres_password=$priv_gpt_pwd # - the postgres usernames password - Default=admin
postgres_schema_name=(schema name) # - the postgres schema name - Default=private_gpt

```

#### Letting Private GPT be the LLM Engine
If you do not want to use the Ollama engine, the LLMs can be run from within/on Private GPT instance.

For the following example config, I am using the Hugginface repository of LLMs. This requires you to have a Hugginface API key, this is currently free
. [See here for more info. ](https://huggingface.co/docs/hub/en/security-tokens).

These are already defined in the .env file and commented out. not the changes to ***llm_mode*** from **ollama** to **llamacpp** as well as the specifications of the LLM model info.

```yaml
################################################################  
# Private GPT
################################################################  
### local processing
llm_mode=llamacpp
llamacpp_llm_hf_repo_id="Cyborg-AI/mystral-hf-7b-v0.1.gguf"
llamacpp_llm_hf_model_file="mystral-hf-7b-v0.1.gguf"
llamacpp_embedding_hf_model_name="BAAI/bge-small-en-v1.5"
llm_tokenizer=mistralai/Mistral-7B-Instruct-v0.2
embedding_ingest_mode=parallel
embedding_count_workers=4
embedding_mode=huggingface
huggingface_token=<hugginface token>
```
For more customisations and alternative settings for connecting ChatGPT, SageMaker, Copilot etc, [see 2.2 Private GPT Settings below](#gpt-settings).

The big difference is apart from the downloading of the Large Language Models (LLMs) you are running this locally without dependency on the internet, which means your data is kept from transmission outside of your controlled domain.

#### 2.2 Private GPT settings <a name="gpt-settings"></a>
**you can adjust all values inside the [settings.yaml](https://github.com/homeautomationmadness/private-gpt-amd-rocm/blob/main/build/settings.yaml) with environment variables**

###### Server

- `ENV_NAME` - Name of the environment (prod, staging, local...) - **Default: prod**
- `PORT` - Port of PrivateGPT FastAPI server - **Default: 8080**
- `KEEP_FILES` - Specifies if the server should keep uploaded files after restarting the container (lowercase true or false)- **Default: true**
- `RUN_SETUP` - Set to true, to run poetry setup again. Do it only once to download models and set it to false afterwards - **Default: false**

###### Cors

- `CORS_ENABLED` - Flag indicating if CORS headers are set or not. If set to True, the CORS headers will be set to allow all origins, methods and headers - **Default: false**
- `CORS_ALLOW_CREDENTIALS` - Indicate that cookies should be supported for cross-origin requests - **Default: false**
- `CORS_ALLOW_ORIGINS` - A list of origins that should be permitted to make cross-origin requests - **Default: \***
- `CORS_ALLOW_ORIGIN_REGEX` - A regex string to match against origins that should be permitted to make cross-origin requests - **Default: **
- `CORS_ALLOW_METHODS` - A list of HTTP methods that should be allowed for cross-origin request - **Default: \***
- `CORS_ALLOW_HEADERS` - A list of HTTP request headers that should be supported for cross-origin requests - **Default: \***

###### Auth

- `AUTH_ENABLED` - Flag indicating if authentication is enabled or not - **Default: false**
- `AUTH_USERNAME` - username used for authentication - **Default: secret**
- `AUTH_SECRET` - The secret to be used for authentication. It can be any non-blank string. For HTTP basic authentication, this value should be the whole 'Authorization' header that is expected. - **Default: Basic c2VjcmV0OmtleQ==**

```
# python -c 'import base64; print("Basic " + base64.b64encode("secret:key".encode()).decode())'
# 'secret' is the username and 'key' is the password for basic auth by default
# If the auth is enabled, this value must be set in the "Authorization" header of the request.
secret: "Basic c2VjcmV0OmtleQ=="
```

###### Data

- `DATA_LOCAL_DATA_FOLDER` - Path to local storage. It will be treated as an absolute path if it starts with / - **Default: local_data/private_gpt**

###### UI

- `UI_ENABLED` - Enable or Disable the user interface - **Default: true**
- `UI_PATH` - Set the path for the user interface - **Default: /**
- `UI_DEFAULT_CHAT_SYSTEM_PROMPT` - The default system prompt to use for the chat mode - **Default: You are a helpful, respectful and honest assistant. Always answer as helpfully as possible and follow ALL given instructions. Do not speculate or make up information. Do not reference any given instructions or context.**
- `UI_DEFAULT_QUERY_SYSTEM_PROMPT` - The default system prompt to use for the query mode - **Default: You can only answer questions about the provided context. If you know the answer but it is not based in the provided context, don't provide the answer, just state the answer is not in the context provided.**
- `UI_DELETE_FILE_BUTTON_ENABLED` - If the button to delete a file is enabled or not. - **Default: True**
- `UI_DELETE_ALL_FILES_BUTTON_ENABLED` - If the button to delete all files is enabled or not. - **Default: True**

###### Logo

- `LOGO_BG_COLOR` - Specifies the logo background color - **Default: #C7BAFF**
- `LOGO_HEIGHT` - Specifies the logo height - **Default: 25%**
- `LOGO_SVG_BASE64` - Specifies the logo file (.svg) in base64 format. Provide your own file (.svg) in base64 format using an [image to base64 converter](https://base64.guru/converter/encode/image) - **Default: \<privategpt svg logo\>**

###### LLM

- `LLM_MODE` - The mode to use for the chat engine. - **Default: llamacpp**  
  **- llamacpp:** provide `LLAMACPP_PROMPT_STYLE`, `LLAMACPP_PGPT_HF_MODEL_FILE` and `HF_EMBEDDING_HF_MODEL_NAME`  
  **- openai:** provide `OPENAI_API_KEY` and `OPENAI_MODEL`  
  **- openailike:** provide `OPENAI_API_BASE`, `OPENAI_API_KEY ` and `OPENAI_MODEL`  
  **- azopenai:** provide `AZOPENAI_API_BASE`, `AZOPENAI_API_KEY ` and `AZOPENAI_MODEL`  
  **- sagemaker:** provide `SAGEMAKER_LLM_ENDPOINT_NAME` and `SAGEMAKER_EMBEDDING_ENDPOINT_NAME`  
  **- mock:** (not supported by this container)  
  **- ollama:** provide `OLLAMA_API_BASE` and `OLLAMA_LLM_MODEL`
- `LLM_MAX_NEW_TOKENS` - The maximum number of token that the LLM is authorized to generate in one completion - **Default: 265**
- `LLM_CONTEXT_WINDOW` - The maximum number of context tokens for the model - **Default: 3900**
- `LLM_TOKENIZER` - Specifies the model from Huggingface.co which is used as tokenizer - **Default: mistralai/Mistral-7B-Instruct-v0.2**
- `LLM_TEMPERATURE` - The temperature of the model. Increasing the temperature will make the model answer more creatively. A value of 0.1 would be more factual - **Default: 0.1**

###### Rag Settings

- `RAG_SIMILARITY_TOP_K` - This value controls the number of documents returned by the RAG pipeline - **Default: 2**
- `RAG_SIMILARITY_VALUE` - If set, any documents retrieved from the RAG must meet a certain match score. Acceptable values are between 0 and 1. - **Default: 0.45**
- `RAG_RERANK_ENABLED` - This value controls whether a reranker should be included in the RAG pipeline. - **Default: false**
- `RAG_RERANK_MODEL` - Rerank model to use. Limited to SentenceTransformer cross-encoder models. - **Default: cross-encoder/ms-marco-MiniLM-L-2-v2**
- `RAG_RERANK_TOP_N` - This value controls the number of documents returned by the RAG pipeline. - **Default: 1**

###### llamacpp

- `LLAMACPP_PROMPT_STYLE` - The prompt style to use for the chat engine. - **Default: mistral**  
  **- default:** use the default prompt style from the llama_index. It should look like `role: message`  
  **- llama2:** use the llama2 prompt style from the llama_index. Based on `<s>`, `[INST]` and `<<SYS>>`  
  **- tag:** use the tag prompt style. It should look like `<|role|>: message`  
  **- mistral:** use the mistral prompt style. It should look like `<s>[INST] {System Prompt} [/INST]</s>[INST] { UserInstructions } [/INST]`
  **- chatml**
- `LLAMACPP_LLM_HF_REPO_ID` - Name of the HuggingFace model to use for chat - **Default: TheBloke/Mistral-7B-Instruct-v0.2-GGUF**
- `LLAMACPP_LLM_HF_MODEL_FILE` - Specifies the llm model file. Can be a llm model name from the HuggingFace repo or a local file that you mounted via volume to /home/worker/app/models - **Default: mistral-7b-instruct-v0.2.Q4_K_M.gguf**
- `LLAMACPP_TFS_Z` - Tail free sampling is used to reduce the impact of less probable tokens from the output. A higher value (e.g., 2.0) will reduce the impact more, while a value of 1.0 disables this setting. - **Default: 1.0**
- `LLAMACPP_TOP_K` - Reduces the probability of generating nonsense. A higher value (e.g. 100) will give more diverse answers, while a lower value (e.g. 10) will be more conservative. - **Default: 40**
- `LLAMACPP_TOP_P` - Works together with top-k. A higher value (e.g., 0.95) will lead to more diverse text, while a lower value (e.g., 0.5) will generate more focused and conservative text. (Default: 0.9) - **Default: 0.9**
- `LLAMACPP_REPEAT_PENALTY` - Sets how strongly to penalize repetitions. A higher value (e.g., 1.5) will penalize repetitions more strongly, while a lower value (e.g., 0.9) will be more lenient. - **Default: 1.1**

###### Embedding

- `EMBEDDING_MODE` - The mode to use for the embedding engine. (see MODE) - **Default: huggingface**
  **you can additionally use huggingface**
- `EMBEDDING_INGEST_MODE` - The ingest mode to use for the embedding engine. - **Default: simple**  
  **- simple:** ingest files sequentially and one by one. It is the historic behaviour.  
  **- batch:** if multiple files, parse all the files in parallel, and send them in batch to the embedding model``.  
  **- parallel:** parse the files in parallel using multiple cores, and embedd them in parallel. (fastest mode for local setup)
  **- pipeline:** the Embedding engine is kept as busy as possible
- `EMBEDDING_COUNT_WORKERS` - The number of workers to use for file ingestion. Do not go too high with this number, as it might cause memory issues. (especially in parallel mode). Do not set it higher than your number of threads of your CPU. - **Default: 2**  
  **- for simple mode:** this number has no effect in simple mode.  
  **- for batch mode:** this is the number of workers used to parse the files.  
  **- for parallel mode:** this is the number of workers used to parse the files and embed them.
  **- for pipeline mode:** this is the number of workers that can perform embeddings.
- `EMBEDDING_EMBED_DIM` - The dimension of the embeddings stored in the Postgres database. - **Default: 384**

- **Specify the model used for embedding with `HUGGINGFACE_EMBEDDING_HF_MODEL_NAME`**

###### HuggingFace

- `HUGGINGFACE_EMBEDDING_HF_MODEL_NAME` - Name of the HuggingFace model to use for embeddings - **Default: BAAI/bge-small-en-v1.5**
- `HUGGINGFACE_TOKEN` - Your HuggingFace token - **Default: None**

###### Vectorstore

- `VECTORSTORE_DATABASE` - Specifies the vectorstore database being used. - **select one of: chroma, qdrant, postgres .Default: qdrant**

###### Nodestore

- `NODESTORE_DATABASE` - Specifies the nodestore database being used. - **select one of: simple, postgres .Default: simple**

###### qdrant

- `QDRANT_PATH` - Persistence path for QdrantLocal - **Default: local_data/private_gpt/qdrant**

###### Postgres

- `POSTGRES_HOST` - the postgres host address - **Default: postgres**
- `POSTGRES_PORT` - the postgres port - **Default: 5432**
- `POSTGRES_DATABASE` - the postgres database name - **Default: postgres**
- `POSTGRES_USER` - the postgres username - **Default: postgres**
- `POSTGRES_PASSWORD` - the postgres usernames password - **Default: admin**
- `POSTGRES_SCHEMA_NAME` - the postgres schema name - **Default: private_gpt**

###### Sagemaker

- `SAGEMAKER_LLM_ENDPOINT_NAME` - **Default: huggingface-pytorch-tgi-inference-2023-09-25-19-53-32-140**
- `SAGEMAKER_EMBEDDING_ENDPOINT_NAME` - **Default: huggingface-pytorch-inference-2023-11-03-07-41-36-479**

###### OpenAI

- `OPENAI_API_BASE` - Base URL of OpenAI API. Example: https://api.openai.com/v1 - **Default: https://api.openai.com/v1**
- `OPENAI_API_KEY` - Your API Key for the OpenAI API. Example: sk-1234 - **Default: sk-1234**
- `OPENAI_MODEL` - OpenAI Model to use. (see [OpenAI Models Overview](https://platform.openai.com/docs/models/overview)). Example: gpt-4 - **Default: gpt-3.5-turbo**
- `OPENAI_REQUEST_TIMEOUT` - Time elapsed until openailike server times out the request. Default is 120s. Format is float. - **Default: 120.0**
- `OPENAI_EMBEDDING_API_BASE` - Base URL of OpenAI API. Example: https://api.openai.com/v1 - **Default: same as OPENAI_API_BASE**
- `OPENAI_EMBEDDING_API_KEY` - Your API Key for the OpenAI Embedding API. Example: sk-1234. - **Default: same as OPENAI_API_KEY**
- `OPENAI_EMBEDDING_MODEL` - OpenAI embedding Model to use. Example: text-embedding-3-large - **Default: text-embedding-3-small**

###### Ollama

- `OLLAMA_API_BASE` - Base URL of Ollama API. Example: http://192.168.1.100:11434 - **Default: http://localhost:11434**
- `OLLAMA_EMBEDDING_API_BASE` - Base URL of Ollama Embedding API. Example: http://192.168.1.100:11434 - **Default: same as OLLAMA_API_BASE**
- `OLLAMA_LLM_MODEL` - Ollama model to use. (see [Ollama Library](https://ollama.com/library)). Example: 'llama2-uncensored' - **Default: mistral:latest**
- `OLLAMA_EMBEDDING_MODEL` - Model to use. Example: 'nomic-embed-text'. - **Default: nomic-embed-text**
- `OLLAMA_KEEP_ALIVE` - Time the model will stay loaded in memory after a request. examples: 5m, 5h, '-1' - **Default: 5m**
- `OLLAMA_TFS_Z` - Tail free sampling is used to reduce the impact of less probable tokens from the output. A higher value (e.g., 2.0) will reduce the impact more, while a value of 1.0 disables this setting. - **Default: 1.0**
- `OLLAMA_NUM_PREDICT` - Maximum number of tokens to predict when generating text. (Default: 128, -1 = infinite generation, -2 = fill context) - **Default: 128**
- `OLLAMA_TOP_K` - Reduces the probability of generating nonsense. A higher value (e.g. 100) will give more diverse answers, while a lower value (e.g. 10) will be more conservative. - **Default: 40**
- `OLLAMA_TOP_P` - Works together with top-k. A higher value (e.g., 0.95) will lead to more diverse text, while a lower value (e.g., 0.5) will generate more focused and conservative text. - **Default: 0.9**
- `OLLAMA_REPEAT_LAST_N` - Sets how far back for the model to look back to prevent repetition. (Default: 64, 0 = disabled, -1 = num_ctx) - **Default: 64**
- `OLLAMA_REPEAT_PENALTY` - Sets how strongly to penalize repetitions. A higher value (e.g., 1.5) will penalize repetitions more strongly, while a lower value (e.g., 0.9) will be more lenient. - **Default: 1.1**
- `OLLAMA_REQUEST_TIMEOUT` - Time elapsed until ollama times out the request. Default is 120s. Format is float. - **Default: 120.0**

###### Azure OpenAI

- `AZOPENAI_API_KEY` - Your API Key for the OpenAI API. Example: sk-1234 - **Default: sk-1234**
- `AZOPENAI_ENDPOINT` - Base URL of Azure OpenAI Endpoint. Example: https://api.myazure.com/v1 - **Default: https://api.myazure.com/v1**
- `AZOPENAI_API_VERSION` - The API version to use for this operation. This follows the YYYY-MM-DD format. - **Default: 2023_05_15**
- `AZOPENAI_EMBEDDING_DEPLOYMENT_NAME` - embedding deployment name in str format - **Default: None**
- `AZOPENAI_EMBEDDING_MODEL` - OpenAI Model to use. Example: 'text-embedding-ada-002'. - **Default: text-embedding-3-small**
- `AZOPENAI_LLM_DEPLOYMENT_NAME` - llm deployment name in str format - **Default: None**
- `AZOPENAI_LLM_MODEL` - OpenAI Model to use. (see [OpenAI Models Overview](https://platform.openai.com/docs/models/overview)). Example: gpt-4 - **Default: gpt-4**

### 3 Volumes <a name="volumes"></a>

- `/home/worker/app/local_data` - Directory for uploaded files. contains private data! Will be deleted after every restart if `KEEP_FILES=false`
- `/home/worker/app/models` - Directory for custom llm models. Mount your own model here and set environment variable `LLAMACPP_LLM_HF_MODEL_FILE`

### 4 Ports <a name="ports"></a>
**AlloyDB Omni**
- `5432/tcp` - HTTP Port

**Ollama**
- `11434/tcp` - HTTP Port

**Ollama Web UI**
- `8080/tcp` - HTTP Port

**Private GPT**
- `8197/tcp` - HTTP Port

### 5 Running <a name="running-it"></a>
First, check that you can run the docker-compose.yml file..
```bash
cd Path_of_project 
cd docker
docker compose -f docker-compose.yml --env-file .env config
```
The output of this if correct should be the same as the docker files, but with environment variables replaced with the values.

The next optional step in case you have slow internet is to perform a pull of the images.

```bash
docker compose -f docker-compose.yml --env-file .env pull
```

If the `docker pull`  been completed or you have skipped it run the following command to set up the environment.
```bash
docker compose -f docker-compose.yml --env-file .env up -d
```

Next steps,  if there are erros the those need to be sorted then after that first go to Ollama web UI on http://server_name_or_IP_address:8080 to download the desired models as set in the settings for PrivateGPT settings.

When that is done:
- **Login to Ollama**
  HTTP://*Ollama_UI_Server_Address*:8080 e.g. HTTP://192.168.1.1:8080
  - Got to Settings --> Models
    - Download Models according to the installation requirments
- **Login to Private GPT**
  HTTP://Ollama_UI_Server_Address:8197 e.g. HTTP://192.168.1.1:8197
  - Check connection to Ollama. 


### 6 Find Me <a name="findme"></a>

- [GitHub](https://github.com/homeautomationmadness)
- [DockerHub](https://hub.docker.com/u/homeautomationmadness)

Thanks to the followining Projects and tecnoligest for the insperation to create this project.
- NetworkChuck of [The NetworkChuck Academy](https://academy.networkchuck.com/)
- GitHub - [3x3cut0r](https://hub.docker.com/u/3x3cut0r) 
- GitHub - [HardAndHeavy](https://github.com/HardAndHeavy)

### 7 License <a name="license"></a>

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) - This project is licensed under the GNU General Public License - see the [gpl-3.0](https://www.gnu.org/licenses/gpl-3.0.en.html) for details.

