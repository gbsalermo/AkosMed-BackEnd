# 18 — Mapa de Métodos dos Services

> Este documento ajuda a verificar se cada módulo possui responsabilidade coerente.  
> Os nomes podem mudar, mas as fronteiras devem permanecer.

---

# Convenção de entrada dos Services

Métodos chamados por Controllers recebem identificadores externos como:

```text
UUID publicId
```

O Service combina o `publicId` com o Tenant atual e resolve a Entity.

Depois da resolução/autorização, IDs `Long` podem ser usados internamente.

Não usar como assinatura pública padrão:

```text
buscarPorId(Long idDoCliente)
atualizar(Long idDoCliente, ...)
```

Referências:

- `15_PADROES_ENTIDADES_DTOS_REPOSITORIES.md`;
- `21_PUBLIC_ID_E_CONCORRENCIA.md`.

---

# TenantService

Administração global do SaaS.

```text
criar(TenantCreateDTO)
buscarPorPublicId(UUID publicId)
listar(Pageable)
atualizar(UUID publicId, TenantUpdateDTO)
ativar(UUID publicId)
suspender(UUID publicId)
buscarPorSlug(String slug)
```

Cuidados:

- `slug` único;
- status;
- timezone;
- `publicId` nunca escolhido pelo cliente;
- autorização de `superAdmin` nas operações globais.

---

# UnidadeService

```text
criar(UnidadeCreateDTO)
buscarPorPublicId(UUID publicId)
listar(Pageable)
atualizar(UUID publicId, UnidadeUpdateDTO)
ativar(UUID publicId)
desativar(UUID publicId)
```

Internamente sempre tenant-scoped.

Cuidados:

- código único por Tenant;
- Tenant suspenso;
- `@Version` avaliado;
- sem hard delete.

---

# PessoaService

Pode ser mais interno do que público.

```text
criar(...)
buscarTenantScopedPorPublicId(UUID publicId)
atualizar(...)
```

Pessoa não precisa de Controller genérico inicialmente se nascer via Paciente/Profissional/Usuário.

Quando outro Service já possui uma `Pessoa` autorizada, pode trabalhar com a própria Entity/ID interno em vez de reconsultar por publicId.

---

# AuthService

```text
login(LoginDTO)
refresh(RefreshRequestDTO)
logout(RefreshRequestDTO)
```

Responsabilidades:

- credencial;
- Tenant por slug;
- `UsuarioTenant`;
- status;
- senha;
- emissão/revogação de tokens.

Claims tenant-scoped devem seguir o padrão público:

```text
usuarioPublicId
tenantPublicId
usuarioTenantPublicId
perfilTenant
```

IDs `Long` ficam no contexto interno depois da resolução.

---

# UsuarioService

```text
criarUsuarioTenant(...)
buscarUsuarioTenantPorPublicId(UUID publicId)
ativarVinculo(UUID publicId)
desativarVinculo(UUID publicId)
alterarPerfil(UUID publicId, ...)
```

Não misturar emissão de token com CRUD administrativo.

O vínculo `UsuarioTenant` é tenant-scoped mesmo que `Usuario` seja global.

---

# UsuarioUnidadeService

```text
adicionarUnidade(UUID usuarioTenantPublicId, UUID unidadePublicId)
removerUnidade(UUID usuarioTenantPublicId, UUID unidadePublicId)
listarUnidadesPermitidas(UUID usuarioTenantPublicId)
validarAcessoUnidade(...)
```

O Service valida que vínculo e Unidade pertencem ao mesmo Tenant.

---

# EspecialidadeService

```text
criar
buscarPorPublicId
listar
atualizar
ativar
desativar
```

Todas as operações são tenant-scoped.

---

# ProcedimentoService

```text
criar
buscarPorPublicId
listar
atualizar
ativar
desativar
```

Cuidados:

- duração > 0;
- BigDecimal para valor;
- código único por Tenant.

---

# ProfissionalService

```text
criar(ProfissionalCreateDTO)
buscarPorPublicId(UUID publicId)
listar(Pageable)
atualizar(UUID publicId, ProfissionalUpdateDTO)
ativar(UUID publicId)
desativar(UUID publicId)
```

Relacionamentos ficam em métodos próprios ou Services de vínculo.

---

# ProfissionalEspecialidadeService

```text
vincular(UUID profissionalPublicId, UUID especialidadePublicId)
desvincular(UUID profissionalPublicId, UUID especialidadePublicId)
listar(UUID profissionalPublicId)
definirPrincipal(UUID profissionalPublicId, UUID especialidadePublicId)
```

Validar mesmo Tenant.

---

# ProfissionalUnidadeService

```text
vincular(UUID profissionalPublicId, UUID unidadePublicId)
desvincular(UUID profissionalPublicId, UUID unidadePublicId)
listar(UUID profissionalPublicId)
validarVinculo(...)
```

A vinculação é pré-condição para agenda naquela Unidade.

---

# ProfissionalProcedimentoService

```text
vincular(UUID profissionalPublicId, UUID procedimentoPublicId, ...)
atualizarConfiguracao(...)
desativar(...)
listar(UUID profissionalPublicId)
resolverDuracao(...)
resolverValor(...)
```

`resolverDuracao`:

```text
duracaoMinutosOverride
senão
Procedimento.duracaoPadraoMinutos
```

---

# PacienteService

```text
criar(PacienteCreateDTO)
buscarPorPublicId(UUID publicId)
listar(Pageable, filtros)
buscarPorCpf(...)
buscarPorNumeroProntuario(...)
atualizar(UUID publicId, PacienteUpdateDTO)
ativar(UUID publicId)
inativar(UUID publicId)
```

Pode criar Pessoa internamente.

Cuidados:

- isolamento de Tenant;
- número de prontuário único por Tenant;
- paciente inativo;
- `@Version` avaliado;
- nenhuma PK Long exposta.

---

# DisponibilidadeService

```text
criarDisponibilidade(UUID profissionalPublicId, ...)
atualizarDisponibilidade(UUID profissionalPublicId, UUID disponibilidadePublicId, ...)
removerOuDesativar(UUID profissionalPublicId, UUID disponibilidadePublicId)
listarPorProfissional(UUID profissionalPublicId)
buscarHorariosDisponiveis(
    UUID profissionalPublicId,
    UUID unidadePublicId,
    UUID procedimentoPublicId,
    LocalDate data
)
```

Não persistir slots calculados.

Services temporais usam `Clock` quando a regra depender do momento atual.

---

# BloqueioAgendaService

```text
criar(UUID profissionalPublicId, ...)
buscarPorPublicId(UUID publicId)
listarPorPeriodo(...)
cancelarOuDesativar(UUID publicId)
```

Validar profissional, Unidade e Tenant.

---

# AgendamentoService

```text
criar(AgendamentoCreateDTO)
buscarPorPublicId(UUID publicId)
listarPorPeriodo(...)
listarPorProfissional(UUID profissionalPublicId, ...)
listarPorPaciente(UUID pacientePublicId, ...)
listarPorUnidade(UUID unidadePublicId, ...)
confirmar(UUID publicId)
cancelar(UUID publicId, ...)
reagendar(UUID publicId, ReagendamentoDTO)
checkIn(UUID publicId)
marcarFalta(UUID publicId)
```

Responsabilidades:

- Tenant;
- disponibilidade;
- vínculo profissional/unidade;
- procedimento;
- conflito/overlap;
- transição de status;
- evento;
- concorrência;
- idempotência quando adotada.

Criação/reagendamento:

```text
@Transactional
→ lock PESSIMISTIC_WRITE no profissional
→ revalidar
→ salvar
```

Dois pacientes disputando o mesmo slot devem produzir exatamente:

```text
1 sucesso
1 AGENDAMENTO_CONFLITO
```

---

# EventoAgendamentoService

Idealmente interno.

```text
registrarCriacao(Agendamento, ...)
registrarMudancaStatus(Agendamento, ...)
registrarReagendamento(Agendamento, ...)
```

Não precisa Controller inicial.

Trabalha com Entity/IDs internos porque é chamado depois de o Agendamento já estar autorizado dentro da transação.

Histórico append-only.

---

# ProntuarioService

```text
obterOuCriarDoPaciente(UUID pacientePublicId)
buscarDoPaciente(UUID pacientePublicId)
listarHistorico(UUID pacientePublicId, ...)
```

Sem delete.

O paciente pode ser resolvido diretamente do contexto em endpoints `/me`.

---

# AtendimentoService

```text
iniciarPorAgendamento(UUID agendamentoPublicId)
criarAvulso(AtendimentoAvulsoCreateDTO)
buscarPorPublicId(UUID publicId)
listarPorPaciente(UUID pacientePublicId, ...)
concluir(UUID publicId)
```

Valida status do Agendamento quando aplicável.

`Atendimento` referencia `Prontuario`; não duplicar paciente apenas para facilitar query.

---

# EvolucaoClinicaService

```text
registrar(UUID atendimentoPublicId, EvolucaoCreateDTO)
listarPorAtendimento(UUID atendimentoPublicId)
retificar(UUID evolucaoPublicId, RetificacaoDTO)
```

Sem update destrutivo.

Retificação cria novo registro e preserva original.

---

# PrescricaoService

```text
criarRascunho(UUID atendimentoPublicId, ...)
buscarPorPublicId(UUID publicId)
listarPorPaciente(UUID pacientePublicId, ...)
listarPorAtendimento(UUID atendimentoPublicId, ...)
adicionarItem(UUID prescricaoPublicId, ...)
atualizarItem(UUID prescricaoPublicId, UUID itemPublicId, ...)
removerItem(UUID prescricaoPublicId, UUID itemPublicId)
emitir(UUID publicId, ...)
cancelar(UUID publicId, ...)
```

Pode usar `ItemPrescricaoRepository` internamente.

Não precisa `ItemPrescricaoService` público se não houver vantagem.

Após EMITIDA, não editar silenciosamente.

Avaliar idempotência na emissão conforme roadmap.

---

# PrescriptionDocumentService

```text
gerar(UUID prescricaoPublicId)
```

O Service resolve a Prescrição autorizada e então gera a representação/documento.

No início pode retornar bytes/DTO/arquivo por abstraction definida na etapa.

Assinatura digital fica para futuro.

---

# StorageService

Interface pequena e independente de provedor:

```text
salvar
abrir
removerOuInativarReferencia
```

Implementações planejadas:

```text
LocalStorageService → desenvolvimento
S3StorageService/compatível → produção futura
```

Não expor `storageKey` ao cliente.

Autorização acontece antes de entregar o conteúdo.

---

# AnexoClinicoService

```text
upload(...)
listarPorProntuario(...)
listarPorAtendimento(UUID atendimentoPublicId)
download(UUID anexoPublicId)
inativarOuRemoverLogicamente(UUID anexoPublicId)
```

Valida Tenant/autorização antes de chamar Storage.

---

# AgendaDiariaService

Query Service.

```text
buscarMinhaAgendaHoje()
buscarMinhaAgenda(LocalDate data)
buscarPacientesAguardando()
buscarResumoDiario()
```

Identidade do profissional vem do contexto autenticado.

Não criar entidade só para o resumo.

---

# NotificacaoService

```text
criar(...)
listarMinhas(Pageable)
marcarComoLida(UUID notificacaoPublicId)
marcarTodasComoLidas()
```

Em `/me`, `UsuarioTenant` vem do contexto.

---

# AuditService

```text
registrar(...)
```

Preferencialmente chamado por Services críticos.

Pode receber IDs internos/metadata segura porque é infraestrutura interna, mas nunca deve gravar conteúdo clínico sensível desnecessariamente.

---

# OrientacaoPacienteService — ETAPA 15, pós-MVP

**Não implementar durante o Core.**

Quando a ETAPA 15 chegar:

```text
criarParaPaciente(UUID pacientePublicId, CriarOrientacaoPacienteDTO)
listarDoPaciente(UUID pacientePublicId, ...)
buscarParaPaciente(UUID pacientePublicId, UUID orientacaoPublicId)
inativar(UUID pacientePublicId, UUID orientacaoPublicId)
listarMinhasOrientacoes()
buscarMinhaOrientacao(UUID orientacaoPublicId)
gerarAcessoTemporarioAoMaterial(UUID orientacaoPublicId)
```

Responsabilidades:

- Tenant;
- paciente/profissional;
- atendimento opcional compatível;
- autorização;
- metadata;
- StorageService;
- URL temporária/assinada quando suportada;
- MIME/tamanho;
- auditoria;
- paciente nunca altera conteúdo enviado pelo profissional.

Detalhes: `13_APPS_ASSISTENTES_E_PRESCRICOES.md`.

---

# Regra de tamanho de Service

Se um Service começar a fazer:

```text
auth + paciente + agenda + prontuario + prescricao
```

ele está grande demais.

Separar por responsabilidade real do domínio, não por abstração artificial.

---

# Checklist de Service

- [ ] domínio claro?
- [ ] entrada pública usa publicId quando há recurso externo?
- [ ] Tenant é resolvido internamente?
- [ ] query externa combina publicId + Tenant quando aplicável?
- [ ] transação está no lugar correto?
- [ ] relacionamentos foram validados?
- [ ] acesso por Unidade foi validado?
- [ ] status/transições foram validados?
- [ ] exception específica?
- [ ] Repository não vazou ao Controller?
- [ ] ResponseDTO não expõe Entity/Long id?
- [ ] `Clock` usado em regra temporal?
- [ ] `@Version` avaliado em update mutável?
- [ ] idempotência avaliada em operação sujeita a retry?
- [ ] concorrência protegida quando há recurso disputado?
- [ ] testes cobrem happy path, conflito e cross-tenant?

---

# Regra final

```text
Controller → publicId/DTO
Service → regra + autorização + transação
Repository → Entity + IDs internos depois da resolução
```

Não transformar PK `Long` em contrato externo por conveniência.
