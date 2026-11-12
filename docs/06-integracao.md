# Integração

O sistema tem duas integrações com parceiros de fora e uma interna.

| Integração | Com quem | Direção | Situação |
|---|---|---|---|
| Exames | Laboratório [nome do laboratório] | Vai o pedido, volta o resultado | Feita no protótipo, com um laboratório simulado |
| Nota fiscal de serviço (NFS-e) | Sefaz do município de [cidade] | Vai o atendimento faturado, volta a nota autorizada | Prevista |
| ETL | Entre `svp` e `svp_dw` | Só ida | Feita no protótipo |

Pagamento online e plano de saúde não fazem parte.

## Laboratório

1. O veterinário registra o pedido de exame no atendimento.
2. A API grava o pedido e, na mesma transação, coloca uma mensagem na fila de envio.
3. O worker pega a mensagem e envia para a API do laboratório.
4. Se o laboratório não responder, o worker tenta de novo, com intervalo cada vez maior.
5. Quando o exame fica pronto, o laboratório chama a API da clínica (webhook) com o resultado e o laudo.
6. A API confere a assinatura, grava o resultado no prontuário e guarda o PDF.

Dois cuidados:

- Cada pedido vai com uma chave única. Se o worker mandar o mesmo pedido duas vezes por causa de uma falha de rede, o laboratório percebe e não duplica.
- O pedido só entra na fila se foi gravado no banco. Não existe pedido enviado sem registro, nem registro que ficou sem enviar.

Como não há um laboratório de verdade no trabalho, foi feito um serviço pequeno que faz o papel dele: recebe o pedido e, depois de um tempo, devolve um resultado.

## Nota fiscal

Consulta, exame, cirurgia e internação são serviço, então a nota é a NFS-e, que é autorizada pela Secretaria da Fazenda do município (Sefaz [cidade]). O sistema não emite a nota sozinho: ele envia os dados e quem autoriza é a Sefaz.

1. O financeiro fecha o atendimento.
2. A API grava o faturamento e coloca na fila de envio um pedido de emissão.
3. O worker monta o arquivo da nota (XML), assina com o certificado digital da clínica e envia para o webservice da Sefaz.
4. A Sefaz devolve o número da nota e o código de verificação, ou o motivo da rejeição.
5. O sistema guarda o número, o XML autorizado e o PDF da nota junto do lançamento financeiro.

Se a Sefaz estiver fora do ar, o pedido fica na fila e o worker tenta de novo, do mesmo jeito que faz com o laboratório.

O que a clínica precisa ter para isso funcionar:

- cadastro no município como prestadora de serviço e autorização para emitir NFS-e;
- certificado digital (e-CNPJ), guardado junto com as outras chaves do sistema (ver [05-seguranca.md](05-seguranca.md));
- [código do serviço e alíquota de ISS usados pela clínica].

A venda de produto no balcão (ração, medicamento) usa outra nota, a NFC-e, que é da Sefaz do estado. Isso não está no escopo do projeto.

## ETL

- Feito em Python, executado manualmente [ou: toda noite].
- Lê do `svp` os registros dos últimos 90 dias e atualiza fatos e dimensões do `svp_dw`.
- Dados que identificam o tutor (nome, CPF) não vão para o `svp_dw`: [são trocados por um código gerado com HMAC]. Para indicador não interessa quem é a pessoa.
