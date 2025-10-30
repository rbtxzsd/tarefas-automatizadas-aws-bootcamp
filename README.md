# Desafio DIO: Infraestrutura como Código com AWS CloudFormation

Este repositório documenta a implementação do desafio "Automatizando com AWS CloudFormation" do bootcamp da [Digital Innovation One (DIO)](https://www.dio.me/).

O objetivo principal foi aplicar os conceitos de **Infraestrutura como Código (IaC)** para provisionar de forma automatizada uma arquitetura serverless na AWS, com base no caso de uso "Sistema de Processamento de Notas Fiscais" apresentado nas aulas.

## 🚀 Arquitetura Implementada

A solução implementada segue o padrão *event-driven* (orientado a eventos) e é 100% serverless, utilizando os principais serviços da AWS.

**Caso de Uso:** Sistema de Processamento de Notas Fiscais.

O fluxograma abaixo (captura de tela da aula) ilustra a arquitetura que foi provisionada:

*(Observação: Adicione a sua captura de tela/imagem aqui, por exemplo: `![Fluxograma da Arquitetura AWS](images/architecture.jpg)`)*

### Fluxo de Dados:

1.  **Upload:** O processo se inicia quando um usuário (ou um sistema externo) faz o upload de um arquivo JSON (contendo os dados da nota fiscal) em um bucket do **Amazon S3**.
2.  **Trigger:** O bucket S3 é configurado com uma notificação de evento (`s3:ObjectCreated:*`). Assim que um novo objeto é criado, ele aciona (trigger) automaticamente uma função **AWS Lambda**.
3.  **Processamento:** A função Lambda é executada, lendo o arquivo JSON que acabou de ser "upado". Ela realiza a validação dos dados e extrai os campos relevantes (ex: número da nota, cliente, valor, data).
4.  **Armazenamento:** Após o processamento, a Lambda grava os dados extraídos como um novo item em uma tabela do **Amazon DynamoDB**.
5.  **(Opcional) Consulta:** Um **Amazon API Gateway** é configurado para expor um endpoint RESTful. Este endpoint permite que outras aplicações consultem os dados das notas fiscais diretamente da tabela do DynamoDB, de forma segura e escalável.

## 🛠️ Implementação com AWS CloudFormation

O pilar central deste desafio é a automação. Toda a infraestrutura descrita acima foi provisionada utilizando um único template do AWS CloudFormation (`template.yaml`).

O uso do CloudFormation garante que o ambiente seja:
* **Repetível:** Posso criar e recriar a mesma infraestrutura dezenas de vezes em segundos.
* **Consistente:** Evita erros manuais de configuração no console da AWS.
* **Versionado:** O template (`template.yaml`) pode ser versionado no Git, permitindo rastrear mudanças.

### Recursos Provisionados pelo Template:

O template do CloudFormation (`template.yaml` neste repositório) define os seguintes recursos:

1.  **`AWS::S3::Bucket` (SourceBucket):**
    * O bucket de entrada para onde os arquivos JSON das notas fiscais são enviados.

2.  **`AWS::DynamoDB::Table` (InvoicesTable):**
    * A tabela NoSQL para armazenar os dados processados.
    * Definida com uma chave de partição (ex: `invoice_id`) para acesso rápido.

3.  **`AWS::IAM::Role` (LambdaExecutionRole):**
    * A "identidade" da nossa função Lambda. É aqui que definimos as permissões.
    * **Permissões concedidas:**
        * `s3:GetObject`: Para que a Lambda possa ler o arquivo JSON do bucket S3.
        * `dynamodb:PutItem`: Para que a Lambda possa gravar os dados na tabela DynamoDB.
        * `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`: Permissões básicas para que a Lambda possa gerar logs no CloudWatch (essencial para debugging).

4.  **`AWS::Lambda::Function` (ProcessInvoiceFunction):**
    * O "cérebro" da operação.
    * Define o *runtime* (ex: `python3.11`), o *handler* (o ponto de entrada do código) e anexa a `LambdaExecutionRole` criada acima.
    * O código-fonte da função (no meu caso, um `lambda_function.py`) foi empacotado e referenciado pelo template.

5.  **`AWS::Lambda::Permission` (S3InvokePermission):**
    * A política que **autoriza** o serviço do S3 a invocar nossa função Lambda. Sem isso, o trigger não funciona.

6.  **`AWS::S3::BucketNotificationConfiguration`:**
    * Este é o recurso que "cola" o S3 e o Lambda.
    * É configurado dentro do `AWS::S3::Bucket` e especifica:
        * **Evento:** `s3:ObjectCreated:*` (qualquer criação de objeto).
        * **Destino:** O ARN (Amazon Resource Name) da nossa `ProcessInvoiceFunction`.

## ⚙️ Como Executar/Testar

1.  **Pré-requisitos:**
    * Conta AWS.
    * AWS CLI configurado localmente.

2.  **Empacotar (se o código Lambda estiver separado):**
    * Para que o CloudFormation encontre o código da Lambda, usamos o comando `package`. Ele faz o upload do código-fonte para um bucket S3 de "stage" e gera um novo template (ex: `packaged-template.yaml`) com a referência correta.
    ```bash
    aws cloudformation package \
        --template-file template.yaml \
        --s3-bucket <SEU_BUCKET_PARA_CODIGO> \
        --output-template-file packaged-template.yaml
    ```

3.  **Deploy:**
    * O comando `deploy` lê o template (o original ou o empacotado) e cria/atualiza a *stack* no CloudFormation.
    ```bash
    aws cloudformation deploy \
        --template-file packaged-template.yaml \
        --stack-name ProcessamentoNotasFiscaisStack \
        --capabilities CAPABILITY_IAM
    ```
    *A flag `--capabilities CAPABILITY_IAM` é obrigatória, pois estamos criando uma IAM Role.*

4.  **Teste:**
    * Após o deploy, basta ir ao Console do S3, encontrar o bucket criado e fazer o upload de um arquivo JSON de teste.
    * Em segundos, é possível verificar a tabela do DynamoDB e ver o novo item criado pela Lambda.

## 💡 Principais Aprendizados e Insights

* **O Poder do IaC:** É impressionante ver toda a arquitetura subindo em menos de um minuto com um único comando. Destruir tudo também é simples (`aws cloudformation delete-stack`), o que é perfeito para estudar e evitar custos.
* **IAM é o Coração (e a Dor):** A parte mais desafiadora é, sem dúvida, acertar as permissões IAM. A `LambdaExecutionRole` precisa de permissão para LER do S3 e ESCREVER no DynamoDB. Além disso, o S3 precisa de uma `Lambda::Permission` para INVOCAR a função. Entender essa relação foi o maior "clique" do desafio.
* **A "Mágica" do Trigger:** A conexão entre S3 e Lambda não é tão automática quanto parece. Ela depende de dois recursos: a `BucketNotificationConfiguration` (no S3) e a `Lambda::Permission` (na Lambda). O CloudFormation abstrai isso muito bem.
* **Serverless é Liberdade:** Não me preocupei com servidores, sistemas operacionais, escalabilidade ou patches de segurança. A AWS gerencia tudo. O foco foi 100% na lógica de negócio (o código Python da Lambda) e na definição da infraestrutura (o template CloudFormation).

## 🔗 Recursos Úteis

* **Documentação Oficial do AWS CloudFormation:** [AWS CloudFormation User Guide](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
* **Documentação do AWS SAM:** (Uma alternativa que também usa CloudFormation, mas é mais focada em Serverless): [AWS Serverless Application Model](https://aws.amazon.com/pt/serverless/sam/)
