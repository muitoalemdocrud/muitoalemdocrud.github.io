---
layout: post
title: Unit of Work em uma Exchange de Criptomoedas
subtitle: Garantindo atomicidade no ledger financeiro com Clean Architecture
tags: [typescript, nestjs, postgresql, architecture, cleanarchitecture]
comments: true
---

***Opa pessoal! Tudo bom com vocês?***

No meu tempo livre tento criar aplicaçoes que me estimulam a aplicar boas praticas e arquitetura de software e o que venho trazer hoje é um conhecimento que ganhei faz um tempo, mas é forma elegante ao aplicar uma boa pratica para sistemas de grande porte, transações de banco de dados

Grandes bancos de dados possuem a capacidade de fazer alterações atomicas no banco, garantindo que caso ocorra algum erro no fluxo os dados alterados voltam para o seu estado atual, deixando o banco de dados seguro para proxima tentativa.

Usar transaçoes atomicas são uma das boas praticas para garantir a resiliencia do sistema, e hoje vou mostrar uma forma elegante para aplicar esse boa pratica, deixando isolado a infraestrutura e dominio.

De acordo com as boas praticas de arquitetura limpa, o dominio deve ser agnostico a tecnologia usada, ou seja, nao deve ter implementaçoes reais da tenologia dentro de um caso de uso.


## O problema que me fez buscar essa solução

Estou desenvolvendo um aplicação focada em criptomoedas, um exchange de cripto e ela tem um otimo fluxo de trabalho que estimula a usar as melhores praticas para um sistema desse tipo e trouxe um problema que normalmente utiliza transaçoes atomicas:

Imagine confirmar um depósito de Bitcoin na exchange. Para o sistema, isso não é uma query — são **três operações atômicas**:
1. Recebe o evento confirmação de deposito de um fila
2. Procura a transação financeira no banco de dados
3. Atualizar o status da transação para `CONFIRMED`
4. Criar um **débito** na tabela ledger (a exchange recebeu)
5. Criar um **crédito** no tabela ledger (o saldo do usuário aumentou)

Se a operação 3 funcionar mas a 4 falhar, o ledger fica desbalanceado. Se a 5 funcionar mas a 4 falhar, o saldo do usuário aumenta sem lastro. Em um sistema financeiro, isso é inaceitável.

Quem ja conhece transações atomicas ja sabe como funciona, mas como fazer esse fluxo isolado do dominio? Como deixar o dominio sabendo desse processo sem saber qual banco de dados ta sendo usado?

Ai chegamos no conceito de UnitOfWork.



## Arquitetura da API

A api usa **NestJS + TypeScript + PostgreSQL** (sem ORM) com Clean Architecture e Domain-Driven Design. Cada módulo de negócio tem suas próprias camadas:

```
src/
├── shared/                          ← domínio compartilhado
│   ├── unit-of-work.ts              ← a abstração
│   └── domain.error.ts
├── infrastructure/
│   └── database/
│       ├── database.service.ts      ← pool + funçoes de transação atomica
│       ├── unit-of-work.postgres.ts ← implementação concreta
└── modules/
    └── financial/
        ├── domain/                  ← entidades + interfaces Repository
        ├── application/             ← use cases
        ├── infrastructure/          ← implementações PostgreSQL
        └── presentation/            ← controllers + DTOs
```

A regra de dependência é clara: **dependências só apontam para dentro**. `domain/` nunca importa de `infrastructure/`. `application/` nunca importa de `presentation/`.

O Unit of Work vive nessa estrutura como uma ponte entre o domínio e a infraestrutura — sem acoplar um ao outro.

## Começando pela abstração (Domain)

Tudo começa com uma interface simples em `src/shared/unit-of-work.ts`:

```typescript
import { TransactionRepository } from '@/modules/financial/domain/transaction.repository';
import { LedgerEntryRepository } from '@/modules/financial/domain/ledger-entry.repository';

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

Antes de tudo, existe uma abstração para execução de queries em `src/infrastructure/database/query-executor.ts`:

```typescript
export abstract class QueryExecutor {
  abstract query<T extends QueryResultRow = QueryResultRow>(
    sql: string,
    params?: unknown[],
  ): Promise<QueryResult<T>>;
}
```

Tanto o `DatabaseService` (pool-level) implementam essa abstração. Isso permite que os repositórios sejam **agnósticos** — eles recebem um `QueryExecutor` como dependencia e não precisam saber se estão operando dentro de uma transação ou diretamente no pool.

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
4. A regra de dupla entrada (Σ débitos = Σ créditos) é garantida pelo atomicidade da transação

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

A implementação possui sim uma implementação mais trabalhosa, mas garante esse isolamento de fluxos onde arquitetura limpa é utilizada

Então é isso pessoal! Comente o que achou, já usaram Unit of Work em algum projeto? Como resolveram o problema de atomicidade?

## Referências
• Clean Architecture — Robert C. Martin
• Patterns of Enterprise Application Architecture — Martin Fowler (Unit of Work)
• PostgreSQL Documentation — Transactions
• NestJS Documentation — Dependency Injection
• Projeto utilizado no artigo: [MyBitcoin - API](https://github.com/paulohsilvavieira/mybitcoin-api)