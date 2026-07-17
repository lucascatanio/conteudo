---
title: Webhooks - Webhook vs Polling vs Fila
tags:
  - desenvolvimento
  - backend
  - APIs
  - integração
  - arquitetura
parte-de: "[[Webhooks]]"
created: 2026-07-15
status: pronto
---

# Webhooks — Webhook vs Polling vs Fila

Quarta parte de [[Webhooks|Webhooks e Webhook Handlers]]. Continuação de [[Webhooks - Implementando um Handler]].

---

## Webhook x polling

### Polling

Seu sistema pergunta periodicamente "tem novidade?".

```mermaid
sequenceDiagram
    participant Sistema
    participant Externo as Sistema Externo

    loop a cada N segundos
        Sistema->>Externo: tem novidade?
        Externo-->>Sistema: não
    end
    Sistema->>Externo: tem novidade?
    Externo-->>Sistema: sim, aqui está
```

### Webhook

O outro sistema avisa quando acontece, sem seu sistema precisar perguntar.

```mermaid
sequenceDiagram
    participant Externo as Sistema Externo
    participant Sistema

    Note over Externo,Sistema: Sistema não faz nada até o evento acontecer
    Externo->>Sistema: POST /webhooks (evento aconteceu)
    Sistema-->>Externo: 200 OK
```

| Critério | Webhook | Polling |
|---|---|---|
| Iniciativa | Sistema externo | Seu sistema |
| Latência | Normalmente baixa | Depende do intervalo |
| Uso de recursos | Mais eficiente | Pode gerar consultas inúteis |
| Complexidade | Exige endpoint público e segurança | Pode ser mais simples |
| Falhas | Exige retentativas e idempotência | Exige controle de estado e paginação |
| Melhor uso | Eventos imprevisíveis | Reconciliação e sistemas sem webhook |

> [!note]
> Webhooks e polling podem ser usados juntos.
> O webhook entrega rapidez; o polling periódico faz reconciliação e corrige eventos perdidos.

---

## Webhook x fila de mensagens

Um webhook normalmente usa HTTP entre sistemas.

Uma fila usa infraestrutura de mensageria, como:

- RabbitMQ;
- Kafka;
- SQS;
- Pub/Sub;
- Redis Streams.

| Critério | Webhook | Fila |
|---|---|---|
| Comunicação | HTTP | Protocolo de mensageria |
| Exposição pública | Frequentemente necessária | Nem sempre |
| Acoplamento | Integração entre serviços | Comunicação interna ou distribuída |
| Entrega | Depende do provedor | Controlada pelo broker |
| Escalabilidade | Boa, com arquitetura adequada | Geralmente melhor para alto volume |

### Padrão comum

```mermaid
flowchart LR
    Ext["Serviço externo"] -->|webhook| Handler["Handler"]
    Handler -->|publica| Fila["Fila interna"]
    Fila --> W1["Worker"]
    Fila --> W2["Worker"]
```

O webhook recebe o evento. A fila distribui o processamento.

---

## Próxima nota

Veja quando evitar webhooks, checklist de produção e erros comuns em [[Webhooks - Boas Práticas, Erros Comuns e Checklist]].
