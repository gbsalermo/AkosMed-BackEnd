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
- solicitações para clínica.

Essa ordem permite lançar um app útil antes de construir um fluxo completo de atendimento digital.

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
ETAPA 7  → estrutura de prescrição
ETAPA 8  → endpoints operacionais que servirão ao futuro assistente
ETAPA 15 → API do paciente
ETAPA 16 → aplicativo Kotlin
ETAPA 17 → Akos Assistant
```

O app e o bot não serão desenvolvidos antes das APIs correspondentes estarem estáveis.


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
