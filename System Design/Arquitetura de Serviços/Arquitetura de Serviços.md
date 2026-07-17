---
title: Arquitetura de Serviços
aliases:
  - Serviços
  - Microsserviços
tags:
  - system-design
  - moc
  - arquitetura
parte-de: "[[System Design]]"
created: 2026-07-16
status: em-expansao
---

# Arquitetura de Serviços

Notas sobre como dividir, integrar e evoluir aplicações sem perder clareza de domínio.

## Notas

- [[Monólito Modular]] - modularidade antes de distribuição.
- [[Microsserviços]] - quando dividir serviços, custos e sinais de maturidade.
- [[CQRS]] - separar modelo de escrita e leitura quando o padrão de acesso pede.
- [[Autenticação e Autorização em Sistemas Distribuídos]] - JWT, OAuth2/OIDC, RBAC, ABAC e identidade entre serviços.
- [[API Gateway]] - ponto de entrada, roteamento e políticas transversais.

## Pergunta guia

> [!question]
> Estou separando serviços porque existe uma fronteira real de negócio ou porque a tecnologia parece mais moderna?
