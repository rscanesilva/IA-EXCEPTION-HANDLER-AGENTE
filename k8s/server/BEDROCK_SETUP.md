# Configuração do AWS Bedrock no Exception Intelligence Server

## Pré-requisitos

- Acesso ao AWS Console com permissão para IAM e Bedrock
- `kubectl` configurado apontando para o cluster EKS
- `aws cli` autenticado na conta AWS

---

## 1. Solicitar acesso ao modelo no AWS Bedrock

No AWS Console, acesse **Bedrock → Model access** na região desejada e solicite acesso ao modelo (ex: `anthropic.claude-sonnet-4-6`).

> A aprovação é automática e leva alguns segundos. É necessário fazer isso apenas uma vez por conta AWS.
> Na primeira vez, a AWS/Anthropic solicita uma descrição do caso de uso (máx. 500 caracteres).

---

## 2. Criar e anexar a policy IAM aos node roles do EKS

O servidor usa o IAM role dos nodes (sem access key) — mais seguro e sem credenciais no código.

### 2.1 Identificar todos os node roles do cluster

O cluster EKS pode possuir múltiplos nodegroups com roles distintos. A policy precisa estar em **todos**, pois o pod pode ser agendado em qualquer node.

Para descobrir os roles:

```bash
# Listar nodegroups e seus roles
aws eks list-nodegroups --cluster-name <CLUSTER_NAME> --region <REGION> \
  --query "nodegroups[]" --output text | tr '\t' '\n' | while read ng; do
  ROLE=$(aws eks describe-nodegroup --cluster-name <CLUSTER_NAME> --nodegroup-name "$ng" \
    --region <REGION> --query "nodegroup.nodeRole" --output text | awk -F'/' '{print $NF}')
  echo "$ng -> $ROLE"
done
```

### 2.2 Criar a policy

Modelos Claude 4+ no Bedrock utilizam **cross-region inference profiles** (prefixo `us.`),
que roteiam automaticamente para múltiplas regiões. Por isso a policy precisa:

- Usar `*` na região (o Bedrock pode rotear para `us-east-1`, `us-east-2`, `us-west-2`, etc.)
- Cobrir tanto `foundation-model` quanto `inference-profile` como resources

```bash
aws iam create-policy \
  --policy-name BedrockInvokeClaudePolicy \
  --description "Allows invoking Claude models via AWS Bedrock Converse API (cross-region)" \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:Converse"],
      "Resource": [
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",
        "arn:aws:bedrock:*:<AWS_ACCOUNT_ID>:inference-profile/*"
      ]
    }]
  }' \
  --region <REGION>
```

> **Gotchas resolvidos na criação dessa policy:**
>
> 1. **Região fixa** — cross-region inference profiles roteiam requests para outras regiões (ex: `us-east-2`). Se a policy restringir a uma região, as chamadas roteadas retornam 403.
> 2. **Apenas `foundation-model` no resource** — inference profiles usam ARN com account ID (`arn:aws:bedrock:<region>:<ACCOUNT_ID>:inference-profile/...`), diferente de foundation models (`arn:aws:bedrock:<region>::foundation-model/...`). Ambos precisam estar na policy.
> 3. **Model ID sem prefixo `us.`** — o model ID no Bedrock é `anthropic.claude-sonnet-4-6`, mas a invocação via cross-region usa `us.anthropic.claude-sonnet-4-6`. A policy deve cobrir o ARN do foundation model (sem `us.`).

### 2.3 Anexar a policy a todos os node roles

```bash
# Repetir para cada role distinto encontrado no passo 2.1
aws iam attach-role-policy \
  --role-name <NODE_ROLE_NAME> \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/BedrockInvokeClaudePolicy \
  --region <REGION>
```

> Esta operação é não destrutiva — não reinicia pods nem afeta aplicações em execução.
> A permissão IAM é avaliada em tempo real, sem necessidade de restart.

---

## 3. Configurar o ConfigMap

Os nomes das variáveis de ambiente devem seguir o padrão do `@ConfigurationProperties` do Spring Boot.
O prefix é `exception-intelligence`, portanto a propriedade `llm.provider` vira `EXCEPTION_INTELLIGENCE_LLM_PROVIDER`.

> **Erro comum:** usar nomes curtos como `EI_LLM_PROVIDER` — o Spring Boot não reconhece e usa o valor padrão (`claude`).

As variáveis relevantes para Bedrock no ConfigMap:

```yaml
EXCEPTION_INTELLIGENCE_LLM_PROVIDER: "bedrock"
EXCEPTION_INTELLIGENCE_LLM_BEDROCK_REGION: "us-east-1"
EXCEPTION_INTELLIGENCE_LLM_BEDROCK_MODEL_ID: "us.anthropic.claude-sonnet-4-6"
EXCEPTION_INTELLIGENCE_LLM_BEDROCK_MAX_TOKENS: "2048"
```

> **Importante:** o `MODEL_ID` deve usar o prefixo `us.` (cross-region inference profile).
> Usar o model ID direto (`anthropic.claude-sonnet-4-6`) retorna erro 400:
> _"Invocation with on-demand throughput isn't supported. Retry with an inference profile."_

O ConfigMap completo está em [configmap.yaml](configmap.yaml). Aplicar com:

```bash
kubectl apply -f k8s/server/configmap.yaml
```

---

## 4. Restart do deployment

```bash
kubectl rollout restart deployment/exception-intelligence-server -n exception-agent
kubectl rollout status deployment/exception-intelligence-server -n exception-agent
```

### Verificar se o Bedrock foi carregado

```bash
kubectl logs -n exception-agent deployment/exception-intelligence-server | grep "\[Config\]"
```

Saída esperada:
```
[Config] LLM provider: bedrock
[Config] Issue tracker: jira
[Config] Notification provider: teams
```

---

## 5. Testar

```bash
# Port-forward (caso não haja ingress configurado)
kubectl port-forward -n exception-agent svc/exception-intelligence-server 8090:8090

# Enviar um report de teste
curl -X POST http://localhost:8090/v1/exceptions \
  -H "Content-Type: application/json" \
  -d '{
    "language": "java",
    "framework": "spring-boot",
    "serviceName": "meu-servico",
    "environment": "production",
    "exception": {
      "type": "java.lang.NullPointerException",
      "message": "Cannot invoke method getName() on null object",
      "rawStackTrace": "java.lang.NullPointerException\n\tat com.example.UserService.getUser(UserService.java:42)",
      "frames": [
        {
          "file": "com/example/UserService.java",
          "function": "com.example.UserService.getUser",
          "line": 42,
          "isProjectCode": true
        }
      ]
    },
    "repository": {
      "owner": "your-github-org",
      "name": "your-repo",
      "branch": "main"
    }
  }'
```

Resposta esperada: **HTTP 202** (sem body). O processamento acontece em background — acompanhe pelos logs.

---

## Troubleshooting

| Erro no log | Causa | Solução |
|---|---|---|
| `LLM provider: claude` (ao invés de `bedrock`) | Variáveis de ambiente com nome errado (`EI_*`) | Usar `EXCEPTION_INTELLIGENCE_*` e reaplicar ConfigMap |
| `end of its life` (404) | Model ID descontinuado | Atualizar para modelo ACTIVE (listar com `aws bedrock list-foundation-models`) |
| `on-demand throughput isn't supported` (400) | Model ID sem prefixo de inference profile | Usar `us.anthropic.claude-sonnet-4-6` ao invés de `anthropic.claude-sonnet-4-6` |
| `not authorized to perform: bedrock:InvokeModel` (403) | Policy IAM ausente ou restritiva | Verificar se a policy cobre: regiões (`*`), inference-profile e foundation-model |
| 403 em região diferente (ex: `us-east-2`) | Policy com região fixa | Usar `*` na região — cross-region profiles roteiam entre regiões |
| 403 após restart (novo node) | Policy só em um node role | Anexar a policy a todos os node roles do cluster |
