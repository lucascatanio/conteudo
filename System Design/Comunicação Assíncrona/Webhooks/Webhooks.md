---
title: Webhooks e Webhook Handlers
aliases:
  - Webhook
  - Webhook Handler
tags:
  - desenvolvimento
  - backend
  - APIs
  - integração
  - arquitetura
  - MOC
created: 2026-07-15
status: pronto
---

# Webhooks e Webhook Handlers

> [!abstract] Em uma frase
> Um **webhook** é uma notificação HTTP enviada automaticamente por um sistema quando um evento acontece.
> Um **webhook handler** é o código que recebe, valida e processa essa notificação.

Esta nota é o índice do tema. O conteúdo completo foi dividido em notas menores para facilitar o estudo e a revisão:

1. [[Webhooks - Conceitos e Fluxo]] — o que é um webhook, webhook x webhook handler, fluxo completo e quando usar.
2. [[Webhooks - Casos de Uso Reais]] — pagamentos, e-commerce, GitHub/CI-CD, mensageria, CRM, autenticação, logística, processamento de arquivos/vídeo/IA.
3. [[Webhooks - Implementando um Handler]] — exemplos de código (Node/Express) e as responsabilidades de um bom handler (autenticidade, idempotência, resposta rápida, retentativas, logging, eventos desconhecidos).
4. [[Webhooks - Webhook vs Polling vs Fila]] — comparativos com polling e com filas de mensagens.
5. [[Webhooks - Boas Práticas, Erros Comuns e Checklist]] — quando não usar, modelo mental de decisão, checklist de produção, erros comuns, estrutura de código sugerida e como testar.

---

## Resumo

> [!summary]
> - **Webhook:** a notificação HTTP enviada por um sistema.
> - **Endpoint:** a URL que recebe a notificação.
> - **Webhook handler:** o código que valida e trata o evento.
> - **Idempotência:** impede que o mesmo evento cause efeitos duplicados.
> - **Assinatura:** confirma que a requisição veio do provedor esperado.
> - **Fila:** permite responder rápido e processar tarefas pesadas depois.
> - **Polling:** pode complementar webhooks para reconciliação.

## Exemplo mental final

```text
Evento externo:
Pagamento aprovado

Webhook:
POST /webhooks/payments

Webhook handler:
Valida assinatura
Verifica event_id
Registra o evento
Coloca tarefa na fila
Responde 200

Worker:
Marca pedido como pago
Emite nota
Envia e-mail
Atualiza o ERP
```

---

## Notas relacionadas

- [[APIs REST]]
- [[Arquitetura Orientada a Eventos]]
- [[Filas e Mensageria|Filas de Mensagens]]
- [[Idempotência]]
- [[HMAC]]
- [[Retries e Backoff]]
- [[Fundamentos - Observabilidade e Estudo de Caso|Observabilidade]]
- [[Integrações entre Sistemas]]
