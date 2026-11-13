[armazenamento_atualizado.md](https://github.com/user-attachments/files/34817049/armazenamento_atualizado.md)
# Armazenamento de Dados — Sistema Vida Plena (SVP)

Este documento detalha a arquitetura de armazenamento de dados, diretrizes de backup e políticas de retenção física e digital do Sistema Vida Plena (SVP).

## 1. Bancos de Dados (PostgreSQL 16)

A arquitetura utiliza dois bancos segregados no mesmo cluster para separar as cargas de trabalho OLTP e OLAP.

| Banco | Carga de Trabalho | Acesso | Modelagem |
| :--- | :--- | :--- | :--- |
| `svp` | Operacional (cadastros, agenda, prontuários, financeiro) | API (leitura/escrita via RBAC) | Relacional (3FN), 46 tabelas. |
| `svp_dw` | Analítico (relatórios e dashboards gerenciais) | Diretoria (leitura), ETL (escrita) | Dimensional (Star Schema): 3 Fatos, 6 Dimensões. |

- **Integridade (`svp`):** Validação nativa no banco (Foreign Keys, CHECK, UNIQUE).
- **Pipeline ETL (`svp_dw`):** Desenvolvido em Python, sincroniza dados do `svp` para o `svp_dw`. No protótipo, a execução é manual via Docker Compose. Janela de processamento de 90 dias, com validação via HMAC.

## 2. Documentos Digitais (PDF)

Os arquivos binários não são salvos no banco. O banco armazena apenas os metadados e o ponteiro para o sistema de arquivos.

- **Storage Protótipo:** Volume Docker local.
- **Storage Produção:** Bucket privado (Object Storage).
- **Metadados registrados no banco:** Identificação, nome original, tipo, paciente/prontuário, timestamp de upload, usuário, path de armazenamento e Hash SHA-256 (para auditoria de integridade do arquivo).

## 3. Arquivo Físico

O sistema atua como indexador do arquivo físico, registrando a localização exata (armário/pasta) de cada documento original para agilizar a busca. A guarda segue a Resolução CFMV nº 1.321/2020 (atualizada pela 1.653/2025): retenção mínima de 5 anos após o último atendimento.

| Documento | Localização Física | Regra de Guarda |
| :--- | :--- | :--- |
| **Termos de consentimento** | Armário de arquivo (trancado) | 5 anos pós-último atendimento. |
| **Radiografias e Laudos** | Armário clínico | Segue o prazo do prontuário vinculado. |
| **Receitas Controladas** | Armário restrito ao veterinário | Conforme legislação do medicamento. |
| **Fichas de contingência** | Recepção (temporário) | Anexadas ao prontuário após digitação no sistema. |

> **Nota:** A digitalização dos documentos não substitui a guarda obrigatória do documento físico original.

## 4. Estratégia de Backup e Disaster Recovery (DR)

- **Escopo:** Dump do banco `svp` + cópia do diretório de documentos digitais. *(O `svp_dw` não possui backup no protótipo, pois pode ser recriado via pipeline ETL).*
- **Armazenamento:** Off-site (storage cloud ou HD externo com criptografia).
- **Retenção Atual:** Diário mantido por 30 dias; Mensal mantido por 6 meses.
- **Validação:** Teste de restore executado mensalmente.
- **Roadmap de Produção:** Implementação de replicação, Point-in-Time Recovery (PITR) via WAL, snapshots de storage e backups imutáveis.

## 5. Segurança e Acesso

- **Aplicação:** Controle de acesso baseado em papéis (RBAC) validado pela API.
- **Banco de Dados:** Controle granular de privilégios nativo do PostgreSQL (roles).
- **Trânsito:** HTTPS/TLS obrigatório em produção.
- **Gestão de Segredos:** O protótipo injeta credenciais via variáveis de ambiente (`.env`). A versão de produção migrará obrigatoriamente para um gerenciador de segredos (Vault/KMS).
