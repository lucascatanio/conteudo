---
title: Entrega e Operação
aliases:
  - Operação
  - Deploy
  - Delivery
tags:
  - system-design
  - moc
  - operacao
parte-de: "[[System Design]]"
created: 2026-07-16
status: em-expansao
---

# Entrega e Operação

Notas sobre colocar sistemas em produção, reduzir risco de mudança e diagnosticar problemas.

## Notas

- [[Deploy sem Downtime]] - rolling, blue-green, canary, feature flags e migrações de banco.
- [[Containers e Docker]] - Dockerfile multi-stage, imagem vs container, docker-compose.
- [[Kubernetes - Fundamentos]] - Pod, Deployment, Service, probes, e como isso já é rolling update por padrão.
- [[Infraestrutura como Código]] - por que declarar infra em código, Terraform, state, drift.
- [[AWS Well-Architected Framework]] - os 6 pilares, conectados às decisões já vistas no vault.
- [[Fundamentos - Observabilidade e Estudo de Caso]] - logs, métricas, traces e OpenTelemetry.
- [[ADR - Architecture Decision Records]] - registro de decisões arquiteturais.

**Prática correspondente no laboratório:** [[Exemplo prático - Containerização com Docker]] (nível Avançado) — Dockerfile real, build e run verificados, persistência via volume provada.

## Pergunta guia

> [!question]
> Se este deploy der errado, eu consigo perceber rápido, reduzir impacto e voltar para um estado saudável?
