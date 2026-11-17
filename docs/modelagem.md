# Modelagem dos dados

São dois modelos, um para cada banco.

## Banco operacional — `svp`

Modelo relacional normalizado até a 3ª forma normal, com [46] tabelas.


As tabelas estão agrupadas assim:

| Grupo | Principais tabelas | Observação |
|---|---|---|
| Cadastros | [tutor, animal, espécie, raça, funcionário...] | |
| Agenda | [agendamento, ...] | |
| Prontuário | [atendimento, prescrição, pedido de exame, resultado...] | |
| Financeiro | [conta, pagamento, nota...] | |
| Controle | [usuário, perfil, sessão, auditoria, fila de envio, fila de recebimento] | Não são do negócio, sustentam segurança e integração |

Regras que o próprio banco garante:

- chave estrangeira em todos os relacionamentos — não existe atendimento sem animal, nem animal sem tutor;
- `UNIQUE` em [CPF do tutor, e-mail do usuário, ...];
- `CHECK` em [valores não negativos, datas coerentes, ...].

[Um parágrafo com uma decisão de modelagem que o grupo discutiu. Exemplo: por que tutor e animal são tabelas separadas, ou como foi tratado o animal com mais de um tutor.]

## Banco analítico — `svp_dw`

Modelo estrela, com 3 tabelas fato e 6 dimensões.

| Fato | O que mede | Grão |
|---|---|---|
| [fato_atendimento] | [quantidade e duração dos atendimentos] | [um atendimento] |
| [fato_exame] | [pedidos e tempo de retorno] | [um pedido] |
| [fato_financeiro] | [valor faturado e recebido] | [um lançamento] |

Dimensões: [tempo, animal, tutor, profissional, serviço, ...].

Indicadores que saem daí:

- [atendimentos por veterinário por mês]
- [tempo médio entre pedido e resultado de exame]
- [faturamento por tipo de serviço]

## Por que dois modelos

No `svp` a prioridade é gravar certo e sem repetição, então os dados ficam espalhados em muitas tabelas. Para relatório isso atrapalha, porque qualquer pergunta vira uma consulta com muitos joins. No `svp_dw` os dados já ficam juntos do jeito que o relatório precisa.



---

## 1. Introdução e Premissas

A modelagem organiza os dados do SVP em 42 tabelas, agrupadas em 8 módulos, com o animal como paciente e o tutor como titular dos dados pessoais.

### Premissas adotadas

* **O paciente é o animal; o responsável é o tutor (pessoa física):** Um tutor pode ter vários animais, e cada animal tem um tutor responsável.
* **A empresa é uma rede:** Agenda, estoque e faturamento são registrados por unidade.
* **Todo atendimento nasce de um agendamento:** Encaixes e chegadas sem hora marcada são lançados pela recepção como agendamento no ato.
* **Repositório de arquivos externo:** Laudos, exames e imagens ficam no repositório de arquivos; o banco guarda apenas o caminho e o hash de cada arquivo.

### Ajustes sugeridos à etapa 1

* **Catálogo de diagnósticos:** CID é a classificação de doenças humanas. O modelo usa um catálogo próprio de diagnósticos veterinários (tabela `diagnostico`), que pode seguir uma nomenclatura padronizada como a VeNom.
* **LGPD e Sigilo Profissional:** Na LGPD, dado sensível é dado de pessoa natural; a saúde do animal não se enquadra no Art. 5º, II. Os dados do tutor são dados pessoais, e os dados clínicos estão protegidos pelo sigilo profissional do médico-veterinário. O modelo protege os dois com o mesmo rigor.
* **Formas de pagamento:** A etapa 1 diz que os clientes são apenas particulares, mas lista operadoras de convênio como parceiras. O modelo trata convênio como forma de pagamento opcional.

---

## 2. Modelo Conceitual

O cadastro diz quem é o animal e quem responde por ele, a agenda marca o atendimento e o prontuário registra o que aconteceu; os demais módulos cobram, abastecem e controlam o acesso.

| Módulo | Entidades | O que representam |
| :--- | :--- | :--- |
| **Cadastro** | `tutor`, `consentimento`, `animal`, `especie`, `raca`, `alergia`, `convenio`, `animal_convenio` | Quem é o animal, quem responde por ele, o que o tutor autorizou e qual plano cobre o animal |
| **Agenda** | `unidade`, `especialidade`, `veterinario`, `veterinario_especialidade`, `agendamento` | Onde, com quem e quando o animal será atendido, presencialmente ou por teleconsulta |
| **Prontuário** | `triagem`, `atendimento`, `diagnostico`, `atendimento_diagnostico`, `evolucao`, `prescricao`, `item_prescricao` | Sinais vitais, anamnese, diagnósticos, evolução e receitas de cada atendimento |
| **Exames** | `laboratorio`, `tipo_exame`, `solicitacao_exame`, `documento` | Pedido, coleta própria, envio ao laboratório terceirizado e laudo digitalizado |
| **Programas Crônicos** | `programa_cronico`, `inscricao_programa` | Acompanhamento contínuo de animais com doença crônica |
| **Financeiro** | `servico`, `fatura`, `item_fatura`, `pagamento`, `nota_fiscal` | Tabela de preços, cobrança, recebimento e nota fiscal de serviço |
| **Estoque** | `fornecedor`, `insumo`, `lote`, `movimentacao_estoque` | Medicamentos e descartáveis por lote, validade e unidade |
| **Equipe e Acesso** | `funcionario`, `usuario`, `perfil`, `permissao`, `perfil_permissao`, `usuario_perfil`, `log_auditoria` | Quem usa o sistema, o que cada perfil pode fazer e o registro de cada acesso |

### Regras de Negócio

* **RN01:** Todo animal tem um tutor responsável; um tutor pode ter vários animais.
* **RN02:** CPF do tutor e microchip do animal são únicos no sistema.
* **RN03:** A raça, quando informada, precisa pertencer à espécie do animal; sem raça, o animal é SRD.
* **RN04:** O agendamento só aceita um veterinário que tenha a especialidade escolhida.
* **RN05:** Um veterinário não pode ter dois agendamentos ativos no mesmo horário.
* **RN06:** O status do agendamento vai de `AGENDADO` para `CONFIRMADO` e `REALIZADO`, ou termina em `CANCELADO` ou `FALTOU`.
* **RN07:** Cada agendamento gera no máximo uma triagem e um atendimento; a teleconsulta dispensa triagem.
* **RN08:** O prontuário não é apagado: correções viram nova evolução ou alteração registrada no log.
* **RN09:** Cada atendimento tem no máximo um diagnóstico principal.
* **RN10:** O pedido de exame passa por `SOLICITADO`, `COLETADO`, `ENVIADO` e `CONCLUIDO`; o laudo fica em `documento`.
* **RN11:** A fatura guarda o valor praticado no dia, mesmo que a tabela de preços mude depois.
* **RN12:** Pagamento por convênio exige a operadora. O sistema nunca guarda número de cartão, apenas o código de autorização.
* **RN13:** Medicamento usado no atendimento dá baixa no lote e fica ligado ao atendimento.
* **RN14:** Teleconsulta, comunicações e envio de amostras ao laboratório exigem consentimento ativo do tutor para aquela finalidade.
* **RN15:** Toda leitura, inclusão, alteração ou exclusão de dado clínico gera um registro no log de auditoria, que não pode ser alterado.

---

## 3. Diagrama Entidade-Relacionamento

> **Núcleo do modelo:** O animal chega por um agendamento, que gera no máximo uma triagem e um atendimento; diagnóstico, prescrição, exames e fatura partem do atendimento. A fatura também referencia o tutor pagador, e estoque, programas crônicos, consentimento e nota fiscal complementam a estrutura lógica.

```
                    ┌─────────────┐        1:N       ┌─────────────┐
                    │   Espécie   │ ──────────────── │    Raça     │
                    │   especie   │                  │    raca     │
                    └─────────────┘                  └──────┬──────┘
                                                            │ 1:N
 ┌─────────────────┐                                 ┌──────▼──────┐       N:1        ┌─────────────┐
 │  Especialidade  │                                 │   Animal    │ ──────────────── │    Tutor    │
 │  especialidade  │                                 │   animal    │                  │    tutor    │
 └────────┬────────┘                                 └──────┬──────┘                  └─────────────┘
          │ N:N                                             │ 1:N
 ┌────────▼────────┐        1:N                      ┌──────▼──────┐       1:0..1     ┌─────────────┐
 │   Veterinário   │ ─────────────────────────────── │ Agendamento │ ──────────────── │   Triagem   │
 │   veterinario   │                                 │ agendamento │                  │   triagem   │
 └─────────────────┘                                 └──────┬──────┘                  └─────────────┘
                                                            │ 1:0..1
 ┌─────────────────┐        N:N                      ┌──────▼──────┐       1:0..1     ┌─────────────┐
 │   Diagnóstico   │ ─────────────────────────────── │ Atendimento │ ──────────────── │   Fatura    │
 │   diagnostico   │                                 │ atendimento │                  │   fatura    │
 └─────────────────┘                                 └──────┬──────┘                  └──────┬──────┘
                                                            │                                │ 1:N
                              ┌─────────────────────────────┼────────────────┐        ┌──────▼──────┐
                              │ 1:N                         │ 1:N            │        │  Pagamento  │
                       ┌──────▼──────────┐           ┌──────▼──────┐         │        │  pagamento  │
                       │ Pedido de exame │           │  Prescrição │         │        └─────────────┘
                       │solicitacao_exame│           │  prescricao │         │
                       └──────┬──────────┘           └─────────────┘         │
                              │ N:1                                          │
                       ┌──────▼──────────┐                                   │
                       │      Laudo      │                                   │
                       │    documento    │                                   │
                       └─────────────────┘                                   │
                                                                             │
═════════════════════════════════════════════════════════════════════════════╪══════════════════════
Segurança e auditoria (transversais a todas as tabelas)                      │
                                                                             │
 ┌─────────────────┐        N:N       ┌─────────────┐        1:N      ┌──────▼──────┐
 │     Perfil      │ ──────────────── │   Usuário   │ ─────────────── │  Auditoria  │ (Grava leitura
 │     perfil      │                  │   usuario   │                 │log_auditoria│  e alteração)
 └─────────────────┘                  └─────────────┘                 └─────────────┘
```

**Classificação dos dados:**
* **Dado clínico sob sigilo:** `triagem`, `atendimento`, `solicitacao_exame`, `documento`, `prescricao`.
* **Dado pessoal ou financeiro (LGPD):** `tutor`, `fatura`, `pagamento`.
* **Cadastro e apoio:** `especie`, `raca`, `animal`, `veterinario`, `especialidade`, `agendamento`, `diagnostico`, `perfil`, `usuario`, `log_auditoria`.

---

## 4. Modelo Lógico

Cada entidade vira uma tabela com chave primária substituta (`id_...`), e cada relacionamento $N:N$ vira uma tabela associativa; o modelo está na 3ª forma normal.

### Cadastro

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`tutor`** | `id_tutor` | — | `nome`, `cpf` (único), `data_nascimento`, `telefone`, `email`, `logradouro`, `numero`, `complemento`, `bairro`, `cidade`, `uf`, `cep`, `contato_validado_em`, `criado_em`, `atualizado_em`, `anonimizado_em` |
| **`consentimento`**| `id_consentimento` | `id_tutor → tutor` | `finalidade`, `versao_termo`, `aceito_em`, `revogado_em`, `canal` |
| **`especie`** | `id_especie` | — | `nome` (único) |
| **`raca`** | `id_raca` | `id_especie → especie` | `nome` |
| **`animal`** | `id_animal` | `id_tutor → tutor`<br>`id_especie → especie`<br>`(id_raca, id_especie) → raca` | `nome`, `sexo`, `data_nascimento`, `nascimento_estimado`, `porte`, `microchip` (único), `castrado`, `data_obito`, `criado_em` |
| **`alergia`** | `id_alergia` | `id_animal → animal`<br>`registrada_por → usuario` | `substancia`, `gravidade`, `observacao`, `registrada_em` |
| **`convenio`** | `id_convenio` | — | `razao_social`, `cnpj` (único), `ativo` |
| **`animal_convenio`**| `(id_animal, id_convenio)` | `id_animal → animal`<br>`id_convenio → convenio` | `numero_carteira`, `validade` |

### Agenda

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`unidade`** | `id_unidade` | — | `nome`, `cnpj` (único), `telefone`, `logradouro`, `numero`, `bairro`, `cidade`, `uf`, `cep`, `ativa` |
| **`especialidade`** | `id_especialidade` | — | `nome` (único) |
| **`veterinario`** | `id_veterinario` | `id_veterinario → funcionario` | `crmv`, `uf_crmv` |
| **`veterinario_especialidade`** | `(id_veterinario, id_especialidade)` | `id_veterinario → veterinario`<br>`id_especialidade → especialidade` | — |
| **`agendamento`** | `id_agendamento` | `id_animal → animal`<br>`(id_veterinario, id_especialidade) → veterinario_especialidade`<br>`id_unidade → unidade`<br>`criado_por → usuario` | `data_hora`, `duracao_min`, `modalidade`, `status`, `motivo`, `criado_em`, `chegada_em` |

### Prontuário

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`triagem`** | `id_triagem` | `id_agendamento → agendamento` (único)<br>`id_funcionario → funcionario` | `realizada_em`, `peso_kg`, `temperatura_c`, `freq_cardiaca`, `freq_respiratoria`, `queixa_principal`, `prioridade` |
| **`atendimento`** | `id_atendimento` | `id_agendamento → agendamento` (único)<br>`id_veterinario → veterinario` | `inicio_em`, `fim_em`, `anamnese`, `exame_fisico`, `conduta`, `finalizado` |
| **`diagnostico`** | `id_diagnostico` | — | `codigo` (único), `descricao`, `cronico` |
| **`atendimento_diagnostico`** | `(id_atendimento, id_diagnostico)` | `id_atendimento → atendimento`<br>`id_diagnostico → diagnostico` | `tipo` |
| **`evolucao`** | `id_evolucao` | `id_animal → animal`<br>`id_atendimento → atendimento` (opcional)<br>`id_funcionario → funcionario` | `registrada_em`, `texto` |
| **`prescricao`** | `id_prescricao` | `id_atendimento → atendimento` | `emitida_em`, `observacao` |
| **`item_prescricao`** | `id_item` | `id_prescricao → prescricao`<br>`id_insumo → insumo` (opcional) | `medicamento_texto`, `dose`, `via`, `frequencia`, `duracao_dias` |

### Exames

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`laboratorio`** | `id_laboratorio` | — | `razao_social`, `cnpj` (único), `email_integracao`, `ativo` |
| **`tipo_exame`** | `id_tipo_exame` | `id_servico → servico` | `nome` (único), `material`, `prazo_dias` |
| **`documento`** | `id_documento` | `id_animal → animal`<br>`enviado_por → usuario` | `tipo`, `nome_arquivo`, `caminho`, `hash_sha256`, `enviado_em` |
| **`solicitacao_exame`** | `id_solicitacao` | `id_atendimento → atendimento`<br>`id_tipo_exame → tipo_exame`<br>`id_laboratorio → laboratorio`<br>`coletado_por → funcionario`<br>`id_documento → documento` | `status`, `coletado_em`, `resultado_em`, `resultado_resumo` |

### Programas Crônicos

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`programa_cronico`** | `id_programa` | `id_servico → servico` | `nome` (único), `descricao`, `intervalo_retorno_dias` |
| **`inscricao_programa`** | `id_inscricao` | `id_animal → animal`<br>`id_programa → programa_cronico`<br>`id_veterinario → veterinario` | `inicio`, `fim`, `status` |

### Financeiro

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`servico`** | `id_servico` | — | `descricao` (único), `tipo`, `valor_base`, `ativo` |
| **`fatura`** | `id_fatura` | `id_tutor → tutor`<br>`id_atendimento → atendimento` (opcional, único)<br>`id_unidade → unidade` | `emitida_em`, `vencimento`, `desconto`, `status` |
| **`item_fatura`** | `id_item_fatura` | `id_fatura → fatura`<br>`id_servico → servico` | `quantidade`, `valor_unitario` |
| **`pagamento`** | `id_pagamento` | `id_fatura → fatura`<br>`id_convenio → convenio` (opcional) | `forma`, `valor`, `parcelas`, `codigo_autorizacao`, `pago_em` |
| **`nota_fiscal`** | `id_nota` | `id_fatura → fatura` (único) | `numero`, `serie`, `codigo_verificacao`, `emitida_em`, `valor` |

### Estoque

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`fornecedor`** | `id_fornecedor` | — | `razao_social`, `cnpj` (único), `telefone`, `email` |
| **`insumo`** | `id_insumo` | — | `nome` (único), `categoria`, `unidade_medida`, `controlado`, `estoque_minimo` |
| **`lote`** | `id_lote` | `id_insumo → insumo`<br>`id_fornecedor → fornecedor`<br>`id_unidade → unidade` | `numero_lote`, `validade`, `custo_unitario` |
| **`movimentacao_estoque`** | `id_movimentacao` | `id_lote → lote`<br>`id_atendimento → atendimento` (opcional)<br>`id_usuario → usuario` | `tipo`, `quantidade`, `realizada_em`, `motivo` |

### Equipe e Acesso

| Tabela | PK | FK | Demais Atributos |
| :--- | :--- | :--- | :--- |
| **`funcionario`** | `id_funcionario` | `id_unidade → unidade` | `nome`, `cpf` (único), `cargo`, `email` (único), `data_admissao`, `data_desligamento` |
| **`usuario`** | `id_usuario` | `id_funcionario → funcionario` (único) | `login` (único), `senha_hash`, `mfa_ativo`, `ativo`, `ultimo_acesso`, `criado_em` |
| **`perfil`** | `id_perfil` | — | `nome` (único), `descricao` |
| **`permissao`** | `id_permissao` | — | `recurso`, `acao` |
| **`perfil_permissao`** | `(id_perfil, id_permissao)` | `id_perfil → perfil`<br>`id_permissao → permissao` | — |
| **`usuario_perfil`** | `(id_usuario, id_perfil)` | `id_usuario → usuario`<br>`id_perfil → perfil` | — |
| **`log_auditoria`** | `id_log` | `id_usuario → usuario` | `ocorrido_em`, `acao`, `tabela`, `id_registro`, `dados_antes`, `dados_depois`, `ip_origem`, `justificativa` |

### Relacionamentos e Cardinalidades

| Relacionamento | Cardinalidade | Como é implementado |
| :--- | :---: | :--- |
| tutor — animal | $1:N$ | `animal.id_tutor` |
| tutor — consentimento | $1:N$ | `consentimento.id_tutor` |
| especie — raca | $1:N$ | `raca.id_especie` |
| animal — alergia | $1:N$ | `alergia.id_animal` |
| animal — convenio | $N:N$ | Tabela `animal_convenio` |
| funcionario — veterinario | $1:0..1$ | Especialização: `veterinario.id_veterinario` é o próprio `id_funcionario` |
| veterinario — especialidade | $N:N$ | Tabela `veterinario_especialidade` |
| animal — agendamento | $1:N$ | `agendamento.id_animal` |
| agendamento — triagem | $1:0..1$ | `triagem.id_agendamento` único |
| agendamento — atendimento | $1:0..1$ | `atendimento.id_agendamento` único |
| atendimento — diagnostico | $N:N$ | Tabela `atendimento_diagnostico` |
| atendimento — prescricao — item_prescricao | $1:N$ e $1:N$ | `prescricao.id_atendimento`; `item_prescricao.id_prescricao` |
| atendimento — solicitacao_exame | $1:N$ | `solicitacao_exame.id_atendimento` |
| documento — solicitacao_exame | $1:N$ | `solicitacao_exame.id_documento` (um laudo pode cobrir vários exames) |
| animal — programa_cronico | $N:N$ | Tabela `inscricao_programa` |
| atendimento — fatura | $1:0..1$ | `fatura.id_atendimento` único |
| fatura — item_fatura / pagamento | $1:N$ | `item_fatura.id_fatura`; `pagamento.id_fatura` |
| fatura — nota_fiscal | $1:0..1$ | `nota_fiscal.id_fatura` único |
| insumo — lote — movimentacao_estoque | $1:N$ e $1:N$ | `lote.id_insumo`; `movimentacao_estoque.id_lote` |
| usuario — perfil | $N:N$ | Tabela `usuario_perfil` |
| usuario — log_auditoria | $1:N$ | `log_auditoria.id_usuario` |

### Normalização

* **1FN:** Todos os atributos são atômicos; o endereço é dividido em campos e não há listas dentro de uma coluna.
* **2FN:** Nas tabelas de chave composta, cada atributo depende da chave inteira (ex.: `tipo` em `atendimento_diagnostico` depende do par `atendimento` + `diagnóstico`).
* **3FN:** Nada derivado é gravado: o total da fatura e o saldo do estoque são calculados nas visões `vw_fatura_total` e `vw_saldo_lote`.
* **Redundância consciente:** `item_fatura.valor_unitario` repete o preço do serviço para preservar o valor cobrado no dia (RN11).

---

## 5. Dicionário de Dados

O dicionário detalha as seis tabelas centrais; as demais seguem o mesmo padrão e estão completas no script da seção 7.

### `tutor`

| Campo | Tipo | Obrigatório | Restrição | Descrição |
| :--- | :--- | :---: | :--- | :--- |
| `id_tutor` | INT (identidade) | Sim | PK | Identificador interno |
| `nome` | VARCHAR(120) | Sim | — | Nome completo do tutor |
| `cpf` | CHAR(11) | Sim | Único; só dígitos | Chave natural que impede cadastro duplicado |
| `data_nascimento` | DATE | Não | — | Data de nascimento do tutor |
| `telefone` | VARCHAR(15) | Sim | — | Contato principal, com DDD |
| `email` | VARCHAR(120) | Não | — | Contato e envio de laudos |
| `logradouro`, `numero`, `complemento`, `bairro`, `cidade` | VARCHAR | Não | — | Endereço |
| `uf` / `cep` | CHAR(2) / CHAR(8) | Não | — | Estado e CEP |
| `contato_validado_em` | DATE | Não | — | Última vez que a recepção confirmou telefone e endereço |
| `criado_em` / `atualizado_em` | TIMESTAMPTZ | Sim | Padrão `now()` | Controle do cadastro |
| `anonimizado_em` | TIMESTAMPTZ | Não | — | Quando os dados pessoais foram anonimizados pela política de descarte |

### `animal`

| Campo | Tipo | Obrigatório | Restrição | Descrição |
| :--- | :--- | :---: | :--- | :--- |
| `id_animal` | INT (identidade) | Sim | PK | Identificador do paciente |
| `id_tutor` | INT | Sim | FK → `tutor` | Responsável pelo animal |
| `id_especie` | INT | Sim | FK → `especie` | Cão, gato, equino, bovino etc. |
| `id_raca` | INT | Não | FK (`id_raca`, `id_especie`) → `raca` | Raça da mesma espécie; vazio = SRD |
| `nome` | VARCHAR(60) | Sim | Único por tutor e espécie, entre animais vivos | Nome do animal |
| `sexo` | CHAR(1) | Sim | M ou F | Sexo |
| `data_nascimento` | DATE | Não | — | Nascimento, real ou estimado |
| `nascimento_estimado` | BOOLEAN | Sim | Padrão falso | Indica se a data é estimada |
| `porte` | CHAR(1) | Sim | P, M ou G | Pequeno, médio ou grande porte |
| `microchip` | VARCHAR(15) | Não | Único | Identificação eletrônica |
| `castrado` | BOOLEAN | Não | — | Situação reprodutiva |
| `data_obito` | DATE | Não | — | Preenchida no óbito; o histórico é mantido |

### `agendamento`

| Campo | Tipo | Obrigatório | Restrição | Descrição |
| :--- | :--- | :---: | :--- | :--- |
| `id_agendamento` | INT (identidade) | Sim | PK | Identificador |
| `id_animal` | INT | Sim | FK → `animal` | Paciente |
| `id_veterinario`, `id_especialidade` | INT | Sim | FK composta → `veterinario_especialidade` | Veterinário e especialidade que ele atende |
| `id_unidade` | INT | Sim | FK → `unidade` | Clínica da rede |
| `data_hora` | TIMESTAMPTZ | Sim | Único por veterinário, exceto cancelados | Início marcado |
| `duracao_min` | SMALLINT | Sim | > 0; padrão 30 | Duração prevista |
| `modalidade` | VARCHAR(12) | Sim | `PRESENCIAL` ou `TELECONSULTA` | Tipo de consulta |
| `status` | VARCHAR(12) | Sim | `AGENDADO`, `CONFIRMADO`, `CANCELADO`, `FALTOU`, `REALIZADO` | Situação |
| `motivo` | VARCHAR(200) | Não | — | Motivo informado na marcação |
| `criado_por` | INT | Sim | FK → `usuario` | Quem marcou |
| `chegada_em` | TIMESTAMPTZ | Não | — | Chegada registrada na recepção; base do tempo de espera |

### `atendimento`

| Campo | Tipo | Obrigatório | Restrição | Descrição |
| :--- | :--- | :---: | :--- | :--- |
| `id_atendimento` | INT (identidade) | Sim | PK | Registro do prontuário |
| `id_agendamento` | INT | Sim | FK → `agendamento`; único | Agendamento que originou o atendimento |
| `id_veterinario` | INT | Sim | FK → `veterinario` | Quem de fato atendeu (pode diferir do agendado) |
| `inicio_em` | TIMESTAMPTZ | Sim | — | Início do atendimento |
| `fim_em` | TIMESTAMPTZ | Não | > `inicio_em` | Fim do atendimento |
| `anamnese` | TEXT | Sim | — | Histórico relatado pelo tutor |
| `exame_fisico` | TEXT | Não | — | Achados do exame clínico |
| `conduta` | TEXT | Não | — | Plano terapêutico e orientações |
| `finalizado` | BOOLEAN | Sim | Se verdadeiro, exige `fim_em` | Indica prontuário fechado |

### `pagamento`

| Campo | Tipo | Obrigatório | Restrição | Descrição |
| :--- | :--- | :---: | :--- | :--- |
| `id_pagamento` | INT (identidade) | Sim | PK | Identificador |
| `id_fatura` | INT | Sim | FK → `fatura` | Fatura quitada (total ou parcial) |
| `forma` | VARCHAR(15) | Sim | `PIX`, `CREDITO`, `DEBITO`, `DINHEIRO`, `BOLETO`, `CONVENIO` | Forma de pagamento |
| `valor` | NUMERIC(10,2) | Sim | > 0 | Valor recebido |
| `parcelas` | SMALLINT | Sim | 1 a 12; padrão 1 | Parcelamento no cartão |
| `codigo_autorizacao`| VARCHAR(40) | Não | — | NSU ou ID da transação; nunca o número do cartão |
| `id_convenio` | INT | Condicional | FK → `convenio`; obrigatório só se `forma = CONVENIO` | Operadora pagadora |
| `pago_em` | TIMESTAMPTZ | Sim | Padrão `now()` | Data e hora do recebimento |

### `log_auditoria`

| Campo | Tipo | Obrigatório | Restrição | Descrição |
| :--- | :--- | :---: | :--- | :--- |
| `id_log` | BIGINT (identidade) | Sim | PK | Identificador sequencial |
| `id_usuario` | INT | Não | FK → `usuario` | Quem agiu; vazio em login com usuário inexistente |
| `ocorrido_em` | TIMESTAMPTZ | Sim | Padrão `now()` | Momento do evento |
| `acao` | VARCHAR(12) | Sim | `LOGIN`, `LOGIN_FALHA`, `LOGOUT`, `LEITURA`, `INSERCAO`, `ALTERACAO`, `EXCLUSAO`, `EXPORTACAO` | Tipo de evento |
| `tabela` | VARCHAR(60) | Não | — | Tabela do registro afetado |
| `id_registro` | BIGINT | Não | — | ID do registro afetado |
| `dados_antes` / `dados_depois` | JSONB | Não | — | Conteúdo antes e depois da alteração |
| `ip_origem` | INET | Não | — | Endereço IP de origem |
| `justificativa` | VARCHAR(200) | Não | Obrigatória em acesso de emergência | Motivo do acesso excepcional |

---

## 6. Classificação dos Dados (LGPD) e Controle de Acesso

Dados pessoais do tutor e dados clínicos do animal recebem a mesma proteção: acesso só por perfil autorizado e registro de cada leitura e alteração.

### Classificação

| Categoria | Onde está no modelo | Por que proteger | Proteção no modelo |
| :--- | :--- | :--- | :--- |
| **Dados pessoais do tutor** | `tutor`, `consentimento` | Dado pessoal (LGPD, Art. 5º, I) | Acesso por perfil; consentimento por finalidade; anonimização (`anonimizado_em`) |
| **Dados clínicos do animal** | `triagem`, `atendimento`, `atendimento_diagnostico`, `evolucao`, `prescricao`, `item_prescricao`, `solicitacao_exame`, `documento`, `alergia` | Sigilo profissional do médico-veterinário; ligados a um tutor identificado | Só perfis clínicos; log de leitura e alteração; exclusão bloqueada |
| **Dados financeiros** | `fatura`, `item_fatura`, `pagamento`, `nota_fiscal`, `animal_convenio` | Dado pessoal; risco de fraude | Sem número de cartão; só código de autorização; acesso de financeiro e recepção |
| **Dados da equipe** | `funcionario` | Dado pessoal do colaborador | Diretoria administrativa e TI |
| **Credenciais** | `usuario.senha_hash` | Porta de entrada do sistema | Hash bcrypt ou Argon2; MFA; usuário desativado no desligamento |
| **Trilha de auditoria** | `log_auditoria` | Prova de conformidade | Somente inclusão; `UPDATE` e `DELETE` revogados |

### Perfis de Acesso (RBAC)

**Legenda:**
* **E** = Lê e edita
* **L** = Só lê
* **R** = Só relatórios agregados, sem identificar o tutor
* **—** = Sem acesso

| Perfil | Cadastro | Agenda | Prontuário | Exames | Financeiro | Estoque | Log |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Veterinário** | L | L | E | E | — | L | — |
| **Enfermagem** | L | L | E (triagem e evolução) | E (coleta) | — | E (baixa) | — |
| **Recepção** | E | E | — | L (status) | E (recebimento) | — | — |
| **Financeiro** | L | L | — | — | E | L | — |
| **Diretoria** | R | R | R | R | R | R | — |
| **Auditoria Interna** | L | L | L (com justificativa) | L | L | L | L |
| **Administrador de TI** | — | — | — | — | — | — | L |

> **Nota:** O administrador de TI gerencia usuários e perfis, mas não lê dados de negócio; um acesso técnico de emergência exige justificativa e fica registrado no log. As operadoras de convênio não têm login: recebem apenas os dados de faturamento exportados, e cada exportação é auditada no sistema.
