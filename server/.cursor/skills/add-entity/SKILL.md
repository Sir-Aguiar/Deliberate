---
name: add-entity
description: Adds a new database entity to the Deliberate backend end to end (Domain class, EF Core configuration, migration, docs, tests). Use when the user asks to create, add, or model a new entity, table, or aggregate.
disable-model-invocation: true
---

# Add Entity

Fluxo para criar uma entidade nova de forma consistente com `.cursor/ENTITIES.md` e as regras em `.cursor/rules/`.

## Checklist

```
- [ ] 1. Especificar a entidade em ENTITIES.md
- [ ] 2. Criar a classe e enums no Domain
- [ ] 3. Criar a configuração do EF Core
- [ ] 4. Registrar no DbContext
- [ ] 5. Gerar e revisar a migration
- [ ] 6. Escrever testes
- [ ] 7. Validar (build + testes)
```

## 1. Especificar (antes de codar)

Se a entidade ainda não está em `.cursor/ENTITIES.md`, adicione primeiro: tabela, campos (tipo, regras), FKs, índices/únicos e invariantes; atualize o diagrama Mermaid e o "Resumo das tabelas". Se faltar informação de negócio, **pergunte** antes de assumir. Atualize `CONTEXT.md` se uma regra de negócio mudar.

Decida e registre:
- Criada pelo app offline? Então `Id` é `Guid` **gerado pelo cliente**, com `UpdatedAt` (delta sync).
- Dado financeiro? Então soft delete (`DeletedAt`/`DeletedByUserId`), sem cascade, e `ActivityLog` na escrita.
- Append-only (ex.: log, ajuste)? Sem update/delete.
- Precisa de papel? Defina quem pode criar/editar (`Member` vs `Owner`).

## 2. Domain (`Deliberate.Domain`)

- Uma classe por arquivo, `sealed`, em pasta por agregado (`Expenses/`, `Groups/`, `Settlements/`).
- Construtor/método de fábrica que valida invariantes; setters `private`/`init`; exceções `DomainException`.
- Dinheiro: `decimal`, sem `float`/`double`. Datas: `DateTimeOffset` (UTC) ou `DateOnly`.
- Enums em arquivo próprio. Sem referência a EF Core, ASP.NET ou atributos de mapeamento.

```csharp
public sealed class Settlement
{
    public Guid Id { get; private set; }
    public decimal Amount { get; private set; }
    public SettlementStatus Status { get; private set; }

    public static Settlement Report(Guid id, Guid groupId, Guid fromUserId, Guid toUserId, decimal amount, DateTimeOffset now)
    {
        if (amount <= 0) throw new DomainException("Settlement amount must be positive.");
        if (fromUserId == toUserId) throw new DomainException("Payer and receiver must differ.");
        // ...
    }
}
```

## 3. Configuração EF Core (`Deliberate.Infrastructure`)

Crie `<Entity>Configuration : IEntityTypeConfiguration<T>` em `Persistence/Configurations/`:
- `ToTable("snake_case_plural")`; `HasKey`; `HasPrecision(18, 2)` para dinheiro; `HasMaxLength` nos textos.
- Enums com `HasConversion<string>()`.
- FKs com `OnDelete(DeleteBehavior.Restrict)`.
- Índices e únicos (parciais com `HasFilter`) conforme `ENTITIES.md`.
- Filtro global `DeletedAt == null` se houver soft delete.

## 4. DbContext

Adicione `DbSet<T>` em `AppDbContext`. As configurations são aplicadas por `ApplyConfigurationsFromAssembly`; não registre manualmente.

## 5. Migration

```bash
dotnet ef migrations add Add<Entity> --project Deliberate.Infrastructure --startup-project Deliberate.API --output-dir Persistence/Migrations
```

Revise o SQL gerado (tipos, índices, FKs, `NULLS NOT DISTINCT` se necessário). Nunca edite migration já aplicada. Não rode `database update` em ambiente compartilhado sem confirmar.

## 6. Testes

- Domain: invariantes válidas e inválidas da entidade.
- Integração (Testcontainers/PostgreSQL): persistir, ler, e as restrições únicas/índices parciais.
- Se houver cálculo envolvido, siga `rules/testing.mdc`.

## 7. Validar

```bash
dotnet build && dotnet test
```

Corrija warnings novos. Conclua resumindo ao usuário: arquivos criados, migration gerada e pendências (ex.: serviço/endpoint, que seguem `rules/architecture-layers.mdc` e `rules/api-conventions.mdc`).

## Armadilhas comuns

- Esquecer de atualizar `ENTITIES.md` (fonte de verdade).
- Cascade delete em dado financeiro.
- `UpdatedAt` ausente em entidade sincronizável (quebra o delta sync).
- Escrever o `ActivityLog` fora da transação da operação.
