# Segurança

## Quem acessa o quê

O acesso é por perfil. Cada usuário tem um perfil e a API confere a permissão em toda requisição.

| Perfil | Pode | Não pode |
|---|---|---|
| Veterinário | Ler e escrever prontuário, pedir exame, prescrever | Ver financeiro |
| Enfermagem | Ler prontuário, registrar medicação e evolução | Alterar diagnóstico e prescrição |
| Recepção | Cadastro e agenda | Ler prontuário |
| Financeiro | Contas, pagamentos e notas | Ler prontuário |
| Diretoria | Indicadores | Alterar dados |
| Auditoria | Ler a trilha de auditoria | Alterar dados do prontuário e etc.|
| TI | Gerenciar usuários e perfis | Ler prontuário |


## Login

- Senha guardada com hash Argon2id; a senha em si nunca fica no banco.
- Depois do login a API entrega um token (JWT) que vale por [tempo].
- Segundo fator (MFA) opcional.
- A sessão pode ser encerrada pelo administrador, por exemplo quando um funcionário sai da clínica.

## Dados em trânsito

| Trecho | Proteção |
|---|---|
| Navegador da equipe → API | HTTPS (TLS) |
| API/worker → laboratório | HTTPS (TLS) |
| Laboratório → API (resultado) | HTTPS (TLS) + assinatura HMAC |
| ETL → `svp_dw` | Rede interna do No protótipo tudo roda local em HTTP, então por enquanto a única proteção
no retorno do laboratório é a assinatura HMAC. O TLS entra na implantação.

A assinatura serve para a API ter certeza de que o resultado veio do
laboratório e não foi alterado no caminho.

A API rejeita o que estiver fora de uma janela de poucos
minutos, para evitar reenvio de resultado antigo ou repetido.


## Dados guardados

- O usuário de banco que a API usa só tem permissão nas tabelas de que precisa.
- Os laudos têm hash SHA-256 registrado.
- As senhas de banco e chaves ficam em um arquivo `.env` fora do repositório. Em produção, em um cofre de segredos.

## Auditoria

Fica registrado quem fez o quê, quando e o que mudou. Serve para prontuários, medicamentos ou não controlados (para ver quem registrou a saída, quantidade da dose e quem administrou a medicação), o financeiro, exames (para verificar a data do resultado e qual a previsão do resultado sair). A auditoria é responsável por todas essas funções e apenas ela que tem todo o controle disso tudo.
