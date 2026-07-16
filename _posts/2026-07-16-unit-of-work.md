---
layout: post
title: Unit of Work em uma Exchange de Criptomoedas
subtitle: Garantindo atomicidade no ledger financeiro com Clean Architecture
tags: [typescript, nestjs, postgresql, architecture, cleanarchitecture]
comments: true
---

***Opa pessoal! Tudo bom com vocês?***

No meu tempo livre, gosto de criar aplicações que me desafiem a aplicar boas práticas e conceitos de arquitetura de software. O que venho trazer hoje é um conhecimento que adquiri há algum tempo: uma forma elegante de aplicar uma boa prática em sistemas de grande porte utilizando transações de banco de dados.

Grandes bancos de dados possuem a capacidade de realizar alterações atômicas, garantindo que, caso ocorra algum erro durante o fluxo, todas as alterações realizadas sejam desfeitas, retornando o banco de dados ao seu estado anterior. Isso torna a aplicação muito mais segura para uma nova tentativa da operação.

Utilizar transações atômicas é uma das melhores práticas para garantir a resiliência de um sistema. Neste artigo, vou mostrar uma forma elegante de aplicar essa boa prática, mantendo a infraestrutura isolada do domínio.

De acordo com os princípios da Clean Architecture, o domínio deve ser agnóstico à tecnologia utilizada. Ou seja, ele não deve possuir implementações concretas de infraestrutura dentro de uma regra de negócio.

## O problema que me fez buscar essa solução

Estou desenvolvendo uma aplicação focada em criptomoedas, uma exchange de criptoativos. Ela possui um fluxo de trabalho extremamente interessante, que incentiva o uso de diversas boas práticas utilizadas em sistemas financeiros.

Durante o desenvolvimento, surgiu um problema bastante comum, que normalmente é resolvido através de transações atômicas.

Imagine a confirmação de um depósito de Bitcoin na exchange. Para o sistema, isso não representa apenas uma query, mas sim **cinco operações que precisam acontecer de forma atômica**:

1. Recebe o evento de confirmação de depósito de uma fila
2. Procura a transação financeira no banco de dados
3. Atualiza o status da transação para `CONFIRMED`
4. Cria um **débito** na tabela ledger (a exchange recebeu)
5. Cria um **crédito** na tabela ledger (o saldo do usuário aumentou)

Se a operação **3** funcionar, mas a **4** falhar, o ledger ficará desbalanceado.

Se a operação **5** funcionar, mas a **4** falhar, o saldo do usuário aumentará sem qualquer lastro financeiro.

Quem já conhece transações atômicas provavelmente sabe como resolver esse problema utilizando uma transação do banco de dados. Mas surgiu outra dúvida:

*Como fazer isso mantendo o domínio completamente isolado da infraestrutura?*

Ou ainda:

*Como permitir que o domínio saiba que está executando uma operação transacional sem conhecer qual banco de dados está sendo utilizado?*

Após alguns momentos de estudo cheguei ao conceito de Unit of Work.

## Arquitetura da API


A API utiliza **NestJS + TypeScript + PostgreSQL** (sem ORM), seguindo os princípios da **Clean Architecture** e do **Domain-Driven Design (DDD)**.

Cada módulo de negócio possui suas próprias camadas:



```text
src/
├── shared/                          ← domínio compartilhado
│   ├── unit-of-work.ts              ← abstração
│   └── domain.error.ts
├── infrastructure/
│   └── database/
│       ├── database.service.ts      ← pool + funções de transação
│       ├── unit-of-work.postgres.ts ← implementação concreta
└── modules/
    └── financial/
        ├── domain/                  ← entidades + interfaces Repository
        ├── application/             ← casos de uso
        ├── infrastructure/          ← implementações PostgreSQL
        └── presentation/            ← controllers + DTOs
```

A regra de dependência é clara:

As dependências sempre apontam para dentro da aplicação.

Isso significa que:

- `domain/` nunca importa nada de `infrastructure/`;
- `application/` nunca importa nada de `presentation/`.

O **Unit of Work** ocupa exatamente esse espaço entre domínio e infraestrutura, funcionando como uma ponte entre ambos, sem gerar acoplamento.

---

## Começando pela abstração (Domain)

Tudo começa com uma classe abstrata simples em:

```typescript

export interface Repositories {
  transactionRepo: TransactionRepository;
  ledgerRepo: LedgerEntryRepository;
}

export abstract class UnitOfWork {
  abstract run<T>(fn: (repos: Repositories) => Promise<T>): Promise<T>;
}
```

**Por que uma classe abstrata e não uma interface?** Porque o NestJS consegue usar classes abstratas como tokens de injeção. `UnitOfWork` vira o token — o use case não precisa saber qual é a implementação.

**Por que `Repositories` é uma interface fixa e não um `Map<string, unknown>`?** Porque ganhamos type safety gratuito. Quando você desestrutura `{ transactionRepo, ledgerRepo }`, o TypeScript já sabe o tipo de cada um. Sem casting, sem `as`, sem erros em runtime.

## O baixo nível da infraestrutura

### QueryExecutor — a base


```typescript
export abstract class QueryExecutor {
  abstract query<T extends QueryResultRow = QueryResultRow>(
    sql: string,
    params?: unknown[],
  ): Promise<QueryResult<T>>;
}
```

Tanto o `DatabaseService` (pool-level) implementam essa abstração. Isso permite que os repositórios sejam **agnósticos** — eles recebem um `QueryExecutor` como dependência e não precisam saber se estão operando dentro de uma transação ou diretamente no pool.

### DatabaseService — o pool

```typescript

import { Inject, Injectable } from '@nestjs/common';

import { Pool, QueryResult, QueryResultRow } from 'pg';

import { POOL_TOKEN } from '@/infrastructure/database/database.token';
import { QueryExecutor } from '@/infrastructure/database/query-executor';

@Injectable()
export class DatabaseService implements QueryExecutor {
  constructor(@Inject(POOL_TOKEN) private readonly pool: Pool) {}

  query<T extends QueryResultRow = QueryResultRow>(
    sql: string,
    params?: unknown[],
  ): Promise<QueryResult<T>> {
    return this.pool.query<T>(sql, params);
  }

  async runInTransaction<T>(fn: (tx: QueryExecutor) => Promise<T>): Promise<T> {
    const client = await this.pool.connect();
    await client.query('BEGIN');
    const tx: QueryExecutor = {
      query: (sql, params) => client.query(sql, params),
    };
    try {
      const result = await fn(tx);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}

```

O padrão é clássico: `BEGIN` → executar callback → `COMMIT` se OK, `ROLLBACK` se erro → sempre `release()` o client no `finally`.


## A cola entre Domain e Infrastructure

O `PostgresUnitOfWork` é onde tudo se conecta em `src/infrastructure/database/unit-of-work-postgres.service.ts`:

```typescript
@Injectable()
export class PostgresUnitOfWork extends UnitOfWork {
  constructor(private readonly databaseService: DatabaseService) {
    super();
  }

  async run<T>(fn: (repos: Repositories) => Promise<T>): Promise<T> {
    return this.databaseService.runInTransaction(async (tx) => {
      const repositories: Repositories = {
        transactionRepo: new PgTransactionRepository(tx),
        ledgerRepo: new PgLedgerEntryRepository(tx),
      };
      return fn(repositories);
    });
  }
}
```

Dentro de `runInTransaction`, todos os repositórios compartilham o **mesmo `PoolClient`** — ou seja, a mesma transação PostgreSQL. Se qualquer query dentro do callback falhar, o `ROLLBACK` garante que nada foi persistido.

**No módulo NestJS**, a injeção é direta:

```typescript
{
  provide: UnitOfWork,
  useClass: PostgresUnitOfWork,
}
```

Classe abstrata como token. Simples.

## O use case (Application)

Agora vamos ver como usar o UnitOfWork. Veja o `ConfirmDepositWithUowUseCase`:

```typescript
export class ConfirmDepositWithUowUseCase {
  constructor(private readonly uow: UnitOfWork) {}

  async execute(input: ConfirmDepositInputDTO): Promise<void> {
    await this.uow.run(async ({ transactionRepo, ledgerRepo }) => {
      const transaction = await transactionRepo.findById(input.transactionId);
      if (!transaction) {
        throw new TransactionNotFoundError(input.transactionId);
      }

      transaction.confirm();

      const debit = LedgerEntry.create({
        transactionId: transaction.id,
        account: `EXCHANGE:TREASURY:${transaction.type.toUpperCase()}`,
        type: 'debit',
        amountSatoshi: transaction.amountSatoshi,
      });

      const credit = LedgerEntry.create({
        transactionId: transaction.id,
        account: `USER:${transaction.accountId}:${transaction.type.toUpperCase()}`,
        type: 'credit',
        amountSatoshi: transaction.amountSatoshi,
      });

      await transactionRepo.save(transaction);
      await ledgerRepo.save(debit);
      await ledgerRepo.save(credit);
    });
  }
}
```

Olha o que acontece aqui:

1. O use case **não sabe** que está dentro de uma transação PostgreSQL
2. Os repositórios são **tipados automaticamente** — `transactionRepo` é `TransactionRepository`, `ledgerRepo` é `LedgerEntryRepository`
3. Se `ledgerRepo.save(credit)` falhar, **todas as operações** são revertidas
4. A regra de dupla entrada (Σ débitos = Σ créditos) é garantida pela atomicidade da transação

O use case injeta `UnitOfWork` (a abstração), não `DatabaseService` (a infraestrutura). Isso satisfaz a regra de dependência da Clean Architecture.


## Testabilidade

Outra vantagem que eu não esperava: o use case é trivial de testar. Basta mockar `UnitOfWork`:

```typescript
const mockUow = {
  run: jest.fn(async (fn) => fn({
    transactionRepo: mockTransactionRepo,
    ledgerRepo: mockLedgerRepo,
  })),
};

const useCase = new ConfirmDepositWithUowUseCase(mockUow);
```

Sem pool de conexão, sem transação real, sem PostgreSQL rodando. O teste valida a lógica de negócio — a infraestrutura é responsabilidade do `PostgresUnitOfWork`.

## Conclusão

O Unit of Work resolveu um problema real que eu tinha no meu projeto: **garantir que operações de múltiplas tabelas sejam atômicas**. Mas ele também resolve um problema arquitetural: manter a Clean Architecture íntegra enquanto faz isso.

A implementação possui sim uma construção mais trabalhosa, mas garante esse isolamento de fluxos em que a arquitetura limpa é utilizada

Então é isso pessoal! Comente o que achou, já usaram Unit of Work em algum projeto? Como resolveram o problema de atomicidade?

## Referências

- Clean Architecture — Robert C. Martin
- Patterns of Enterprise Application Architecture — Martin Fowler (Unit of Work)
- PostgreSQL Documentation — Transactions
- NestJS Documentation — Dependency Injection
- Projeto utilizado no artigo: [MyBitcoin - API](https://github.com/paulohsilvavieira/mybitcoin-api)