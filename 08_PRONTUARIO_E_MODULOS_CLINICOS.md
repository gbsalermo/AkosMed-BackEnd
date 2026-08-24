# 08 — Prontuário e Módulos Clínicos

## Core clínico

```text
Paciente
  ↓
Prontuario
  ↓
Atendimento
  ├── EvolucaoClinica
  ├── Prescricao
  └── AnexoClinico
```

## Prontuário

O `Prontuario` é um agregador, não uma ficha gigante.

Não colocar campos do tipo:

```text
pressao
peso
queixa
odontograma
avaliacao_psicologica
...
```

no prontuário base.

Cada atendimento registra o que aconteceu.

## Evolução clínica

MVP:

```text
conteudo TEXT
```

Isso mantém o Core genérico.

Depois, especializações podem criar dados estruturados adicionais.

## Retificação

Fluxo:

```text
Evolução original
      ↓
Nova evolução com retificacaoDeId
```

A original permanece.

## Diagnóstico

Não é obrigatório no primeiro corte.

Pode ser adicionado como entidade estruturada depois quando houver definição de terminologia/código e necessidade real.

Até lá, informação clínica permanece na evolução.

## Anexos

Categorias simples:

```text
EXAME
IMAGEM
LAUDO
RECEITA
ATESTADO
RELATORIO
OUTRO
```

## Especializações

O Core não deve importar diretamente classes `dental`, `psychology` etc.

### Dental

Futuro:

```text
Odontograma
DenteRegistro
PlanoTratamentoOdontologico
```

### Psicologia

Futuro:

```text
RegistroPsicologicoEstruturado
PlanoAcompanhamento
InstrumentoAvaliacao
```

### Oftalmologia

Futuro:

```text
AvaliacaoVisual
Refracao
PrescricaoOptica
```

## Estratégia de extensão

Quando a primeira especialização real for implementada:

```text
modules/dental
```

Ela referencia `Atendimento` como raiz clínica.

Exemplo:

```text
Atendimento 1:1 OdontogramaSessao
```

Não duplicar Paciente, Profissional, Agendamento ou Prontuário.

## Formulários customizáveis

Boa ideia, mas não MVP.

Somente criar `FormularioModelo` + JSONB quando existir mais de uma clínica pedindo formulários diferentes.

Antes disso, DTOs/entidades específicas são mais simples e seguros.


---

## Regra de implementação clínica

Prontuário é uma área de histórico.

Portanto:

- sem hard delete;
- sem `CascadeType.REMOVE` partindo de Paciente/Prontuario;
- sem update silencioso de EvolucaoClinica;
- retificação preserva original;
- acesso precisa de tenant + perfil/autorização.

Consultar `16_RELACIONAMENTOS_JPA_CASCADE_FETCH.md`.
