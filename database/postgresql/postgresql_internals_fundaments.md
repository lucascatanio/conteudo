# PostgreSQL Internals - Fundamentos


***

## Índice

1. [Introdução](#introdu%C3%A7%C3%A3o)
2. [Índices no PostgreSQL](#%C3%ADndices-no-postgresql)
3. [EXPLAIN e EXPLAIN ANALYZE](#explain-e-explain-analyze)
4. [Query Planning e Otimização](#query-planning-e-otimiza%C3%A7%C3%A3o)
5. [MVCC e Dead Tuples](#mvcc-e-dead-tuples)
6. [Vacuum e Autovacuum](#vacuum-e-autovacuum)
7. [Bloqueio de Tabelas](#bloqueio-de-tabelas)
8. [Conceitos Importantes](#conceitos-importantes)
9. [Armadilhas Comuns](#armadilhas-comuns)
10. [Exercícios Práticos](#exerc%C3%ADcios-pr%C3%A1ticos)
11. [Checklist de Conhecimento](#checklist-de-conhecimento)
12. [Recursos para Aprofundamento](#recursos-para-aprofundamento)

***

## Introdução

### O que é PostgreSQL Internals?

PostgreSQL Internals refere-se ao funcionamento interno do banco de dados: como ele armazena dados fisicamente, planeja queries, gerencia índices, mantém consistência transacional e otimiza performance.

### Por que importa?

Em produção, 80% dos problemas de performance vêm de queries mal escritas ou falta de índices adequados. Um desenvolvedor júnior escreve SQL e espera que funcione. Um desenvolvedor senior entende:

- Por que uma query levou 3 segundos em vez de 30ms
- Quando adicionar um índice resolve (e quando piora)
- Como o Vacuum mantém o banco saudável
- Por que tabelas crescem e não encolhem sozinhas


### Arquitetura Básica

PostgreSQL usa **MVCC (Multi-Version Concurrency Control)** que permite:

- Leituras sem bloqueio
- Múltiplas versões da mesma linha simultaneamente
- Alta concorrência entre transações

***

## Índices no PostgreSQL

### O que são Índices?

Índices são estruturas de dados que aceleram buscas em tabelas, funcionando como um "índice de livro" que aponta para onde os dados estão armazenados fisicamente.

**Analogia:** Buscar um nome na agenda telefônica:

- **Sem índice (Seq Scan):** Ler página por página até achar
- **Com índice (Index Scan):** Ir direto na letra inicial


### Tipos de Índices

#### 1. B-tree (padrão)

**Características:**

- Estrutura de árvore balanceada
- Ideal para comparações: `=, <, >, <=, >=, BETWEEN, IN`
- Suporta ordenação: `ORDER BY`
- **Uso:** 99% dos casos (queries com WHERE, JOIN, ORDER BY)

**Exemplo:**

```sql
-- Criar índice B-tree (padrão)
CREATE INDEX idx_users_email ON users(email);

-- Query otimizada
SELECT * FROM users WHERE email = 'user@example.com';
-- Usa Index Scan
```


#### 2. Hash

**Características:**

- Apenas para igualdade (`=`)
- Menor que B-tree, mas limitado
- **Uso:** Raramente útil (B-tree geralmente é melhor)

**Exemplo:**

```sql
CREATE INDEX idx_users_email_hash ON users USING HASH(email);
```


#### 3. GiST (Generalized Search Tree)

**Características:**

- Para dados geométricos e full-text search
- Estrutura flexível para tipos customizados
- **Uso:** Dados geoespaciais (PostGIS), busca de texto

**Exemplo:**

```sql
-- Dados geoespaciais
CREATE INDEX idx_locations_point ON locations USING GIST(coordinates);
```


#### 4. GIN (Generalized Inverted Index)

**Características:**

- Para arrays, JSONB, full-text search
- Índice invertido (cada elemento vira entrada)
- **Uso:** Queries em colunas JSONB ou arrays

**Exemplo:**

```sql
-- Índice em JSONB
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    metadata JSONB
);

CREATE INDEX idx_products_metadata ON products USING GIN(metadata);

-- Query otimizada
SELECT * FROM products WHERE metadata @> '{"category": "electronics"}';
-- Usa GIN index
```


### Índices Compostos

Índices com múltiplas colunas. **A ordem importa!**

```sql
-- Índice composto
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- ✅ USA o índice (começa por user_id)
SELECT * FROM orders WHERE user_id = 123;

-- ✅ USA o índice (user_id + created_at)
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2026-01-01';

-- ❌ NÃO USA o índice (não começa por user_id)
SELECT * FROM orders WHERE created_at > '2026-01-01';
```

**Regra de ouro:** Coloque primeiro as colunas com maior **seletividade** (que filtram mais linhas).

### Índices Funcionais

Para queries com funções em WHERE:

```sql
-- Problema: Função não usa índice normal
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';
-- Seq Scan (lento)

-- Solução: Índice funcional
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- Agora usa Index Scan
```


### Quando NÃO criar índices

❌ **Evite índices em:**

- Colunas raramente usadas em WHERE/JOIN
- Colunas de baixa cardinalidade (ex: `gender` com apenas M/F)
- Tabelas pequenas (<1000 linhas)
- Colunas com muitos NULLs e poucas queries `IS NULL`

**Trade-off:** Cada índice custa em `INSERT/UPDATE/DELETE`.

```sql
-- Exemplo de impacto
-- SEM índices extras: 10k INSERTs/min
-- COM 5 índices extras: ~7k INSERTs/min (30% mais lento)
```


***

## EXPLAIN e EXPLAIN ANALYZE

### EXPLAIN - O Plano de Execução

Mostra **como** o PostgreSQL vai executar a query (sem executá-la):

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

**Output:**

```
Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=100)
  Index Cond: (email = 'test@example.com')
```


### EXPLAIN ANALYZE - A Verdade Real

Executa a query e mostra tempo real + estimativas:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

**Output:**

```
Index Scan using idx_users_email on users  
  (cost=0.42..8.44 rows=1 width=100) 
  (actual time=0.025..0.026 rows=1 loops=1)
Planning Time: 0.15 ms
Execution Time: 0.05 ms
```


### Interpretando o Output

**Tipos de Scan:**


| Tipo | Descrição | Performance |
| :-- | :-- | :-- |
| **Seq Scan** | Lê tabela inteira sequencialmente | ❌ Ruim para tabelas grandes |
| **Index Scan** | Usa índice para localizar dados | ✅ Bom |
| **Index Only Scan** | Dados vêm só do índice | ⭐ Excelente |
| **Bitmap Heap Scan** | Combina múltiplos índices | ✅ Bom para OR conditions |

**Métricas importantes:**

- **cost=0.42..8.44:** Estimativa de custo (startup..total). Números relativos, não absolutos.
- **rows=1:** Estimativa de linhas retornadas
- **actual time=0.025..0.026:** Tempo real em milissegundos
- **loops=1:** Quantas vezes o nó foi executado


### Quando usar cada um

- **EXPLAIN:** Query rápida para ver o plano (não executa)
- **EXPLAIN ANALYZE:** Debugging de performance (executa de verdade)

⚠️ **CUIDADO:** EXPLAIN ANALYZE executa a query de verdade!

```sql
-- ❌ NUNCA faça isso em produção sem WHERE!
EXPLAIN ANALYZE DELETE FROM users WHERE status = 'inactive';
-- Vai DELETAR de verdade!
```


### Exemplo Prático de Debug

```sql
-- Query lenta
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE user_id = 123;

-- Output mostra Seq Scan:
Seq Scan on orders (cost=0.00..18234.00 rows=1 width=200)
  (actual time=523.45..523.50 rows=1 loops=1)
Execution Time: 523.67 ms

-- Solução: Criar índice
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Testar novamente:
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE user_id = 123;

-- Agora usa Index Scan:
Index Scan using idx_orders_user_id on orders
  (cost=0.42..8.44 rows=1 width=200)
  (actual time=0.025..0.026 rows=1 loops=1)
Execution Time: 0.05 ms
-- 10.000x mais rápido!
```


***

## Query Planning e Otimização

### Como o PostgreSQL Decide o Plano

1. **Parse:** Analisa SQL sintaticamente
2. **Rewrite:** Aplica regras (views, etc.)
3. **Plan:** Calcula diferentes caminhos (usar índice X, Y, ou Seq Scan)
4. **Execute:** Executa o plano escolhido

### O Query Planner

O **Query Planner** usa **estatísticas** coletadas pelo comando `ANALYZE` para estimar custos e escolher o melhor plano.

### Atualizar Estatísticas

```sql
-- Atualizar estatísticas de uma tabela
ANALYZE users;

-- Atualizar estatísticas de todo o banco
ANALYZE;

-- Ver última vez que estatísticas foram atualizadas
SELECT relname, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'users';
```


### Quando o Planner Erra

**Sintoma:** EXPLAIN estima 10 rows, mas query retorna 1 milhão.

**Causas:**

1. Estatísticas desatualizadas
2. Statistics target muito baixo
3. Distribuição de dados não uniforme

**Soluções:**

```sql
-- Solução 1: ANALYZE
ANALYZE users;

-- Solução 2: Aumentar statistics target
ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;
ANALYZE users;

-- Solução 3: Verificar estatísticas
SELECT tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'users';
```


### Forçar Plano (apenas para debug!)

```sql
-- Forçar uso de índice
SET enable_seqscan = OFF;

-- Desabilitar nested loop
SET enable_nestloop = OFF;

-- Voltar ao padrão
RESET enable_seqscan;
```

⚠️ **Nunca use isso em produção!** É apenas para testes.

***

## MVCC e Dead Tuples

### O que é MVCC?

**Multi-Version Concurrency Control:** Mecanismo que permite múltiplas versões da mesma linha existirem simultaneamente, cada uma visível para diferentes transações.

### Por que usar MVCC?

**Objetivo principal:** Permitir **leituras sem bloqueio** (alta concorrência).

```sql
-- Transação 1: UPDATE (cria nova versão)
BEGIN;
UPDATE users SET email = 'novo@email.com' WHERE id = 1;
-- Ainda não fez COMMIT

-- Transação 2 (outra conexão): SELECT (lê versão antiga)
SELECT email FROM users WHERE id = 1;
-- Retorna 'antigo@email.com' (versão antiga ainda visível)
-- NÃO BLOQUEIA! ✅

-- Transação 1: COMMIT
COMMIT;

-- Agora Transação 2 vê a nova versão em queries futuras
```


### Transaction IDs

Cada transação recebe um **XID (Transaction ID)** único e crescente.

```sql
-- Ver transaction ID atual
SELECT txid_current();

-- Ver range de XIDs
SELECT datname, age(datfrozenxid) 
FROM pg_database;
```


### Dead Tuples (Linhas Mortas)

Quando você faz `UPDATE` ou `DELETE`, o PostgreSQL:

1. **Não apaga** a linha antiga do disco (por causa do MVCC)
2. **Cria uma nova versão** (no caso de UPDATE)
3. Marca a antiga como **"morta"** (dead tuple)
```sql
-- Estado inicial: Linha ocupa 100 bytes no disco
id | name  | email
1  | João  | joao@email.com

-- UPDATE
UPDATE users SET email = 'novo@email.com' WHERE id = 1;

-- Disco AGORA (fisicamente):
id | name  | email              | xmin | xmax | status
1  | João  | joao@email.com     | 100  | 101  | DEAD
1  | João  | novo@email.com     | 101  | 0    | LIVE
```

**Consequências:**

- Tabela ocupa mais espaço
- Seq Scan fica mais lento (lê linhas mortas)
- Índices ficam maiores (apontam para versões mortas)


### Verificar Dead Tuples

```sql
-- Ver linhas vivas vs mortas
SELECT 
    relname AS table_name,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_percentage
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_dead_tup DESC;

-- Ver tamanho físico da tabela
SELECT pg_size_pretty(pg_total_relation_size('users'));
```


***

## Vacuum e Autovacuum

### O Problema do Bloat

Com o tempo, dead tuples acumulam e causam:

- **Bloat:** Tabelas/índices ocupam mais espaço que necessário
- **Performance degradada:** Queries mais lentas
- **Desperdício de disco**


### O que é Vacuum?

Processo que:

1. Remove dead tuples
2. Libera espaço para reutilização (marca como disponível)
3. Atualiza estatísticas para o Query Planner
4. Previne **transaction ID wraparound** (crítico!)

### Tipos de Vacuum

#### VACUUM (normal)

```sql
-- Vacuum normal
VACUUM users;

-- Vacuum + atualizar estatísticas
VACUUM ANALYZE users;

-- Vacuum em todo o banco
VACUUM;
```

**Características:**

- ✅ Não bloqueia tabela (permite leituras e escritas)
- ✅ Libera espaço para reutilização
- ❌ Não devolve espaço ao sistema operacional
- ⏱️ Rápido (minutos para milhões de linhas)


#### VACUUM FULL

```sql
-- Vacuum Full (use com CUIDADO!)
VACUUM FULL users;
```

**Características:**

- ❌ Bloqueia tabela completamente (ACCESS EXCLUSIVE LOCK)
- ✅ Reescreve tabela do zero
- ✅ Devolve espaço ao sistema operacional
- ⏱️ Lento (pode levar horas)
- 💾 Precisa de espaço temporário (2x tamanho da tabela)

**Quando usar VACUUM FULL:**

- Tabela com >50% de bloat
- Fora do horário de produção
- Manutenção agendada

**Alternativa melhor:** `pg_repack` (reescreve sem downtime)

```bash
# Instalar extensão
CREATE EXTENSION pg_repack;

# Rodar repack (não bloqueia)
pg_repack -t users -d production_db
```


### Autovacuum - O Herói Silencioso

PostgreSQL roda `autovacuum` automaticamente em background.

#### Como Funciona

**Threshold (limite):**

```
vacuum roda quando:
linhas_modificadas >= autovacuum_vacuum_threshold + 
                      (autovacuum_vacuum_scale_factor × total_linhas)
```

**Configuração padrão:**

```sql
autovacuum_vacuum_threshold = 50 linhas
autovacuum_vacuum_scale_factor = 0.20 (20%)
```

**Exemplos:**

Tabela com **10.000 linhas:**

```
Threshold = 50 + (0.20 × 10.000) = 2.050 linhas
Vacuum roda após 2.050 UPDATEs/DELETEs
```

Tabela com **1.000.000 linhas:**

```
Threshold = 50 + (0.20 × 1.000.000) = 200.050 linhas
Vacuum roda após 200 mil modificações
```


#### Verificar Autovacuum

```sql
-- Ver configuração global
SHOW autovacuum;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_vacuum_threshold;

-- Ver last vacuum/analyze de uma tabela
SELECT 
    relname,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'users';
```


#### Ajustar Autovacuum por Tabela

Para tabelas críticas ou de alta rotatividade:

```sql
-- Configurar autovacuum mais agressivo
ALTER TABLE sessions SET (
  autovacuum_vacuum_scale_factor = 0.05,  -- 5% em vez de 20%
  autovacuum_analyze_scale_factor = 0.02, -- 2% para ANALYZE
  autovacuum_vacuum_cost_delay = 10       -- Mais rápido
);

-- Ver configuração específica da tabela
SELECT reloptions 
FROM pg_class 
WHERE relname = 'sessions';
```


### Quando NÃO Confiar no Autovacuum

#### Cenário 1: Cargas Batch Pesadas

**Problema:** Job noturno que atualiza 2 milhões de linhas. Autovacuum não acompanha.

**Solução:**

```java
@Scheduled(cron = "0 0 2 * * *")
public void archiveOldOrders() {
    // 1. Processar batch
    jdbcTemplate.batchUpdate(
        "UPDATE orders SET status = 'ARCHIVED' WHERE created_at < ?",
        batchArgs
    );
    
    // 2. Limpar imediatamente
    jdbcTemplate.execute("VACUUM ANALYZE orders");
}
```


#### Cenário 2: Transações Longas

Transações abertas há muito tempo **impedem vacuum de limpar** dead tuples.

```sql
-- Identificar transações antigas
SELECT 
    pid,
    age(backend_xid) AS xid_age,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY age(backend_xid) DESC;

-- Matar transação travada (cuidado!)
SELECT pg_terminate_backend(12345);  -- PID da transação
```


### Transaction ID Wraparound

PostgreSQL tem limite de ~2 bilhões de transaction IDs. Quando atinge, precisa **"congelar"** linhas antigas.

```sql
-- Ver idade das tabelas (em XIDs)
SELECT 
    relname,
    age(relfrozenxid) AS xid_age,
    pg_size_pretty(pg_total_relation_size(oid)) AS size
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;

-- Alerta: Se age > 200 milhões, precisa vacuum urgente!
```

**Prevenção:** Autovacuum cuida disso automaticamente, mas monitore!

***

## Bloqueio de Tabelas

### O que é Bloqueio (Lock)?

**Lock:** Mecanismo que impede conflitos entre transações concorrentes, controlando quem pode ler/escrever dados.

### Tipos de Locks

#### 1. Row-Level Locks (nível de linha)

**FOR UPDATE:**

```sql
BEGIN;
SELECT * FROM users WHERE id = 1 FOR UPDATE;
-- Bloqueia APENAS a linha id=1 para escrita
-- Outras linhas podem ser modificadas normalmente
UPDATE users SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

**FOR SHARE:**

```sql
SELECT * FROM users WHERE id = 1 FOR SHARE;
-- Bloqueia linha para escrita, mas permite leituras
```


#### 2. Table-Level Locks

| Lock Mode | Permite Leitura | Permite Escrita | Exemplo |
| :-- | :-- | :-- | :-- |
| **ACCESS SHARE** | ✅ | ✅ | SELECT |
| **ROW EXCLUSIVE** | ✅ | ❌ (outras escritas esperam) | INSERT, UPDATE, DELETE |
| **SHARE** | ✅ | ❌ | CREATE INDEX CONCURRENTLY |
| **ACCESS EXCLUSIVE** | ❌ | ❌ | VACUUM FULL, ALTER TABLE, DROP TABLE |

### VACUUM FULL - O Bloqueio Total

```sql
-- ❌ Bloqueia TUDO (leitura + escrita)
VACUUM FULL users;
```

**Efeito prático:**

```java
// Sua API durante VACUUM FULL:
userRepository.findByEmail(email);  
// ⏳ TIMEOUT após 30 segundos
// Usuários não conseguem logar
// Monitoring dispara alertas
// Potencial perda de receita
```

**Duração:** Pode levar horas em tabelas grandes!

### Deadlocks

**Deadlock:** Duas transações esperam uma pela outra infinitamente.

```sql
-- Transação 1
BEGIN;
UPDATE users SET balance = 100 WHERE id = 1;
-- Espera liberar id=2...
UPDATE users SET balance = 200 WHERE id = 2;

-- Transação 2 (simultaneamente)
BEGIN;
UPDATE users SET balance = 300 WHERE id = 2;
-- Espera liberar id=1...
UPDATE users SET balance = 400 WHERE id = 1;

-- ❌ DEADLOCK! PostgreSQL detecta e aborta uma transação
ERROR: deadlock detected
```

**Prevenção:**

1. Sempre acessar recursos na mesma ordem
2. Manter transações curtas
3. Usar `SELECT FOR UPDATE` com cuidado

### Monitorar Locks

```sql
-- Ver locks ativos
SELECT 
    pid,
    usename,
    pg_blocking_pids(pid) AS blocked_by,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Ver deadlocks no log
SHOW deadlock_timeout;  -- Padrão: 1s
```


***

## Conceitos Importantes

### 1. Cargas Batch Pesadas

**Definição:** Processar muitos registros de uma vez, geralmente fora do horário de pico.

**Exemplos:**

```java
// Batch insert - 1 milhão de linhas
jdbcTemplate.batchUpdate(
    "INSERT INTO products (name, price) VALUES (?, ?)",
    batchArgs
);

// Job noturno - processar 500 mil pedidos
@Scheduled(cron = "0 0 2 * * *")
public void processOrdersDaily() {
    jdbcTemplate.batchUpdate(
        "UPDATE orders SET status = 'ARCHIVED' WHERE created_at < ?",
        yesterday
    );
}
```

**Por que é "pesado":**

- Gera milhões de dead tuples rapidamente
- Autovacuum não acompanha
- Tabela incha em minutos

**Solução:** VACUUM manual após batch.

### 2. Thresholds (Limites)

**Definição:** Limite que dispara uma ação automática.

**Exemplos:**

**Autovacuum:**

```
Threshold = 50 + (0.20 × total_linhas)
```

**Circuit Breaker (Fase 3):**

```java
@CircuitBreaker(failureRateThreshold = 50)  // Abre se >50% erros
```

**Rate Limiting:**

```java
@RateLimiter(requestsPerSecond = 100)  // Máximo 100 req/s
```


### 3. Cache de Curta Duração

**Definição:** Dados que ficam válidos por **segundos/minutos**, não horas/dias.

**Exemplos práticos:**

✅ **Use Redis (curta duração):**

```java
// Session token (expira em 15 min)
redisTemplate.opsForValue().set(
    "session:" + sessionId, 
    userData, 
    15, TimeUnit.MINUTES
);

// Rate limiting (expira em 1 min)
redisTemplate.opsForValue().increment("rate:user:123");
redisTemplate.expire("rate:user:123", 60, TimeUnit.SECONDS);

// OTP code (expira em 5 min)
redisTemplate.opsForValue().set("otp:" + email, "123456", 5, TimeUnit.MINUTES);
```

❌ **Não use PostgreSQL para isso:**

```java
// MAL: Session tokens no PostgreSQL
// - 1000 INSERTs/segundo sobrecarrega banco
// - Precisa job para deletar expirados
// - Desperdício de recursos
```

**Por que Redis?**

- Velocidade: ~0.1ms vs ~5ms (PostgreSQL)
- Expiração automática (TTL nativo)
- Zero overhead em disco
- Projetado para dados temporários


### 4. Reescrever Fisicamente

**VACUUM normal:**

- Marca espaço como "disponível"
- NÃO libera para sistema operacional

**VACUUM FULL (reescreve fisicamente):**

1. Cria nova tabela
2. Copia apenas linhas vivas
3. Apaga tabela antiga
4. Renomeia nova
```
ANTES: [LIVE][DEAD][LIVE][DEAD][DEAD][LIVE] → 6 blocos
DEPOIS: [LIVE][LIVE][LIVE] → 3 blocos
```

**Trade-off:**

- ✅ Libera espaço de verdade
- ❌ Bloqueia tabela (downtime)
- ❌ Precisa 2x espaço temporário

***

## Armadilhas Comuns

### 1. Index Scan vs Seq Scan

PostgreSQL pode escolher **Seq Scan** mesmo com índice se:

- Tabela é pequena (<10k linhas)
- Query retorna >10-15% da tabela
- Estatísticas desatualizadas

```sql
-- Verificar se índice está sendo usado
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Se não estiver, verificar estatísticas
SELECT * FROM pg_stat_user_tables WHERE relname = 'users';

-- Atualizar
ANALYZE users;
```


### 2. Índices em Colunas de Baixa Cardinalidade

❌ **Evite:**

```sql
-- Índice inútil (apenas 2 valores distintos)
CREATE INDEX idx_users_gender ON users(gender);  -- M/F

-- PostgreSQL ignora e faz Seq Scan de qualquer forma
```

✅ **Exceção:** Índices parciais

```sql
-- Útil se queries filtram por valor raro
CREATE INDEX idx_users_premium ON users(is_premium) WHERE is_premium = true;
-- Apenas 5% dos usuários são premium
```


### 3. Missing Indexes em Foreign Keys

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id)
);

-- ❌ SEM índice em user_id
-- JOINs ficam lentos!

-- ✅ Criar índice
CREATE INDEX idx_orders_user_id ON orders(user_id);
```


### 4. SELECT * Desnecessário

```sql
-- ❌ MAL: Traz colunas não usadas
SELECT * FROM users WHERE email = 'test@example.com';

-- ✅ BOM: Index Only Scan possível
SELECT id, email FROM users WHERE email = 'test@example.com';
```


### 5. Função em WHERE sem Índice Funcional

```sql
-- ❌ Não usa índice
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- ✅ Criar índice funcional
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```


### 6. Transações Longas

```java
// ❌ MAL: Transação aberta por horas
@Transactional
public void processLargeDataset() {
    for (int i = 0; i < 1_000_000; i++) {
        // Processa linha por linha
        // Impede autovacuum de limpar dead tuples!
    }
}

// ✅ BOM: Processar em chunks
public void processLargeDataset() {
    int batchSize = 1000;
    for (int offset = 0; offset < 1_000_000; offset += batchSize) {
        processChunk(offset, batchSize);  // Transação curta
    }
}
```


### 7. VACUUM FULL em Produção

```sql
-- ❌ NUNCA em horário de pico!
VACUUM FULL users;  -- Bloqueia por horas

-- ✅ Use alternativas:
-- 1. VACUUM normal (não bloqueia)
VACUUM ANALYZE users;

-- 2. pg_repack (sem downtime)
pg_repack -t users -d mydb

-- 3. VACUUM FULL fora de horário de pico
-- Agendar para 3h da manhã
```


***

## Exercícios Práticos

### Exercício 1: Análise de Performance com EXPLAIN

**Objetivo:** Praticar EXPLAIN e otimização de queries.

```sql
-- 1. Criar tabela de produtos
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255),
    category VARCHAR(100),
    price DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Inserir 100k linhas
INSERT INTO products (name, category, price)
SELECT 
    'Product ' || i,
    CASE (i % 5)
        WHEN 0 THEN 'Electronics'
        WHEN 1 THEN 'Clothing'
        WHEN 2 THEN 'Books'
        WHEN 3 THEN 'Home'
        ELSE 'Sports'
    END,
    (random() * 1000)::DECIMAL(10, 2)
FROM generate_series(1, 100000) AS i;

-- 3. Query SEM índice
EXPLAIN ANALYZE
SELECT * FROM products WHERE category = 'Electronics';
-- Anote: Execution Time = ?ms

-- 4. Criar índice
CREATE INDEX idx_products_category ON products(category);

-- 5. Query COM índice
EXPLAIN ANALYZE
SELECT * FROM products WHERE category = 'Electronics';
-- Compare: Execution Time = ?ms

-- 6. Query com ordenação
EXPLAIN ANALYZE
SELECT * FROM products 
WHERE category = 'Electronics' 
ORDER BY created_at DESC 
LIMIT 10;

-- 7. Criar índice composto
CREATE INDEX idx_products_cat_date ON products(category, created_at DESC);

-- 8. Testar novamente
EXPLAIN ANALYZE
SELECT * FROM products 
WHERE category = 'Electronics' 
ORDER BY created_at DESC 
LIMIT 10;
```

**Perguntas:**

1. Qual a diferença de tempo entre queries com/sem índice?
2. Qual tipo de scan foi usado em cada caso?
3. O índice composto melhorou a ordenação?

***

### Exercício 2: Simulando Bloat e Vacuum

**Objetivo:** Entender dead tuples e vacuum na prática.

```sql
-- 1. Criar tabela
CREATE TABLE sessions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,
    token VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 2. Inserir 50k sessões
INSERT INTO sessions (user_id, token)
SELECT 
    (random() * 10000)::BIGINT,
    md5(random()::text)
FROM generate_series(1, 50000);

-- 3. Ver tamanho inicial
SELECT 
    pg_size_pretty(pg_total_relation_size('sessions')) AS size,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'sessions';

-- 4. Simular expiração (UPDATE massivo)
UPDATE sessions SET token = md5(random()::text);

-- 5. Ver dead tuples
SELECT 
    pg_size_pretty(pg_total_relation_size('sessions')) AS size,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'sessions';
-- Dead tuples = 50k!

-- 6. Rodar VACUUM
VACUUM ANALYZE sessions;

-- 7. Verificar limpeza
SELECT 
    pg_size_pretty(pg_total_relation_size('sessions')) AS size,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'sessions';
-- Dead tuples = 0

-- 8. Simular mais updates
UPDATE sessions SET token = md5(random()::text);
UPDATE sessions SET token = md5(random()::text);
UPDATE sessions SET token = md5(random()::text);

-- 9. Ver bloat severo
SELECT 
    pg_size_pretty(pg_total_relation_size('sessions')) AS size,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'sessions';

-- 10. VACUUM FULL (cuidado em produção!)
VACUUM FULL sessions;

-- 11. Ver espaço liberado
SELECT 
    pg_size_pretty(pg_total_relation_size('sessions')) AS size,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'sessions';
```

**Perguntas:**

1. Quanto o tamanho aumentou após os UPDATEs?
2. VACUUM normal liberou espaço para o SO?
3. Qual a diferença de tamanho após VACUUM FULL?

***

### Exercício 3: Configurar Autovacuum Customizado

**Objetivo:** Ajustar autovacuum para tabela de alta rotatividade.

```sql
-- 1. Ver configuração global
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_vacuum_threshold;

-- 2. Calcular threshold para sua tabela
-- Exemplo: 500k linhas
-- Threshold = 50 + (0.20 × 500000) = 100.050 linhas

-- 3. Ajustar para 5% (mais agressivo)
ALTER TABLE sessions SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_analyze_scale_factor = 0.02
);
-- Novo threshold = 50 + (0.05 × 500000) = 25.050 linhas

-- 4. Ver configuração específica
SELECT reloptions FROM pg_class WHERE relname = 'sessions';

-- 5. Monitorar autovacuum
SELECT 
    relname,
    last_vacuum,
    last_autovacuum,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'sessions';
```


***

### Exercício 4: Debug de Query Lenta (Real World)

**Cenário:** Endpoint de dashboard está lento.

```sql
-- Query problemática
EXPLAIN ANALYZE
SELECT 
    u.name,
    COUNT(o.id) AS total_orders,
    SUM(o.amount) AS total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id, u.name
ORDER BY total_spent DESC
LIMIT 10;
```

**Tarefas:**

1. Identificar qual parte está lenta (EXPLAIN ANALYZE)
2. Criar índices necessários
3. Comparar tempo antes/depois
4. Explicar por que os índices ajudaram

**Índices sugeridos:**

```sql
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_amount ON orders(amount);
```


***

## Checklist de Conhecimento

Use este checklist para validar seu conhecimento antes de avançar:

### Índices

- [ ] Sei os 4 tipos de índices e quando usar cada um
- [ ] Entendo diferença entre Index Scan e Index Only Scan
- [ ] Sei criar índices compostos com ordem correta
- [ ] Entendo quando NÃO criar índices
- [ ] Sei criar índices funcionais
- [ ] Entendo trade-off: velocidade de leitura vs escrita


### EXPLAIN

- [ ] Sei usar EXPLAIN e EXPLAIN ANALYZE
- [ ] Sei interpretar output (cost, rows, actual time)
- [ ] Identifico Seq Scan vs Index Scan
- [ ] Entendo quando PostgreSQL escolhe Seq Scan mesmo com índice
- [ ] Sei debugar queries lentas com EXPLAIN


### Query Planning

- [ ] Entendo como Query Planner funciona
- [ ] Sei atualizar estatísticas com ANALYZE
- [ ] Entendo quando estatísticas ficam desatualizadas
- [ ] Sei verificar último ANALYZE de uma tabela


### MVCC e Dead Tuples

- [ ] Entendo o que é MVCC e por que existe
- [ ] Sei explicar por que PostgreSQL não apaga linhas fisicamente
- [ ] Entendo o conceito de transaction IDs (XID)
- [ ] Sei verificar quantidade de dead tuples
- [ ] Entendo impacto de dead tuples em performance


### Vacuum

- [ ] Entendo diferença entre VACUUM e VACUUM FULL
- [ ] Sei quando usar cada um
- [ ] Entendo o que é bloat
- [ ] Sei verificar tamanho físico de tabelas
- [ ] Entendo trade-offs de VACUUM FULL
- [ ] Conheço alternativa pg_repack


### Autovacuum

- [ ] Entendo como autovacuum funciona
- [ ] Sei calcular threshold de autovacuum
- [ ] Sei ajustar autovacuum por tabela
- [ ] Entendo quando autovacuum não é suficiente
- [ ] Sei monitorar last_autovacuum
- [ ] Entendo impacto de transações longas em autovacuum


### Bloqueio e Locks

- [ ] Entendo diferença entre row-level e table-level locks
- [ ] Sei o que é ACCESS EXCLUSIVE LOCK
- [ ] Entendo impacto de VACUUM FULL em produção
- [ ] Sei identificar deadlocks
- [ ] Entendo SELECT FOR UPDATE
- [ ] Sei monitorar locks ativos


### Conceitos Avançados

- [ ] Entendo o que são cargas batch pesadas
- [ ] Sei o que são thresholds
- [ ] Entendo diferença entre cache curta e longa duração
- [ ] Sei quando usar Redis vs PostgreSQL
- [ ] Entendo o que é "reescrever fisicamente"
- [ ] Sei evitar armadilhas comuns (índices desnecessários, transações longas)


### Prática

- [ ] Fiz exercício de EXPLAIN na prática
- [ ] Simulei bloat e rodei VACUUM
- [ ] Configurei autovacuum customizado
- [ ] Debuguei query lenta real

***

## Recursos para Aprofundamento

### Livros Essenciais

1. **High Performance PostgreSQL** - Gregory Smith
    - Capítulos 5-8: Índices, VACUUM, Autovacuum
    - Nível: Intermediário
2. **PostgreSQL: Up and Running** - Regina Obe \& Leo Hsu
    - Capítulo 4: Performance Tuning
    - Nível: Iniciante/Intermediário
3. **Designing Data-Intensive Applications** - Martin Kleppmann
    - Capítulo 3: Storage and Retrieval (MVCC explicado)
    - Nível: Avançado

### Documentação Oficial

- [PostgreSQL Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [EXPLAIN Guide](https://www.postgresql.org/docs/current/using-explain.html)
- [Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [MVCC Explanation](https://www.postgresql.org/docs/current/mvcc-intro.html)


### Artigos e Tutoriais

- [Use The Index, Luke!](https://use-the-index-luke.com/) - Guia definitivo sobre índices
- [PostgreSQL MVCC Unmasked](https://momjian.us/main/writings/pgsql/mvcc.pdf) - Bruce Momjian
- [Explaining EXPLAIN](https://wiki.postgresql.org/wiki/Using_EXPLAIN)


### Ferramentas

- **pgAdmin:** Interface gráfica para PostgreSQL
- **pg_stat_statements:** Extensão para monitorar queries
- **pgBadger:** Análise de logs do PostgreSQL
- **pg_repack:** Repack sem downtime


### SQL para Monitoramento

```sql
-- Criar extensão pg_stat_statements
CREATE EXTENSION pg_stat_statements;

-- Ver queries mais lentas
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Ver tamanho das tabelas
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Ver índices não utilizados
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```


***

## Próximos Passos

Após dominar **PostgreSQL Internals - Fundamentos**, continue para:
1. **Transações e Isolamento**
    - ACID properties
    - Isolation levels (Read Committed, Repeatable Read, Serializable)
    - Deadlocks e como evitá-los
    - Row-level vs table-level locks
2. **Connection Pooling**
    - HikariCP configuration
    - Pool sizing strategy
    - Timeouts e connection leaks

### Validação de Conhecimento

Você está pronto para avançar quando consegue:

- ✅ Explicar diferença entre MVCC e locking tradicional
- ✅ Debugar uma query lenta usando EXPLAIN ANALYZE
- ✅ Decidir qual tipo de índice criar para uma query específica
- ✅ Calcular threshold de autovacuum
- ✅ Diagnosticar problema de bloat em produção
- ✅ Explicar trade-offs de VACUUM vs VACUUM FULL
- ✅ Configurar autovacuum customizado para tabela crítica
- ✅ Identificar quando usar Redis vs PostgreSQL

***

## Dicionário

| Termo | Definição |
| :-- | :-- |
| **Bloat** | Espaço desperdiçado por dead tuples |
| **Dead Tuple** | Linha marcada como apagada mas ainda no disco |
| **Index Scan** | Usar índice para localizar dados |
| **Seq Scan** | Ler tabela inteira sequencialmente |
| **MVCC** | Multi-Version Concurrency Control |
| **Query Planner** | Componente que decide como executar query |
| **Threshold** | Limite que dispara ação automática |
| **Transaction ID (XID)** | Identificador único de transação |
| **Vacuum** | Processo de limpeza de dead tuples |
| **Autovacuum** | Vacuum automático em background |
| **Lock** | Mecanismo de controle de concorrência |
| **Deadlock** | Duas transações esperando uma pela outra |


