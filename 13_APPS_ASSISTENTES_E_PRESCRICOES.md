# 13 — Apps, Assistentes e Prescrições

## Ecossistema

```text
                    AkosMed Backend
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   AkosMed Web      Akos Assistant   AkosMed Patient
```

Todos consomem o mesmo domínio/backend.

---

# Akos Assistant

## Primeira implementação

Não criar bot ainda.

Criar Services/endpoints:

```text
ProfessionalDayService
```

Retorna:

- consultas do dia;
- próxima consulta;
- pacientes em check-in;
- quantidade restante;
- cancelamentos;
- atendimentos em aberto.

Endpoints:

```http
GET /api/v1/me/agenda/hoje
GET /api/v1/me/pacientes/aguardando
GET /api/v1/me/resumo-diario
```

## Evolução

Depois:

- push;
- app do profissional;
- bot.

O bot deve ser um adapter, não conter regra de negócio.

---

# AkosMed Patient

## Tecnologia sugerida

Kotlin/Android.

## Backend necessário

```text
/api/v1/me/*
```

O usuário autenticado é convertido em Paciente pelo backend.

## MVP do app

1. login;
2. próximas consultas;
3. detalhes da consulta;
4. prescrições;
5. notificações.

## Segunda fase

- horários disponíveis;
- autoagendamento;
- cancelamento/reagendamento;
- documentos;
- materiais terapêuticos enviados pelo profissional;
- vídeos de orientação/exercícios;
- solicitações para clínica.

Essa ordem permite lançar um app útil antes de construir um fluxo completo de atendimento digital.

---

# Materiais terapêuticos e vídeos para o paciente

## Objetivo

Permitir que um profissional envie ao paciente uma orientação reutilizável e segura após o atendimento.

Exemplo principal:

```text
Fisioterapeuta
→ seleciona o paciente
→ envia/associa vídeo demonstrando o exercício correto
→ adiciona título e instruções
→ paciente visualiza no AkosMed Patient
```

O recurso não deve ser modelado apenas como "vídeo de fisioterapia". Criar um conceito mais genérico para permitir evolução para:

- exercícios de fisioterapia;
- orientações pós-operatórias;
- exercícios de fonoaudiologia;
- cuidados de enfermagem;
- orientações nutricionais;
- PDFs, imagens ou links educativos no futuro.

## Modelo sugerido

```text
OrientacaoPaciente
```

Campos planejados:

```text
id
publicId
tenant
paciente
profissional
atendimento nullable
titulo
descricao
tipo
storageKey nullable
urlExterna nullable
mimeType nullable
duracaoSegundos nullable
visivelAPartirDe nullable
expiraEm nullable
ativo
createdAt
```

Enum inicial:

```text
TipoOrientacaoPaciente
VIDEO
DOCUMENTO
LINK
TEXTO
```

Para a primeira implementação do app, `VIDEO` pode ser o único tipo utilizado, mantendo o domínio preparado para expansão.

## Relacionamentos

```text
Paciente 1:N OrientacaoPaciente
ProfissionalSaude 1:N OrientacaoPaciente
Atendimento 1:N OrientacaoPaciente (opcional)
```

A orientação deve sempre pertencer ao mesmo tenant do paciente e do profissional.

Quando vinculada a um atendimento, o Service deve validar que atendimento, paciente e profissional são compatíveis.

## Storage

Não armazenar o binário do vídeo diretamente no PostgreSQL.

O banco guarda apenas metadata e referência do arquivo.

Usar a abstração já planejada:

```text
StorageService
```

Desenvolvimento:

```text
filesystem local
```

Produção futura:

```text
S3 / storage compatível
```

Evitar URL pública permanente do vídeo. Quando possível, o backend deve gerar URL temporária/assinada após validar autorização.

## Endpoints planejados

Paciente autenticado:

```http
GET /api/v1/me/orientacoes
GET /api/v1/me/orientacoes/{orientacaoPublicId}
```

Profissional autorizado:

```http
POST /api/v1/pacientes/{pacientePublicId}/orientacoes
GET  /api/v1/pacientes/{pacientePublicId}/orientacoes
DELETE /api/v1/pacientes/{pacientePublicId}/orientacoes/{orientacaoPublicId}
```

O `DELETE` deve representar inativação/remoção lógica quando houver necessidade de preservar histórico/auditoria.

O fluxo exato de upload pode ser definido somente quando o `StorageService` estiver estabilizado.

## DTOs esperados

```text
CriarOrientacaoPacienteDTO
OrientacaoPacienteResponseDTO
OrientacaoPacienteResumoDTO
```

`ResponseDTO` nunca expõe `id Long`, `storageKey` interno ou caminho físico do arquivo.

## Segurança e privacidade

- paciente só acessa orientações vinculadas a ele;
- profissional só envia material dentro do tenant autorizado;
- não aceitar `tenantId` vindo do cliente;
- URLs de mídia não devem permitir acesso cross-tenant;
- registrar criação/remoção em auditoria quando o módulo estiver ativo;
- definir limite de tamanho e tipos MIME aceitos antes de liberar upload em produção;
- não permitir arquivos executáveis;
- considerar o vídeo/material parte dos dados relacionados ao cuidado do paciente e aplicar os mesmos cuidados de privacidade do restante do sistema.

## Experiência do app

No AkosMed Patient, criar uma área como:

```text
Orientações do profissional
```

Cada item pode mostrar:

- profissional responsável;
- título;
- instrução curta;
- data de envio;
- vídeo/material;
- vínculo com consulta/atendimento quando existir.

Exemplo:

```text
Exercício para mobilidade do ombro
Enviado por: Fisioterapeuta
Orientação: 3 séries de 10 repetições, respeitando o limite de dor.
[Assistir vídeo]
```

O app não deve permitir que o paciente altere o conteúdo clínico enviado pelo profissional.

---

# Prescrição

## Modelo prático

Não criar `Medicamento` de catálogo no MVP.

```text
Prescricao
└── ItemPrescricao
```

`ItemPrescricao` armazena o texto estruturado do item.

## Campos

```text
medicamentoNome
concentracaoApresentacao
dose
unidadeDose
viaAdministracao
vezesAoDia
intervaloHoras
duracaoDias
dataInicio
dataFim
usoContinuo
seNecessario
instrucoes
```

Nem todos precisam ser obrigatórios ao mesmo tempo.

Exemplo:

```text
Dipirona 500 mg
1 comprimido
via oral
até 4 vezes ao dia
se necessário para dor
```

Outro:

```text
Amoxicilina 500 mg
1 cápsula
via oral
3 vezes ao dia
8/8h
7 dias
```

## Regras

- prescrição nasce como RASCUNHO;
- itens podem ser alterados em RASCUNHO;
- emissão muda para EMITIDA;
- após emissão, não editar normalmente;
- correção gera nova prescrição/cancelamento conforme regra futura.

## Medicamentos ativos

Não salvar lista em `Paciente`.

Criar consulta derivada:

```text
PrescriptionQueryService.listActiveForPatient(...)
```

Critério pode usar:

- `usoContinuo`;
- `dataInicio/dataFim`;
- `duracaoDias`.

## Receita

Gerador futuro:

```text
PrescriptionDocumentService
```

Entrada:

```text
Prescricao
```

Saída:

```text
PDF/documento
```

A mesma prescrição alimenta:

- prontuário;
- app;
- documento;
- histórico.

---

# Lembrete de medicamento

Não é Core.

Quando chegar a hora:

```text
MedicationReminderService
```

Pode gerar lembretes a partir dos itens prescritos.

Não criar tabela de horários de medicamento antes de definir experiência do app.

---

# Contato do paciente

Evitar começar com chat.

Módulo futuro simples:

```text
SolicitacaoPaciente
- paciente
- unidade
- tipo
- mensagem
- status
```

Isso atende boa parte das necessidades administrativas com muito menos complexidade.


---

## Posição no roadmap oficial

```text
ETAPA 7  → estrutura de prescrição + StorageService/AnexoClinico
ETAPA 8  → endpoints operacionais que servirão ao futuro assistente
ETAPA 15 → API do paciente + endpoints de orientações/materiais
ETAPA 16 → aplicativo Kotlin + área de orientações/vídeos
ETAPA 17 → Akos Assistant
```

O app e o bot não serão desenvolvidos antes das APIs correspondentes estarem estáveis.

O upload/streaming de vídeo só entra depois da abstração de storage, autenticação e isolamento multi-tenant estarem validados.


---

## Dependência de qualidade do backend

O app Kotlin e o Akos Assistant dependem de contratos REST estáveis.

Por isso:

- DTOs devem ser consistentes;
- erros devem possuir `code`;
- paginação deve ser previsível;
- endpoints `/me` não recebem pacienteId arbitrário;
- Swagger só será gerado após PostgreSQL estabilizado;
- Postman validará o contrato antes do mobile.
