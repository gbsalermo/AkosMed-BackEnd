# 19 — Checklist de Revisão de Qualidade

Use antes de marcar qualquer subetapa como concluída.

---

# A. Domínio

- [ ] A Entity representa uma coisa real do domínio?
- [ ] Não criei Entity apenas para “organizar código”?
- [ ] Campos pertencem realmente a essa entidade?
- [ ] Status tem enum?
- [ ] A regra de ativação/inativação está definida?
- [ ] Hard delete é seguro? Se não, não usei.
- [ ] Regras importantes estão no Service?

---

# B. Relacionamentos

- [ ] Quem possui a FK está claro?
- [ ] Os dois lados pertencem ao mesmo tenant?
- [ ] Evitei `ManyToMany`?
- [ ] LAZY está definido?
- [ ] Cascade foi justificado?
- [ ] Não usei `CascadeType.ALL` por comodidade?
- [ ] Não usei `orphanRemoval` em histórico?
- [ ] Existe unique constraint quando necessário?
- [ ] Não criei coleção bidirecional sem necessidade?

---

# C. DTO

- [ ] CreateDTO só contém campos criáveis?
- [ ] UpdateDTO só contém campos editáveis?
- [ ] ResponseDTO não expõe hash/sigilo?
- [ ] Entity não sai diretamente no Controller?
- [ ] TenantId não vem livremente do frontend?
- [ ] Respostas compostas usam SummaryDTO quando necessário?
- [ ] Datas estão em tipo adequado?
- [ ] Dinheiro é BigDecimal?

---

# D. Repository

- [ ] Repository trabalha com Entity?
- [ ] Query tenant-scoped possui tenantId?
- [ ] Evitei `findById()` onde deveria usar tenant?
- [ ] Listagem grande usa paginação?
- [ ] Query é legível?
- [ ] SQL nativo foi realmente necessário?
- [ ] Query de conflito considera overlap correto?

---

# E. Service

- [ ] `@Transactional` em escrita?
- [ ] `readOnly=true` em leitura?
- [ ] Tenant vem do contexto?
- [ ] Carrego relacionamentos tenant-scoped?
- [ ] Valido vínculo de unidade?
- [ ] Valido status?
- [ ] Evito duplicidade?
- [ ] Lanço exception de negócio adequada?
- [ ] Não engulo exception com `catch(Exception)`?
- [ ] Não há regra de negócio no Controller?

---

# F. Controller

- [ ] Usa `@Valid`?
- [ ] Recebe DTO?
- [ ] Retorna DTO?
- [ ] Status HTTP correto?
- [ ] Não chama Repository?
- [ ] Não monta Entity?
- [ ] Não decide regra de tenant?
- [ ] Ações de domínio têm endpoint explícito?

---

# G. Multi-tenant

- [ ] Criei recurso no Tenant A?
- [ ] Usuário Tenant B não consegue ler?
- [ ] Usuário Tenant B não consegue alterar?
- [ ] Usuário Tenant B não consegue usar ID relacionado?
- [ ] Unidade pertence ao mesmo tenant?
- [ ] Profissional/paciente/procedimento são do mesmo tenant?

---

# H. Segurança

- [ ] senha nunca retorna?
- [ ] token não aparece em log?
- [ ] dado clínico sensível não vai para log?
- [ ] endpoint exige perfil apropriado?
- [ ] paciente só vê seus dados?
- [ ] acesso por unidade foi considerado?

---

# I. Testes

- [ ] happy path?
- [ ] not found?
- [ ] validação?
- [ ] conflito?
- [ ] tenant incorreto?
- [ ] status inválido?
- [ ] relacionamento inválido?
- [ ] `mvn test` executado?
- [ ] resultado registrado?

---

# J. Performance básica

- [ ] alguma lista pode crescer demais?
- [ ] paginação?
- [ ] EAGER inesperado?
- [ ] loop faz query por item?
- [ ] DTO força carregamento de grafo inteiro?
- [ ] endpoint retorna dados demais?

Não otimizar prematuramente, mas evitar N+1 óbvio.

---

# K. H2 → PostgreSQL

Durante Core:

- [ ] regra não depende de H2-specific SQL?
- [ ] dinheiro não é float?
- [ ] constraints lógicas estão anotadas/documentadas?

Na ETAPA 11:

- [ ] migrations;
- [ ] indexes;
- [ ] concurrency;
- [ ] Testcontainers;
- [ ] timezone;
- [ ] unique constraints.

---

# L. Swagger/Postman

Só nas etapas finais.

Swagger:

- [ ] contratos;
- [ ] exemplos;
- [ ] errors;
- [ ] JWT;
- [ ] paginação.

Postman:

- [ ] happy path;
- [ ] auth;
- [ ] cross-tenant;
- [ ] conflicts;
- [ ] fluxo completo.

---

# M. Qualidade de código

- [ ] nomes claros;
- [ ] método não faz coisas demais;
- [ ] classe não possui responsabilidade múltipla;
- [ ] constantes/enums em vez de magic strings;
- [ ] sem código comentado abandonado;
- [ ] sem TODO crítico;
- [ ] commit pequeno;
- [ ] CONTINUIDADE atualizado.

---

# Regra final

Se houver dúvida em qualquer item crítico de:

```text
tenant
segurança
prontuário
prescrição
agendamento
```

não marque a subetapa como concluída até revisar.
