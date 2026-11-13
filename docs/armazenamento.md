# Armazenamento

## Os dois bancos

| Banco | Para que serve | Quem usa | Modelo |
|---|---|---|---|
| `svp` | Operação do dia a dia: cadastro, agenda, prontuário, financeiro | Toda a equipe, pela API | Relacional, 3FN |
| `svp_dw` | Relatórios e indicadores | Diretoria | Estrela |

Os dois são PostgreSQL e ficam no mesmo servidor. O `svp_dw` só recebe dados pelo ETL; ninguém digita nada nele.

## Documentos digitais

Laudos e resultados de exame chegam em PDF. O arquivo fica em uma pasta do servidor (um volume do Docker) e o banco guarda o caminho, a data, quem anexou e o hash SHA-256. O hash serve para conferir depois que o arquivo não foi trocado.

Em produção a ideia é passar esses arquivos para um bucket privado.

## Arquivo físico

Nem tudo na clínica é digital.

| Material | Onde fica | Como é organizado | Por quanto tempo |
|---|---|---|---|
| Termos assinados pelo tutor (cirurgia, anestesia, internação, eutanásia) | [armário com chave na recepção] | [por número do prontuário] | [prazo] |
| Radiografias em filme e laudos impressos | [sala / armário] | [por animal e data] | [prazo] |
| Receitas de medicamento controlado (via da clínica) | [armário trancado, acesso só do veterinário responsável] | [por data] | [prazo] |
| Fichas de contingência (preenchidas quando o sistema está fora) | [pasta na recepção] | [por data] | Até serem lançadas no sistema, depois [prazo] |

Quando dá, o papel é digitalizado e anexado ao prontuário, e o original continua guardado. O sistema registra em que armário/pasta o original está, para ser possível achar.

[Conferir os prazos de guarda com a resolução do CFMV sobre prontuário.]

## Backup

- Backup completo do `svp` toda noite, copiado para fora do servidor [HD externo / nuvem].
- A pasta de documentos entra no mesmo backup.
- O `svp_dw` não precisa de backup próprio: dá para refazer rodando o ETL.
- Teste de restauração [uma vez por mês].

Para produção está previsto réplica do banco e restauração por ponto no tempo. No protótipo isso não está ativo.
