# Arquitetura

![Arquitetura final](../diagramas/arquitetura-final.png)

## Como ler o desenho

O desenho tem três partes.

**Acesso e integrações.** A equipe da clínica usa o sistema pelo navegador, que conversa com a API. A API é quem valida o usuário e decide o que cada perfil pode fazer. Quando precisa falar com alguém de fora (laboratório, nota fiscal), ela não chama direto: deixa o pedido numa fila e um worker faz o envio.

**Persistência.** Tudo o que é do dia a dia fica no banco `svp`. Os laudos em PDF ficam em uma pasta separada e o banco guarda só a referência e o hash do arquivo.

**Análise.** Um ETL copia os dados do `svp` para o `svp_dw`, que é organizado para relatório. Assim a diretoria consulta indicadores sem pesar o banco que a recepção está usando.

## Componentes

| Componente | O que faz | Tecnologia |
|---|---|---|
| API | Regras do sistema, login e permissões | Python / FastAPI |
| Banco operacional (`svp`) | Cadastros, agenda, prontuário, financeiro | PostgreSQL 16 |
| Worker | Envia pedidos para fora e tenta de novo se falhar | Python, fila no próprio PostgreSQL |
| Documentos | Laudos em PDF | Volume em disco |
| ETL | Leva dados do `svp` para o `svp_dw` | Python |
| Banco analítico (`svp_dw`) | Indicadores | PostgreSQL |

## Por que tudo roda em um servidor só

A Vida Plena é uma clínica pequena, sem equipe de infraestrutura. Por isso a proposta é um único servidor dentro da clínica, com cada componente em um contêiner (Docker Compose). Sai barato, é simples de instalar e o sistema continua funcionando se a internet cair — a única coisa que depende de internet é a comunicação com o laboratório e a emissão de nota, e essas ficam na fila até a conexão voltar.

O ponto fraco é conhecido: se esse servidor parar, para tudo. O ideal seria cada serviço em uma máquina. Para o tamanho da clínica, a escolha foi aceitar esse risco e compensar com:

- backup diário guardado fora do servidor (ver [04-armazenamento.md](04-armazenamento.md));
- nobreak no servidor;
- plano de contingência em papel: [descrever — ficha de atendimento impressa, quem lança no sistema depois e em quanto tempo].

## Protótipo x produção

O que foi montado para a disciplina é um protótipo. Réplica do banco, restauração por ponto no tempo, HTTPS e cofre de senhas estão descritos como plano para produção e aparecem tracejados no desenho; não estão rodando.
