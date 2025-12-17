# 🗄️ aws-s3-notifications-lab: S3 Bucket Sharing with SNS Notifications

![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![S3](https://img.shields.io/badge/S3-569A31?style=for-the-badge&logo=amazon-s3&logoColor=white)
![SNS](https://img.shields.io/badge/SNS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![IAM](https://img.shields.io/badge/IAM-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Lab](https://img.shields.io/badge/TYPE-INTERMEDIATE_LAB-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/STATUS-COMPLETED-green?style=for-the-badge)

**Implementação de bucket S3 compartilhado com usuário externo e sistema de notificações via Amazon SNS para monitoramento de alterações em tempo real.**

## 🎯 OBJETIVOS DE APRENDIZADO

- ✅ Criar e configurar bucket S3 com permissões granulares
- ✅ Gerenciar políticas IAM para usuários externos
- ✅ Implementar notificações de eventos S3 via SNS
- ✅ Testar permissões de upload/download/exclusão
- ✅ Validar fluxo de notificações por email
- ✅ Configurar segurança com Block Public Access

## 🏗️ CENÁRIO DA ARQUITETURA

| Componente | Especificação | Finalidade |
|------------|---------------|-------------|
| **EC2 Instance** | CLI Host (Amazon Linux 2) | Ambiente para configuração AWS CLI |
| **S3 Bucket** | cafe-200619 | Bucket para compartilhamento de imagens |
| **IAM User** | mediacouser | Usuário externo da empresa de mídia |
| **IAM Group** | mediaco | Grupo com políticas S3 específicas |
| **SNS Topic** | s3NotificationTopic | Tópico para notificações de eventos |
| **SNS Subscription** | Email endpoint | Recebimento de notificações por email |

![Diagrama da Arquitetura AWS Cloud](foto1.jpeg)

## 🛠️ EXECUÇÃO PASSO A PASSO

### 1. Configuração Inicial do Ambiente

```bash
# Configurar AWS CLI com credenciais voclabs
aws configure
```

**Credenciais configuradas:**
```
AWS Access Key ID: AXIACCDDEPO9784YV5W
Default region: us-west-2
Output format: json
```

![Configuração AWS CLI no EC2](foto2.jpeg)

### 2. Criação do Bucket S3

```bash
# Primeira tentativa (nome indisponível)
aws s3 mb s3://cafe-2006 --region us-west-2

# Segunda tentativa - sucesso
aws s3 mb s3://cafe-200619 --region us-west-2
```

**Resultado:**
```
make_bucket: cafe-200619
```

### 3. Upload de Imagens Iniciais

```bash
# Sincronizar imagens da pasta local para S3
aws s3 sync ~/initial-images/ s3://cafe-200619/images
```

**Upload realizado:**
```
upload: initial-images/Strawberry-Tarts.jpg to s3://cafe-200619/images/Strawberry-Tarts.jpg
upload: initial-images/Cup-of-Hot-Chocolate.jpg to s3://cafe-200619/images/Cup-of-Hot-Chocolate.jpg
upload: initial-images/Donuts.jpg to s3://cafe-200619/images/Donuts.jpg
```

### 4. Verificação do Conteúdo do Bucket

```bash
# Listar objetos com sumário
aws s3 ls s3://cafe-200619/images/ --human-readable --summarize
```

**Resultado:**
```
2025-11-29 00:18:57  308.7 KiB cup-of-Hot-cChocolate.jpg
2025-11-29 00:18:57  371.8 KiB Donuts.jpg
2025-11-29 00:18:57  468.0 KiB Strawberry-Tarts.jpg
Total Objects: 3
Total Size: 1.1 MiB
```

![Listagem de imagens no bucket](foto3.jpeg)

### 5. Configuração de Credenciais mediacouser

```bash
# Limpar configuração atual
rm -rf ~/.aws/

# Configurar com credenciais do mediacouser
aws configure
```

**Credenciais mediacouser:**
```
AWS Access Key ID: AKIAJACROKPSORSTYNIB
AWS Secret Access Key: [oculto]
Default region: us-west-2
Default output: json
```

![Credenciais de acesso do mediacouser](foto4.jpeg)

### 6. Testes com Usuário mediacouser

#### Teste de Upload
```bash
# Upload de nova imagem
aws s3api put-object --bucket cafe-200619 --key images/Caramel-Delight.jpg --body ./new-images/Caramel-Delight.jpg
```

**Resultado:**
```json
{
    "ETag": "\"31ac30da613244b0ce78cf106eef3df7\"",
    "ServerSideEncryption": "AES256"
}
```

#### Teste de Download
```bash
# Download de imagem existente
aws s3api get-object --bucket cafe-200619 --key images/Donuts.jpg Donuts.jpg
```

**Resultado:**
```json
{
    "AcceptRanges": "bytes",
    "ContentType": "image/jpeg",
    "LastModified": "Sat, 29 Nov 2025 00:18:57 GMT",
    "ContentLength": 380732,
    "ETag": "\"40b50bec53cb5aa71dc967dc1422bf4\"",
    "ServerSideEncryption": "AES256",
    "Metadata": {}
}
```

#### Teste de Exclusão
```bash
# Excluir imagem
aws s3api delete-object --bucket cafe-200619 --key images/Strawberry-Tarts.jpg
```

#### Teste de Upload via Console Web
![Upload via Console AWS](foto7.jpeg)

**Arquivo uploadado com sucesso:**
- `cotovaineic.png` (427.8 KB)
- Status: Bem-sucedida

### 7. Configuração de Notificações SNS

#### Criação do Tópico SNS
```bash
# Criar tópico SNS para notificações
aws sns create-topic --name s3NotificationTopic
```

**Tópico criado com sucesso:**
```
ARN: arn:aws:sns:us-west-2:967515471709:s3NotificationTopic
```

![Criação do tópico SNS](foto9.jpeg)

#### Configuração da Política de Acesso do Tópico

**Arquivo de política:**
```json
{
    "Version": "2008-10-17",
    "Id": "S3PublishPolicy",
    "Statement": [
        {
            "Sid": "AllowPublishFromS3",
            "Effect": "Allow",
            "Principal": {
                "Service": "s3.amazonaws.com"
            },
            "Action": "SNS:Publish",
            "Resource": "arn:aws:sns:us-west-2:967515471709:s3NotificationTopic",
            "Condition": {
                "ArnLike": {
                    "aws:SourceArn": "arn:aws:s3:::cafe-200619"
                }
            }
        }
    ]
}
```

![Configuração da política SNS](foto11.jpeg)

#### Criação de Subscription Email
```bash
# Criar subscription com endpoint email
aws sns subscribe \
    --topic-arn arn:aws:sns:us-west-2:967515471709:s3NotificationTopic \
    --protocol email \
    --notification-endpoint wagnerprecimei06@gmail.com
```

**Subscription criada:**
```
Subscription ARN: arn:aws:sns:us-west-2:967515471709:s3NotificationTopic:04e73fe7-afe8-4b75-a574-0092cf953078
```

![Configuração da subscription SNS](foto10.jpeg)

### 8. Configuração de Event Notifications no S3

**Arquivo de configuração de eventos:**
```json
{
    "TopicConfigurations": [
        {
            "TopicArn": "arn:aws:sns:us-west-2:967515471709:s3NotificationTopic",
            "Events": ["s3:ObjectCreated:", "s3:ObjectRemoved:"],
            "Filter": {
                "Key": {
                    "FilterRules": [
                        {
                            "Name": "prefix",
                            "Value": "images/"
                        }
                    ]
                }
            }
        }
    ]
}
```

**Aplicar configuração:**
```bash
aws s3api put-bucket-notification-configuration \
    --bucket cafe-200619 \
    --notification-configuration file://s3EventNotification.json
```

### 9. Testes de Notificações

#### Notificação de Teste
**Email recebido:**
```json
{
    "Service": "Amazon S3",
    "Event": "s3:TestEvent",
    "Time": "2025-11-29T09:44:42.759Z",
    "Bucket": "cafe-200619",
    "RequestId": "FPATXFFPSGX5C6KW",
    "HostId": "MIUMIAs/QYNXM9FSA+35MnJWU1Tngdns1Ua0f9VwL22cmUc8/Tgq65r6i8hdEUu7LupASu4c4="
}
```

#### Notificação de Upload
**Email recebido após upload:**
```json
{
    "eventVersion": "1.0",
    "eventSource": "aws:s3",
    "awsRegion": "us-west-2",
    "eventTime": "2025-11-29T09:45:28.845Z",
    "eventName": "ObjectCreated:Put",
    "userIdentity": {"principalId": "AWS:AIDACCRDXP6D5ESZCCD"},
    "requestParameters": {"sourceIPAddress": "44.251.21.217"},
    "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "NDOS/OAAYWYNHT1Z00MzhAdLTkJlWgZTFMAeAZWOYYnRF",
        "bucket": {
            "name": "cafe-200619",
            "ownerIdentity": {"principalId": "A3GTBBBLSIFDJ"},
            "arn": "arn:aws:s3:::cafe-200619"
        },
        "object": {
            "key": "images/",
            "size": 0,
            "eTag": "d41d8cd98f00b204e9800998ecf8427e",
            "sequencer": "00692A4228CCDZF4EF"
        }
    }
}
```

![Notificações SNS recebidas por email](foto13.jpeg)

### 10. Testes Adicionais de Funcionalidades

#### Upload de Arquivo de Teste
```bash
# Criar arquivo de teste
echo "Test image file" > test-image.jpg

# Upload para S3
aws s3 cp test-image.jpg s3://cafe-200619/images/
```

#### Verificação Recursiva
```bash
# Listar todos os objetos recursivamente
aws s3 ls s3://cafe-200619/images/ --recursive
```

#### Remoção com Delay para Notificação
```bash
# Aguardar processamento
sleep 10

# Remover arquivo de teste
aws s3 rm s3://cafe-200619/images/test-image.jpg
```

![Testes de upload e remoção](foto12.jpeg)

#### Teste de Acesso Público Bloqueado
```bash
# Tentativa de tornar objeto público (deve falhar)
aws s3api put-object-acl --bucket cafe-200619 --key images/Donuts.jpg --acl public-read
```

**Erro esperado:**
```
An error occurred (AccessDenied) when calling the PutObjectAcl operation: 
User: arn:aws:iam::967515471709:user/mediacouser is not authorized to perform: 
s3:PutObjectAcl on resource: "arn:aws:s3:::cafe-200619/images/Donuts.jpg" 
because public ACLs are prevented by the BlockPublicAccess setting in S3 Block Public Access.
```

![Teste de acesso público bloqueado](foto15.jpeg)

### 11. Acesso via Browser

**URL do objeto:**
```
https://cafe-200619.s3.us-west-2.amazonaws.com/images/Notowagner.png?X-Amz-Algorithm=...
```

![Acesso ao objeto via navegador](foto8.jpeg)

## ⚡ ARQUITETURA TÉCNICA DETALHADA

### Fluxo de Notificações
```
┌─────────────────────────────────────────────────────────────────┐
│                  EC2 (CLI Host)                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. Upload/Delete via AWS CLI (mediacouser credentials) │   │
│  └───────────────────────┬─────────────────────────────────┘   │
│                          │                                      │
│                          ▼                                      │
│              ┌─────────────────────┐                           │
│              │   S3 Bucket         │                           │
│              │   cafe-200619       │                           │
│              │   images/ prefix    │                           │
│              └──────────┬──────────┘                           │
│                         │ 2. Event Trigger                     │
│                         │    (ObjectCreated/ObjectRemoved)     │
│                         ▼                                      │
│              ┌─────────────────────┐                           │
│              │   SNS Topic         │                           │
│              │s3NotificationTopic  │                           │
│              └──────────┬──────────┘                           │
│                         │ 3. Publish Message                   │
│                         ▼                                      │
│              ┌─────────────────────┐                           │
│              │   Email Subscription│                           │
│              │   wagnerprecimei06  │                           │
│              │   @gmail.com        │                           │
│              └─────────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

### Políticas de Segurança Implementadas

#### 1. IAM Group Policy (mediaco)
```json
{
    "Statement": [
        {
            "Sid": "AllowGroupToSeeBucketListInTheConsole",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        },
        {
            "Sid": "AllowRootLevelListingOfTheBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::cafe-200619"
        },
        {
            "Sid": "AllowUserSpecificActionsOnlyInTheSpecificPrefix",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:GetObjectVersion",
                "s3:DeleteObjectVersion"
            ],
            "Resource": "arn:aws:s3:::cafe-200619/images/*"
        }
    ]
}
```

#### 2. SNS Access Policy
- Permite apenas o bucket `cafe-200619` publicar no tópico
- Usa condição `ArnLike` para restringir origem
- Garante que apenas eventos legítimos do S3 gerem notificações

#### 3. S3 Block Public Access
- Impede que usuários configurem ACLs públicas
- Mantém dados privados por padrão
- Previne exposição acidental de dados

## 🎓 CONCLUSÕES E COMPETÊNCIAS

### ✅ COMPETÊNCIAS DESENVOLVIDAS
1. **Gestão de Buckets S3**: Criação e configuração com prefixos organizacionais
2. **Controle de Acesso IAM**: Políticas granulares para usuários externos
3. **Notificações em Tempo Real**: Integração S3 → SNS → Email
4. **Segurança Multi-camada**: Combinação de IAM policies + Block Public Access
5. **Monitoramento Proativo**: Detecção automática de alterações no bucket

### 📚 LIÇÕES APRENDIDAS
- **Least Privilege**: Usuário externo tem acesso apenas ao prefixo `images/`
- **Event Filtering**: Notificações filtradas por prefixo e tipo de evento
- **Security Layers**: IAM + S3 Policies + Block Public Access
- **Real-time Alerts**: Notificações imediatas para auditoria e compliance

### 🚀 APLICAÇÕES PRÁTICAS
- **Colaboração Externa**: Compartilhamento seguro com parceiros
- **Monitoramento de Conteúdo**: Alertas para uploads/exclusões
- **Workflow de Aprovação**: Notificações para revisão de conteúdo
- **Backup e Versioning**: Controle de alterações em assets digitais

## 📚 RECURSOS E REFERÊNCIAS

### Comandos AWS CLI Utilizados
- `aws s3 mb` - Criar bucket S3
- `aws s3 sync` - Sincronizar diretório com S3
- `aws s3api put-object` - Upload com metadados
- `aws s3api get-object` - Download específico
- `aws s3api delete-object` - Exclusão de objeto
- `aws sns create-topic` - Criar tópico SNS
- `aws sns subscribe` - Criar subscription
- `aws s3api put-bucket-notification-configuration` - Configurar eventos

### Boas Práticas Implementadas
1. **Nomenclatura**: Buckets com prefixo identificador
2. **Organização**: Uso de prefixos para estrutura de pastas
3. **Segurança**: Block Public Access ativado por padrão
4. **Monitoramento**: Notificações para todas as operações críticas
5. **Controle de Acesso**: Políticas baseadas em prefixo específico

### Documentação Oficial
- [Amazon S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)
- [SNS Integration with S3](https://docs.aws.amazon.com/sns/latest/dg/sns-s3-as-notification-source.html)
- [IAM Policies for S3](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_s3.html)
- [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)

