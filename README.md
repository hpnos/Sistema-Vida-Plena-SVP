# Sistema Vida Plena (SVP)

Documentação e protótipo do Sistema Vida Plena, da Vida Plena Clínicas Veterinárias.

## Integrantes

| Nome |
|--- |
| Levi Falcão | 
| Jonatas Paixão | 
| Saulo de Melo | 
| Joabe Irlan  | 
| Otávio vinícius | 
| Moisés Oliveira |
| Rafael Santos | 

# Sistema Vida Plena (SVP)


O SVP é o sistema de uma clínica veterinária fictícia, a Vida Plena. Ele cobre cadastro de tutores e animais, agenda, prontuário, pedidos de exame e financeiro. Aqui no repositório está a parte de dados do projeto: como a clínica guarda, protege, integra e analisa as informações.

## Arquitetura


Resumindo o desenho: a equipe da clínica acessa a API, a API grava no PostgreSQL, um worker cuida da comunicação com o laboratório e com a emissão de nota, e um ETL leva os dados para um segundo banco usado só para relatórios. 

## Documentação

```
sistema-vida-plena/
├── README.md
├── Modelagem de Dados — Sistema Vida Plena (SVP) (1).pdf
├── docs/
│   ├── 01-empresa.md
│   ├── 02-arquitetura.md
│   ├── 03-modelagem.md
│   ├── 04-armazenamento.md
│   ├── 05-seguranca.md
│   ├── 06-integracao.md
│   └── 07-etica-em-dados.md
└── diagramas/
    ├── arquitetura-final.pdf

```

| Documento | Assunto |
|---|---|
| 01-empresa | Quem é a clínica e que problema o sistema resolve 
| 02-arquitetura | Desenho geral e por que foi montado assim
| 03-modelagem | Modelo relacional e modelo estrela 
| 04-armazenamento | Bancos, documentos, arquivo físico e backup 
| 05-seguranca | Acesso, criptografia e auditoria 
| 06-integracao | Laboratório, nota fiscal e ETL 
| 07-etica-em-dados | Discussão feita em sala 
