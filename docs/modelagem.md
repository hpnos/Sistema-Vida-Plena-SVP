# Modelagem dos dados

São dois modelos, um para cada banco.

## Banco operacional — `svp`

Modelo relacional normalizado até a 3ª forma normal, com [46] tabelas.

![DER](../diagramas/der-svp.png)

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

![Modelo estrela](../diagramas/estrela-svp-dw.png)

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
