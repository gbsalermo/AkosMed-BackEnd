# 16 — Relacionamentos JPA, Cascade e Fetch

> Referência obrigatória antes de implementar relações do AkosMed.  
> Modelo canônico do MVP: `04_ENTIDADES_E_RELACIONAMENTOS.md`.

---

# 1. Regra geral

Preferir relacionamentos explícitos, unidirecionais quando possível e controlados pelo Service.

Não usar:

```java
@ManyToMany
```

quando a relação possui ou pode possuir:

- campos próprios;
- status;
- data;
- configuração;
- regras de vínculo.

Por isso o AkosMed usa Entities de vínculo como:

```text
UsuarioUnidade
ProfissionalEspecialidade
ProfissionalUnidade
ProfissionalProcedimento
```

---

# 2. Public ID não muda as FKs

Toda Entity persistida usa:

```text
Long id       → PK/FK interna
UUID publicId → contrato externo
```

Relacionamentos JPA continuam usando a Entity/FK numérica.

Exemplo:

```text
Agendamento.profissional_id → BIGINT FK
AgendamentoCreateDTO.profissionalPublicId → UUID
```

O Service resolve `publicId + Tenant`, obtém a Entity e só então monta o relacionamento.

Não usar `publicId` como FK no MVP por padrão.

---

# 3. Fetch

## To-One

Preferir explicitamente:

```java
@ManyToOne(fetch = FetchType.LAZY)
@OneToOne(fetch = FetchType.LAZY)
```

quando viável.

## Collections

`@OneToMany` já é LAZY por padrão.

Não trocar para EAGER para “resolver” `LazyInitializationException`.

Se uma resposta precisa de dados relacionados, resolver com:

- transação adequada;
- query específica;
- fetch join/projection quando houver necessidade real;
- mapper dentro da fronteira correta.

Não serializar Entity diretamente.

---

# 4. Cascade

Não usar:

```java
cascade = CascadeType.ALL
```

sem justificativa explícita.

Pergunta obrigatória:

> Se eu salvar/remover A, B deve realmente ser salvo/removido automaticamente como parte do mesmo ciclo de vida?

Na maior parte do AkosMed:

```text
NÃO
```

Histórico clínico exige cuidado adicional.

---

# 5. Tenant → Unidade

```text
Tenant 1:N Unidade
```

Implementação prática:

```text
Unidade → Tenant
```

Não é obrigatório `Tenant` possuir `List<Unidade>`.

Cascade:

```text
nenhum
```

Nunca apagar Tenant e automaticamente apagar dados clínicos/históricos.

---

# 6. Pessoa

```text
Pessoa → Tenant
```

Tenant-scoped.

Sem cascade para Tenant.

Uma Pessoa pode ser referenciada por Paciente/Profissional/UsuarioTenant conforme regras do domínio.

---

# 7. Usuario

`Usuario` é credencial global.

Não pertence diretamente a um Tenant.

Relação com organização ocorre por:

```text
UsuarioTenant
```

Não mapear automaticamente coleções gigantes de vínculos se não houver necessidade de navegação.

---

# 8. UsuarioTenant

```text
UsuarioTenant → Usuario
UsuarioTenant → Tenant
UsuarioTenant → Pessoa
```

To-one LAZY.

Sem cascade por conveniência.

Service cria/resolve objetos na ordem correta e valida que `Pessoa` pertence ao mesmo Tenant do vínculo.

Constraints canônicas devem ser consultadas em `04_ENTIDADES_E_RELACIONAMENTOS.md`.

---

# 9. UsuarioUnidade

```text
UsuarioUnidade → UsuarioTenant
UsuarioUnidade → Unidade
```

Sem cascade.

Service valida:

```text
UsuarioTenant.tenant == Unidade.tenant
```

Não aceitar `unidadeId Long` do cliente; API usa `unidadePublicId`.

---

# 10. RefreshToken

Relações planejadas:

```text
RefreshToken → Usuario
RefreshToken → UsuarioTenant nullable
```

Para sessão tenant-scoped comum, a lógica exige `UsuarioTenant`.

Nullable existe apenas para possível sessão administrativa global de superAdmin conforme modelo canônico.

Sem cascade destrutivo.

---

# 11. Especialidade e Procedimento

Ambas são tenant-scoped:

```text
Especialidade → Tenant
Procedimento → Tenant
```

Sem cascade para Tenant.

Vínculos com Profissional são Entities próprias.

---

# 12. ProfissionalSaude

```text
ProfissionalSaude → Tenant
ProfissionalSaude → Pessoa
```

Relação lógica 1:1 entre Pessoa e Profissional dentro do Tenant.

Pode ser modelada com `@OneToOne(fetch = LAZY)` e unique constraint, desde que compatível com o modelo/DDL definitivo.

Sem cascade para Pessoa/Tenant por conveniência.

---

# 13. Paciente

```text
Paciente → Tenant
Paciente → Pessoa
```

Também é relação lógica 1:1 por Tenant.

Constraint canônica:

```text
unique(tenant_id, pessoa_id)
```

Sem cascade destrutivo.

---

# 14. ProfissionalEspecialidade

```text
ProfissionalEspecialidade → ProfissionalSaude
ProfissionalEspecialidade → Especialidade
```

Constraint:

```text
unique(profissional_id, especialidade_id)
```

Sem cascade.

Service valida mesmo Tenant.

---

# 15. ProfissionalUnidade

```text
ProfissionalUnidade → ProfissionalSaude
ProfissionalUnidade → Unidade
```

Constraint:

```text
unique(profissional_id, unidade_id)
```

Sem cascade.

O vínculo ativo é pré-condição para agenda naquela Unidade.

---

# 16. ProfissionalProcedimento

```text
ProfissionalProcedimento → ProfissionalSaude
ProfissionalProcedimento → Procedimento
```

Constraint:

```text
unique(profissional_id, procedimento_id)
```

Sem cascade.

Campos próprios como duração/valor override justificam não usar ManyToMany.

---

# 17. DisponibilidadeProfissional

```text
DisponibilidadeProfissional
→ Tenant
→ ProfissionalSaude
→ Unidade
```

`tenant_id` pode parecer redundante, mas é intencional para isolamento/queries.

Service deve garantir:

```text
Disponibilidade.tenant
== Profissional.tenant
== Unidade.tenant
```

Sem cascade.

---

# 18. BloqueioAgenda

```text
BloqueioAgenda
→ Tenant
→ ProfissionalSaude
→ Unidade
→ Usuario criadoPor, quando modelado como relação
```

No MVP, Unidade é obrigatória.

Sem cascade.

Inativar/cancelar conforme regra; não remover histórico por cascade.

---

# 19. Agendamento

```text
Agendamento
→ Tenant
→ Unidade
→ Paciente
→ ProfissionalSaude
→ Procedimento
→ Usuario criadoPor
```

Sem cascade.

Agendamento não cria automaticamente Paciente/Profissional/Procedimento/Unidade.

Service resolve cada `publicId` tenant-scoped.

Concorrência não é resolvida por cascade/relacionamento; consultar `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# 20. EventoAgendamento

```text
EventoAgendamento → Tenant
EventoAgendamento → Agendamento
EventoAgendamento → Usuario nullable
```

Histórico append-only.

Não é obrigatório mapear:

```text
Agendamento.getEventos()
```

Preferir consultar via Repository para evitar carregar histórico acidentalmente.

Sem orphanRemoval.

Sem cascade remove.

---

# 21. Prontuario

```text
Prontuario → Tenant
Prontuario → Paciente
```

Constraint:

```text
unique(tenant_id, paciente_id)
```

Sem cascade destrutivo.

Paciente não apaga Prontuário automaticamente.

---

# 22. Atendimento

```text
Atendimento
→ Tenant
→ Prontuario
→ Unidade
→ ProfissionalSaude
→ Agendamento nullable
```

Sem cascade.

Não duplicar `Paciente` em Atendimento; ele é derivado:

```text
Atendimento → Prontuario → Paciente
```

Quando `agendamento_id` existir, considerar/usar constraint UNIQUE conforme decisão canônica para impedir dois Atendimentos derivados do mesmo Agendamento.

---

# 23. EvolucaoClinica

```text
EvolucaoClinica
→ Tenant
→ Atendimento
→ ProfissionalSaude
→ EvolucaoClinica retificacaoDe nullable
```

Self-reference LAZY.

Sem orphanRemoval.

Sem cascade remove.

Retificação cria novo registro; original permanece.

---

# 24. Prescricao

```text
Prescricao
→ Tenant
→ Atendimento
→ ProfissionalSaude
```

Não duplicar Paciente; derivar:

```text
Prescricao → Atendimento → Prontuario → Paciente
```

Uma Prescrição possui itens.

Embora cascade/orphanRemoval possa parecer conveniente enquanto RASCUNHO, o projeto prefere inicialmente salvar/remover `ItemPrescricao` explicitamente pelo Service para manter o ciclo de vida claro.

Não usar cascade destrutivo depois da emissão.

---

# 25. ItemPrescricao

```text
ItemPrescricao → Prescricao
```

Sem necessidade de relação reversa se o código puder consultar via Repository.

Enquanto RASCUNHO:

- adicionar;
- editar;
- remover.

Depois de EMITIDA, o ciclo de vida fica protegido pela regra de Prescrição.

---

# 26. AnexoClinico

```text
AnexoClinico
→ Tenant
→ Prontuario
→ Atendimento nullable
→ Usuario uploadedBy, conforme modelo
```

Não duplicar Paciente; derivar do Prontuário.

Sem cascade.

Excluir/inativar metadata nunca remove Prontuário/Atendimento.

Arquivo real é tratado pelo `StorageService`, fora do banco.

---

# 27. Notificacao

Modelo canônico do Core:

```text
Notificacao
→ Tenant
→ UsuarioTenant
```

Campos operacionais incluem:

```text
categoria
titulo
mensagem
lida
createdAt
lidaEm
```

Essa relação deixa explícito que a notificação pertence ao contexto de um usuário dentro de um Tenant.

Não modelar automaticamente com `usuarioId` global se isso perder o contexto do Tenant.

Sem cascade.

Endpoints `/me` resolvem `UsuarioTenant` pelo contexto autenticado.

---

# 28. AuditLog

Aqui a decisão é diferente de um agregado de domínio.

Preferir IDs primitivos/valores simples:

```text
tenantId
unidadeId
usuarioId
usuarioTenantId
recursoId
```

Esses são **IDs internos persistidos na auditoria**, não contrato externo HTTP.

Não mapear dezenas de relações JPA.

Motivo:

- reduzir acoplamento;
- manter histórico mesmo após evolução da modelagem;
- evitar carregamento de grafos;
- preservar auditoria quando recursos mudarem de estado.

AuditLog pode armazenar `correlationId` quando aplicável.

---

# 29. OrientacaoPaciente — pós-MVP

Não implementar durante o Core.

Quando a ETAPA 15 chegar:

```text
OrientacaoPaciente
→ Tenant
→ Paciente
→ ProfissionalSaude
→ Atendimento nullable
```

Sem cascade destrutivo.

Service valida:

```text
mesmo Tenant
+
Atendimento compatível quando informado
+
autorização
```

Arquivo/mídia é acessado por `StorageService`; não há relacionamento JPA com binário.

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

# 30. Bidirecionalidade

Só criar relação bidirecional quando o código realmente precisa navegar nos dois sentidos.

Evitar coleções como:

```text
Tenant.getTodasAsPessoas()
Paciente.getTodosAgendamentos()
Prontuario.getTodosAtendimentos()
Profissional.getTodosAgendamentos()
```

quando Repository resolve melhor.

Benefícios:

- menos ciclos;
- menos N+1 acidental;
- menos coleções gigantes;
- menos risco de serialização recursiva;
- modelo mais simples.

---

# 31. orphanRemoval

Usar somente quando o filho não faz sentido sem o pai **e** a remoção física é compatível com o domínio.

Não usar em histórico:

- Agendamento;
- EventoAgendamento;
- Atendimento;
- EvolucaoClinica;
- Prontuario;
- Prescricao emitida;
- AuditLog.

Para itens de rascunho, ainda preferir controle explícito inicialmente.

---

# 32. `@Version`

`@Version` não é relacionamento/cascade, mas deve ser avaliado junto da Entity mutável.

Candidatas estão em `22_CONCORRENCIA_IDEMPOTENCIA_CLOCK_OPERACAO.md`.

Não aplicar por reflexo em históricos append-only.

---

# 33. Checklist JPA

Antes de concluir um relacionamento:

- [ ] quem possui a FK?
- [ ] `publicId` continua apenas contrato externo?
- [ ] Tenant de todos os lados foi validado?
- [ ] relação precisa ser bidirecional?
- [ ] fetch está LAZY?
- [ ] cascade é realmente necessário?
- [ ] orphanRemoval é seguro?
- [ ] existe unique constraint lógica?
- [ ] Service resolve relações por `publicId + Tenant`?
- [ ] JSON não serializa Entity?
- [ ] delete do pai não apaga histórico?
- [ ] `@Version` foi avaliado quando aplicável?

---

# Regra final

```text
DTO/URL → UUID publicId
Service → resolve Tenant e relacionamentos
JPA/FK → Long id interno
Cascade → somente quando ciclo de vida justificar
Histórico → preservado
```
