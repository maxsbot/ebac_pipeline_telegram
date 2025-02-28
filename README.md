# **EBAC | Projeto de Pipeline de Dados do Telegram** 

Bem-vindo(a) ao repositório do **Pipeline de Dados do Telegram**! Aqui você encontrará toda a documentação, código e instruções necessárias para reproduzir um pipeline completo que captura mensagens de um grupo do Telegram, armazena-as em um _bucket_ S3 no formato bruto (_raw_), enriquece esses dados no formato **Parquet** e disponibiliza os resultados para análise via **AWS Athena**. Este projeto foi desenvolvido em etapas, abrangendo **Ingestão**, **ETL** e **Apresentação** de dados.

---
## **Índice**
1. [Objetivo do Projeto](#objetivo-do-projeto)  
2. [Visão Geral da Arquitetura](#visao-geral-da-arquitetura)  
3. [Requisitos e Ferramentas Utilizadas](#requisitos-e-ferramentas-utilizadas)  
4. [Estrutura de Pastas e Arquivos](#estrutura-de-pastas-e-arquivos)  
5. [Passo a Passo do Projeto](#passo-a-passo-do-projeto)  
   - [Parte 1: Configuração do Bot e Grupo Telegram](#parte-1-configuração-do-bot-e-grupo-telegram)  
   - [Parte 2: Ingestão dos Dados (AWS S3, AWS Lambda e AWS API Gateway)](#parte-2-ingestão-dos-dados-aws-s3-aws-lambda-e-aws-api-gateway)  
   - [Parte 3: ETL (AWS Lambda e PyArrow)](#parte-3-etl-aws-lambda-e-pyarrow)  
   - [Parte 4: Apresentação (AWS Athena)](#parte-4-apresentação-aws-athena)  
6. [Notas Importantes e Boas Práticas](#notas-importantes-e-boas-práticas)  
7. [Como Executar Localmente (Testes)](#como-executar-localmente-testes)  
8. [Referências](#referências)

---

## **Objetivo do Projeto**
O principal objetivo deste projeto é exemplificar um pipeline completo de dados que parte da coleta de mensagens de um *chat* do Telegram até a análise em consultas no **AWS Athena**. As etapas cobrem:

- Criação e configuração de um *bot* no Telegram;
- Recebimento e armazenamento das mensagens (incluindo diversos tipos de conteúdo) em formato **JSON** no **Amazon S3** (_bucket_ *-raw*);
- Transformação e enriquecimento dos dados em **Parquet** (_bucket_ *-enriched*);
- Análise e exploração dos dados por meio de **AWS Athena**.

---

## **Visão Geral da Arquitetura**
A figura abaixo (ilustrativa) descreve a arquitetura básica do projeto:

```
[Usuario/Telegram] --(mensagens)--> [Grupo Telegram]
                        |
                        |  Webhook (API Bot do Telegram)
                        v
           [AWS API Gateway] -> [AWS Lambda de Ingestão] -> [S3 - RAW Bucket]
                                                             |
                                                             | (EventBridge Agendado)
                                                             v
                                                [AWS Lambda de ETL] -> [S3 - ENRICHED Bucket]
                                                                             |
                                                                             v
                                                                        [AWS Athena]
```

1. **Telegram**: É enviado um texto, imagem, vídeo ou outro arquivo no grupo.  
2. **API de Bot do Telegram**: O *bot* recebe via *webhook* os conteúdos das mensagens e dispara para a **API Gateway**.  
3. **AWS API Gateway**: Rota responsável por receber a requisição do Telegram e invocar a **AWS Lambda** de ingestão.  
4. **AWS Lambda (Ingestão)**: Recebe o corpo da mensagem (em JSON) e armazena os arquivos no bucket S3 com sufixo `-raw`.  
5. **AWS S3 (Bucket RAW)**: Armazenamento dos dados crus.  
6. **AWS EventBridge**: Agenda a execução diária para a **AWS Lambda** de ETL à meia noite (horário de Brasília, GMT-3).  
7. **AWS Lambda (ETL)**: Processa os arquivos brutos do dia anterior (ou do mesmo dia para testes), converte para **Parquet** e salva num bucket com sufixo `-enriched`.  
8. **AWS S3 (Bucket ENRICHED)**: Local onde ficam os dados já particionados e enriquecidos no formato **Parquet**.  
9. **AWS Athena**: Serviço que permite criar tabelas sobre os dados no S3 e executar consultas SQL para análise.  

---

## **Requisitos e Ferramentas Utilizadas**

- **Python 3+**  
- **Jupyter Notebook** ou Google Colab (para desenvolvimento e testes locais)  
- **Pacote `requests`** (para chamadas à API do Telegram)  
- **Pacote `dotenv`** (para carregar variáveis de ambiente)  
- **AWS**:
  - **Amazon S3** (para armazenamento de dados brutos e enriquecidos)  
  - **AWS Lambda** (para funções de ingestão e de ETL)  
  - **AWS IAM** (para configuração de permissões)  
  - **AWS API Gateway** (para expor endpoint público para o *webhook* do Telegram)  
  - **AWS EventBridge** (para agendamento da execução de ETL)  
  - **AWS Athena** (para consultas e análise dos dados)  

---
## **Estrutura de Pastas e Arquivos**

```
.
├── notebooks
│   ├── modulo_43_exercicio.ipynb        # Notebook referente à parte 1 (Telegram)
│   └── modulo_44_exercicio.ipynb        # Notebook referente às partes 2, 3 e 4 (Ingestão, ETL, Apresentação)
├── README.md                            # Este arquivo
└── .env                                 # Arquivo de variáveis de ambiente (NÃO COMMITAR)
```

**Obs**: É fundamental **não** versionar o arquivo `.env`, pois nele constam informações sensíveis, como o *token* do bot e a URL da API no Gateway.

---

## **Passo a Passo do Projeto**

### **Parte 1: Configuração do Bot e Grupo Telegram**
1. **Conta no Telegram**  
   - Se ainda não tiver uma conta, crie uma e acesse via [Telegram Web](https://web.telegram.org).
2. **Criação do Bot**  
   - Através do `@BotFather` no Telegram, gere o seu *token* de acesso.  
   - **Não compartilhe seu token publicamente**.
3. **Criação de um Grupo**  
   - Crie o grupo, convide o bot e **torne-o administrador**.
4. **Desabilitar Adição Automática em Novos Grupos**  
   - Nas configurações do bot, desabilite a opção de “permitir ser adicionado” a novos grupos para garantir segurança.
5. **Envio de Mensagens Diversificadas**  
   - No grupo, envie mensagens de texto, imagens, vídeos, arquivos, áudios e outros formatos para teste.
6. **Consumo via API de Bots do Telegram**  
   - Utilize o método `getUpdates` para ver as mensagens recebidas pelo bot.  
   - Confira exemplos no [notebook `modulo_43_exercicio.ipynb`](./notebooks/modulo_43_exercicio.ipynb).

### **Parte 2: Ingestão dos Dados (AWS S3, AWS Lambda e AWS API Gateway)**
1. **Criação do Bucket S3 para Dados Crus (`-raw`)**  
   - Nomeie o bucket no padrão `seu-nome-raw` (ou similar).  
2. **Função AWS Lambda para Armazenamento**  
   - Configure no código da Lambda para receber o *body* (JSON) do Telegram.  
   - Salve o arquivo no S3 com partição baseada na data (por exemplo, `YYYY-MM-DD`).
   - Ajuste as variáveis de ambiente (por exemplo, `BUCKET_RAW_NAME`, etc.).  
   - Adicione as permissões de escrita no S3 via `AWS IAM` (Role da Lambda).
3. **Integração com o AWS API Gateway**  
   - Crie uma API HTTP ou REST e associe a função Lambda.  
   - Garanta que o método `POST` (ou `ANY`) chame a Lambda corretamente.
4. **Configuração do Webhook do Bot**  
   - Chame `setWebhook` via API do Telegram apontando para a URL do API Gateway.  
   - Valide a configuração chamando `getWebhookInfo`.
   - **Não** divulgue a URL gerada.
5. **Teste**  
   - Envie mensagens no grupo e verifique se elas aparecem no bucket S3 *raw*.  
   - Para testes locais na Lambda, substitua `message = json.loads(event["body"])` por `message = event`.

### **Parte 3: ETL (AWS Lambda e PyArrow)**
1. **Criação do Bucket S3 para Dados Enriquecidos (`-enriched`)**  
   - Nomeie o bucket no padrão `seu-nome-enriched`.
2. **Função AWS Lambda para Processamento e Geração de Parquet**  
   - Leia os arquivos JSON referentes a uma data específica do bucket *raw*.  
   - Converta-os para um único arquivo **Parquet** por data, salvando no bucket *-enriched*.  
   - Utilize a _layer_ com **PyArrow** (ou faça *deployment* do pacote no código, caso prefira).  
   - Ajuste o *timeout* da função para evitar erros em lotes maiores.  
   - Dica: para testes, use `(datetime.now() - timedelta(days=0))` em vez de `days=1`.
3. **Agendamento com AWS EventBridge**  
   - Crie uma regra que executa a Lambda de ETL **diariamente** à meia-noite no fuso horário do Brasil (GMT-3).  
   - Assim, os dados do dia anterior ficarão prontos para análise na manhã seguinte.

### **Parte 4: Apresentação (AWS Athena)**
1. **Criação da Tabela no AWS Athena**  
   - Aponte para o _bucket_ de dados enriquecidos.  
   - Exemplo de criação de tabela (vide [notebook `modulo_44_exercicio.ipynb`](./notebooks/modulo_44_exercicio.ipynb)):
     ```sql
     CREATE EXTERNAL TABLE `telegram`(
       `message_id` bigint, 
       `user_id` bigint, 
       `user_is_bot` boolean, 
       `user_first_name` string, 
       `chat_id` bigint, 
       `chat_type` string, 
       `text` string, 
       `date` bigint)
     PARTITIONED BY (
       `context_date` date)
     ROW FORMAT SERDE
       'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
     STORED AS INPUTFORMAT
       'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
     OUTPUTFORMAT
       'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
     LOCATION
       's3://<bucket-enriquecido>/';
     ```
2. **Carregar Partições**  
   ```sql
   MSCK REPAIR TABLE telegram;
   ```
3. **Consultas Exemplares**  
   - **Quantidade de mensagens por dia**:
     ```sql
     SELECT context_date, count(1) AS message_amount
     FROM telegram
     GROUP BY context_date
     ORDER BY context_date DESC;
     ```
   - **Quantidade de mensagens por usuário e dia**:
     ```sql
     SELECT 
       user_id,
       user_first_name,
       context_date,
       count(1) AS message_amount
     FROM telegram
     GROUP BY user_id, user_first_name, context_date
     ORDER BY context_date DESC;
     ```
   - **Média do tamanho das mensagens por usuário e dia**:
     ```sql
     SELECT 
       user_id,
       user_first_name,
       context_date,
       CAST(AVG(length(text)) AS INT) AS average_message_length
     FROM telegram
     GROUP BY user_id, user_first_name, context_date
     ORDER BY context_date DESC;
     ```
   - **Quantidade de mensagens por hora, dia da semana e número da semana**:
     ```sql
     WITH parsed_date_cte AS (
       SELECT 
         *,
         CAST(date_format(from_unixtime("date"), '%Y-%m-%d %H:%i:%s') AS timestamp) AS parsed_date
       FROM telegram
     ),
     hour_week_cte AS (
       SELECT
         *,
         EXTRACT(hour FROM parsed_date) AS parsed_date_hour,
         EXTRACT(dow FROM parsed_date)  AS parsed_date_weekday,
         EXTRACT(week FROM parsed_date) AS parsed_date_weeknum
       FROM parsed_date_cte
     )
     SELECT
       parsed_date_hour,
       parsed_date_weekday,
       parsed_date_weeknum,
       count(1) AS message_amount
     FROM hour_week_cte
     GROUP BY parsed_date_hour, parsed_date_weekday, parsed_date_weeknum
     ORDER BY parsed_date_weeknum, parsed_date_weekday;
     ```
4. **Explorando os Resultados**  
   - Verifique se os dados estão corretos, se as partições estão atualizadas e se o formato **Parquet** está funcionando adequadamente.

---

## **Notas Importantes e Boas Práticas**

1. **Nunca compartilhe seu Token do Bot**  
   - Mantenha em variáveis de ambiente ou serviços de *secret manager*.
2. **Nunca compartilhe a URL do seu API Gateway**  
   - Evite ataques ou uso indevido do endpoint público.
3. **Controle de Acesso no S3**  
   - Se possível, utilize criptografia e políticas de acesso restrito.
4. **Logs e Monitoramento**  
   - Ative *CloudWatch Logs* para monitorar as execuções de suas Lambdas.
5. **Particionamento no S3**  
   - Manter uma estrutura de pastas como `s3://bucket-name/year=YYYY/month=MM/day=DD/...` auxilia na organização e performance em Athena.
6. **Limpeza de Dados**  
   - Dependendo do uso, pode ser interessante criar um processo para limpar dados antigos ou irrelevantes.

---

## **Como Executar Localmente (Testes)**

1. **Clonar o repositório**  
   ```bash
   git clone https://github.com/seu-usuario/seu-repo.git
   cd seu-repo
   ```
2. **Criar ambiente virtual e instalar dependências**  
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # ou .venv\Scripts\activate em Windows
   pip install -r requirements.txt
   ```
3. **Criar arquivo `.env`**  
   ```text
   TOKEN=<seu_bot_token>
   API=<sua_url_gerada_no_api_gateway>
   BUCKET_RAW_NAME=<seu-bucket-raw>
   BUCKET_ENRICHED_NAME=<seu-bucket-enriched>
   ```
4. **Executar Notebooks**  
   - Abra o **Jupyter Notebook** ou **Google Colab** e rode os passos em `notebooks/modulo_43_exercicio.ipynb` (parte 1) e `notebooks/modulo_44_exercicio.ipynb` (partes 2, 3 e 4).

---

## **Referências**
- [Documentação Oficial do Telegram Bot API](https://core.telegram.org/bots/api)  
- [Documentação do AWS S3](https://docs.aws.amazon.com/s3/index.html)  
- [Documentação do AWS Lambda](https://docs.aws.amazon.com/lambda/index.html)  
- [Documentação do AWS IAM](https://docs.aws.amazon.com/iam/index.html)  
- [Documentação do AWS API Gateway](https://docs.aws.amazon.com/apigateway/index.html)  
- [Documentação do AWS EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html)  
- [Documentação do AWS Athena](https://docs.aws.amazon.com/athena/index.html)  
- [Documentação PyArrow](https://arrow.apache.org/docs/python/)

---

### **Contato**
- Autor: [Max Santos / LinkedIn](https://www.linkedin.com/mxvinicius)  

---

**Obrigado(a) pela leitura e bons estudos!**   
Qualquer dúvida ou sugestão, fique à vontade para abrir uma _issue_ ou entrar em contato.
