---
title: Infraestrutura como Código
aliases:
  - IaC
  - Terraform
tags:
  - system-design
  - iac
  - terraform
  - operacao
  - infraestrutura
parte-de: "[[Entrega e Operação]]"
created: 2026-07-16
status: pronto
---

# Infraestrutura como Código

> [!abstract] Em uma frase
> Infraestrutura como Código (IaC) é declarar em arquivo versionado o estado desejado da infraestrutura (servidores, redes, bancos, permissões) e deixar uma ferramenta calcular e aplicar a diferença — em vez de alguém clicar num console cloud e ninguém mais saber exatamente o que foi clicado.

> [!warning] Não executado neste ambiente
> Terraform não está instalado no ambiente onde este material foi escrito, e não há conta cloud disponível para aplicar de verdade. O HCL abaixo é sintaticamente correto e segue o padrão real do provider AWS, mas **não foi rodado** (`terraform plan`/`apply`). Trate como referência de estrutura.

## Por que declarar em vez de clicar

- **Revisão via PR.** Uma mudança de infraestrutura vira diff de código, revisável como qualquer outro PR — em vez de uma mudança de console que só existe no histórico de auditoria da cloud (se existir).
- **Idempotência.** Aplicar o mesmo arquivo duas vezes não duplica recursos — a ferramenta compara o estado desejado (código) com o estado atual e só aplica a diferença.
- **Reprodutibilidade.** Recriar o ambiente de staging igual ao de produção é rodar o mesmo código com variáveis diferentes, não repetir de memória uma sequência de cliques.
- **Drift detection.** Se alguém mudar algo manualmente no console, a próxima execução do plano mostra a diferença entre o que o código diz e o que existe de fato — o "drift".

## O ponto mais perigoso: state

Terraform mantém um arquivo de **state** que mapeia "este recurso no código corresponde a este recurso real na cloud, com este ID". Sem o state, o Terraform não sabe se um `resource` no código já existe ou precisa ser criado.

Dois riscos concretos de equipe:

1. **State local no laptop de alguém.** Se só uma pessoa tem o arquivo de state, ninguém mais consegue aplicar mudanças com segurança — o Terraform de outra máquina não sabe o que já existe e tentaria recriar tudo.
2. **Duas pessoas aplicando ao mesmo tempo.** Sem lock de state (normalmente um bucket S3 + tabela DynamoDB para lock, no caso da AWS), duas aplicações concorrentes corrompem o state.

A prática padrão é **state remoto com lock** — nunca commitar o state no Git (ele pode conter segredos em texto plano) e nunca deixá-lo só numa máquina local.

```hcl
terraform {
  backend "s3" {
    bucket         = "minha-empresa-terraform-state"
    key            = "producao/rede/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

## Exemplo ilustrativo: bucket S3 + role IAM

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "recibos_pagamento" {
  bucket = "minha-empresa-recibos-pagamento"
}

resource "aws_s3_bucket_versioning" "recibos_pagamento" {
  bucket = aws_s3_bucket.recibos_pagamento.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_iam_role" "api_pagamentos" {
  name = "api-pagamentos-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "acesso_recibos" {
  name = "acesso-bucket-recibos"
  role = aws_iam_role.api_pagamentos.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:PutObject", "s3:GetObject"]
      Resource = "${aws_s3_bucket.recibos_pagamento.arn}/*"
    }]
  })
}
```

Este trecho conecta diretamente com o desafio extra do [[Mini-projeto - Webhook Handler de Pagamentos]] (subir recibo em PDF pro S3) — a role IAM acima é exatamente o que a API precisaria em produção para ter permissão de escrever no bucket, sem usar credenciais de usuário fixas.

## `plan` antes de `apply` — o hábito que evita surpresa

```bash
terraform plan   # mostra o que MUDARIA, sem aplicar nada
terraform apply  # aplica, pedindo confirmação por padrão
```

`plan` é o equivalente de um `dry-run`: mostra exatamente quais recursos seriam criados, alterados ou **destruídos**. Aplicar sem revisar o plano — especialmente em produção — é a causa mais comum de "IaC apagou algo que não devia" (um recurso renomeado no código, por exemplo, aparece no plano como "destruir o antigo, criar um novo com outro nome", não como "renomear").

## Alternativa: IaC com linguagem de propósito geral

Terraform usa HCL, uma linguagem declarativa própria. Para quem já é .NET, o **Pulumi** oferece a mesma ideia (state, plan/apply, providers para AWS/Azure/GCP) mas com um SDK em C# — a infraestrutura vira uma classe, com loop, condicional e reuso de código como qualquer outro projeto .NET, em vez de uma DSL separada. AWS CDK tem suporte a C# na mesma linha. A troca é: ganha-se familiaridade de linguagem e testabilidade (dá pra escrever teste unitário da definição de infraestrutura); perde-se a portabilidade de HCL como padrão de mercado mais universal entre equipes de infra.

## Checklist

- [ ] State remoto com lock (nunca local, nunca commitado no Git).
- [ ] `terraform plan` revisado antes de todo `apply` em produção.
- [ ] Segredos (senhas, chaves) nunca em texto plano no `.tf` — usar um secret manager e referenciar por ARN/id.
- [ ] Mudança de infraestrutura passa por PR revisado, igual a mudança de código de aplicação.

## Notas relacionadas

- [[AWS Well-Architected Framework]]
- [[Containers e Docker]]
- [[Deploy sem Downtime]]
