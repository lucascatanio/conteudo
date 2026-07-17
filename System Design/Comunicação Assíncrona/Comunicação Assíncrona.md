---
title: Comunicação Assíncrona
aliases:
  - Arquitetura Assíncrona
  - Mensageria
tags:
  - system-design
  - moc
  - comunicacao-assincrona
parte-de: "[[System Design]]"
created: 2026-07-16
status: em-expansao
---

# Comunicação Assíncrona

Comunicação assíncrona entra quando um sistema não precisa, ou não deve, esperar tudo acontecer dentro da mesma requisição.

## Notas

- [[Webhooks]] - eventos enviados via HTTP por sistemas externos.
- [[Filas e Mensageria]] - brokers, filas, tópicos, consumidores, retries e DLQ.
- [[Arquitetura Orientada a Eventos]] - eventos de domínio, eventos de integração e acoplamento temporal.
- [[Outbox e Inbox Pattern]] - consistência entre banco e mensageria.
- [[Sagas e Transações Distribuídas]] - coordenação de fluxos longos entre serviços.

## Pergunta guia

> [!question]
> O usuário precisa da resposta agora ou o sistema só precisa garantir que o trabalho será processado com segurança depois?
