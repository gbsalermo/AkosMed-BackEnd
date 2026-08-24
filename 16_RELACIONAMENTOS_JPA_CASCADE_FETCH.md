# 16 — Relacionamentos JPA, Cascade e Fetch

Este é um dos documentos mais importantes para evitar bugs de modelagem.

---

# 1. Regra geral

Preferir relacionamentos explícitos e controlados.

Não usar:

```java
@ManyToMany
```

quando a relação possui:

- campos próprios;
- status;
- data;
- configurações.

Por isso o AkosMed usa entidades de vínculo.

---

# 2. Fetch

## To-One

Mesmo que JPA tenha defaults diferentes, no projeto preferir explicitamente:

```java
@ManyToOne(fetch = FetchType.LAZY)
@OneToOne(fetch = FetchType.LAZY)
```

quando viável.

## Collections

```java
@OneToMany
```

já são LAZY por padrão.

Não trocar para EAGER para “resolver erro”.

---

# 3. Cascade

Não usar:

```java
cascade = CascadeType.ALL
```

sem motivo.

Pergunta antes de adicionar cascade:

> se eu salvar/apagar A, B deve ser automaticamente salvo/apagado?

Na maioria das relações do AkosMed:

```text
NÃO
```

---

# 4. Relação Tenant → Unidade

```text
Tenant 1:N Unidade
```

Implementação prática:

`Unidade` possui FK para Tenant.

Não é obrigatório Tenant possuir `List<Unidade>`.

Para simplificar, pode manter relação unidirecional:

```java
Unidade -> Tenant
```

Isso reduz coleções e ciclos.

Cascade recomendado:

```text
nenhum
```

Nunca apagar Tenant e automaticamente apagar histórico.

---

# 5. Pessoa

```text
Pessoa -> Tenant
```

Pessoa tenant-scoped.

Sem cascade para Tenant.

---

# 6. UsuarioTenant

```text
UsuarioTenant -> Usuario
UsuarioTenant -> Tenant
UsuarioTenant -> Pessoa
```

Todos `ManyToOne LAZY`.

Sem cascade.

Service cria os objetos na ordem correta.

---

# 7. UsuarioUnidade

```text
UsuarioUnidade -> UsuarioTenant
UsuarioUnidade -> Unidade
```

Sem cascade.

Service valida mesmo tenant.

---

# 8. ProfissionalSaude

```text
ProfissionalSaude -> Tenant
ProfissionalSaude -> Pessoa
```

One-to-one lógico com Pessoa, mas pode ser modelado `ManyToOne` com unique constraint ou `OneToOne`.

Recomendação prática:

```java
@OneToOne(fetch = LAZY)
@JoinColumn(unique = true)
```

se uma Pessoa só puder ter um ProfissionalSaude por tenant.

---

# 9. Paciente

```text
Paciente -> Tenant
Paciente -> Pessoa
```

Também relação lógica 1:1 com Pessoa.

Se a mesma Pessoa só pode ter um Paciente por tenant:

constraint unique em `pessoa_id`.

---

# 10. ProfissionalEspecialidade

```text
ProfissionalEspecialidade -> ProfissionalSaude
ProfissionalEspecialidade -> Especialidade
```

Constraint:

```text
unique(profissional_id, especialidade_id)
```

Sem cascade.

---

# 11. ProfissionalUnidade

```text
ProfissionalUnidade -> ProfissionalSaude
ProfissionalUnidade -> Unidade
```

Constraint:

```text
unique(profissional_id, unidade_id)
```

---

# 12. ProfissionalProcedimento

```text
ProfissionalProcedimento -> ProfissionalSaude
ProfissionalProcedimento -> Procedimento
```

Constraint:

```text
unique(profissional_id, procedimento_id)
```

---

# 13. DisponibilidadeProfissional

```text
DisponibilidadeProfissional
 -> ProfissionalSaude
 -> Unidade
 -> Tenant
```

`tenant_id` pode parecer redundante, mas ajuda isolamento e queries.

Se mantido, Service deve garantir consistência entre os FKs.

---

# 14. Agendamento

```text
Agendamento
 -> Tenant
 -> Unidade
 -> Paciente
 -> ProfissionalSaude
 -> Procedimento
 -> Usuario (criadoPor)
```

Sem cascade.

Agendamento não deve criar automaticamente Paciente/Profissional.

---

# 15. EventoAgendamento

```text
EventoAgendamento -> Agendamento
```

Aqui pode existir:

```text
Agendamento 1:N EventoAgendamento
```

Mas não é obrigatório mapear coleção no Agendamento.

Recomendação prática:

manter apenas:

```text
EventoAgendamento -> Agendamento
```

e consultar via Repository.

Evita carregar histórico acidentalmente.

---

# 16. Prontuario

```text
Prontuario -> Tenant
Prontuario -> Paciente
```

Constraint:

```text
unique(tenant_id, paciente_id)
```

Sem cascade destrutivo.

---

# 17. Atendimento

```text
Atendimento
 -> Tenant
 -> Prontuario
 -> Unidade
 -> ProfissionalSaude
 -> Agendamento nullable
```

Sem cascade.

Agendamento pode ser null.

Não duplicar `Paciente` dentro de Atendimento; ele vem de `Prontuario`.

Quando `agendamento_id` existir, considerar constraint UNIQUE para impedir dois atendimentos derivados do mesmo agendamento.

---

# 18. EvolucaoClinica

```text
EvolucaoClinica
 -> Tenant
 -> Atendimento
 -> ProfissionalSaude
 -> EvolucaoClinica retificacaoDe nullable
```

Self-reference LAZY.

Não usar orphanRemoval.

---

# 19. Prescricao

```text
Prescricao
 -> Tenant
 -> Atendimento
 -> ProfissionalSaude
```

Não duplicar `Paciente` em Prescricao; ele é derivado do Atendimento/Prontuario.

Uma prescrição possui itens.

Aqui existe um caso onde cascade pode ser útil.

Se o aggregate for controlado somente por Prescricao:

```java
@OneToMany(
    mappedBy = "prescricao",
    cascade = CascadeType.ALL,
    orphanRemoval = true
)
```

**somente enquanto RASCUNHO**.

Mas isso exige cuidado após emissão.

Alternativa mais simples e segura:

- ItemPrescricao possui FK Prescricao;
- Service salva/remove itens explicitamente;
- sem cascade.

Para o MVP, preferir **salvar explicitamente**.

---

# 20. AnexoClinico

```text
AnexoClinico -> Tenant
AnexoClinico -> Prontuario
AnexoClinico -> Atendimento nullable
```

Não duplicar `Paciente` em AnexoClinico; ele é derivado do Prontuario.

Sem cascade.

Excluir metadata não deve apagar Prontuario.

Storage é tratado pelo Service.

---

# 21. Notificacao

Pode referenciar:

```text
tenantId
usuarioId
```

Recomendação:

relacionamento leve ou IDs.

Como é operacional, usar IDs pode reduzir acoplamento.

---

# 22. AuditLog

Preferir IDs primitivos:

```text
tenantId
usuarioId
unidadeId
recursoId
```

Não mapear dezenas de relações.

AuditLog precisa sobreviver à evolução do domínio.

---

# 23. Bidirecionalidade

Só criar relação bidirecional quando o código realmente precisar navegar nos dois sentidos.

Exemplo desnecessário:

```text
Tenant.getTodasAsPessoas()
```

Isso pode gerar coleções enormes.

Preferir Repository.

---

# 24. orphanRemoval

Usar somente quando filho não faz sentido sem pai.

Mesmo assim, revisar impacto.

Não usar em:

- atendimento;
- evolução;
- agendamento;
- evento;
- audit log;
- prontuário.

---

# 25. Checklist JPA

Antes de concluir um relacionamento:

- [ ] quem possui FK?
- [ ] relação precisa ser bidirecional?
- [ ] fetch está LAZY?
- [ ] cascade é realmente necessário?
- [ ] orphanRemoval é seguro?
- [ ] há unique constraint lógica?
- [ ] Service valida mesmo tenant?
- [ ] JSON não vai serializar Entity?
- [ ] delete do pai não apaga histórico?
