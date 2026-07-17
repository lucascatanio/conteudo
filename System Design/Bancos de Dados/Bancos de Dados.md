---
title: Bancos de Dados
aliases:
  - Dados
  - Persistência
tags:
  - system-design
  - moc
  - banco-de-dados
parte-de: "[[System Design]]"
created: 2026-07-16
status: em-expansao
---

# Bancos de Dados

Notas sobre modelagem, escolha de banco, escala de leitura/escrita e trade-offs de persistência.

## Notas

- [[SQL, NoSQL e Quando Usar]] - comparação prática entre bancos relacionais e não-relacionais.
- [[SQL vs NoSQL]] - nota bibliográfica original sobre o tema.
- [[Read Replicas]] - escala de leitura e consistência eventual.
- [[Fundamentos - Cache, CDN e Banco de Dados]] - cache, CDN, índices e banco no contexto de system design.
- [[Fundamentos - Replicação, Sharding e Consistent Hashing]] - replicação, sharding e resharding.

## Pergunta guia

> [!question]
> O padrão de acesso pede relacionamentos e transações fortes, ou pede escala horizontal com consultas previsíveis?
