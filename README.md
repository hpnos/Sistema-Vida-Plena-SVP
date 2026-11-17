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

# A empresa

## Clínica Veterinária Vida Plena

Clínica Veterinária especializada em atendimento de animais de pequeno, médio e grande porte. Equipe profissional e qualificada em todos os setores da clínica para cumprir com os deveres e responsabilidades.


## Quem trabalha lá

- Veterinários
- Enfermagem
- Recepção
- Financeiro
- Diretoria
- Auditoria
- TI


## O problema

A clínica não tinha equipamentos de qualidade para garantir agilidade e precisão no dia-a-dia, fazer toda a guia do animal (com tamanho, peso, idade, altura, nome dos donos do animal) a mão, problemas de controle do estoque de medicamentos e materiais utilizados na clínica.

## O que o sistema faz

- Cadastro de tutores e animais
- Agenda
- Prontuário
- Pedido de exame e recebimento do resultado do laboratório
- Financeiro e emissão de nota
- Indicadores para a diretoria

## O que ficou de fora

Plano de saúde animal e pagamento online não fazem parte do projeto. A clínica recebe no balcão e o sistema só registra.


## Arquitetura


Resumindo o desenho: a equipe da clínica acessa a API, a API grava no PostgreSQL, um worker cuida da comunicação com o laboratório e com a emissão de nota, e um ETL leva os dados para um segundo banco usado só para relatórios. 

## Documentação

```
sistema-vida-plena/
├── README.md
├── docs/
│   ├── empresa.md
│   ├── arquitetura.md
│   ├── modelagem.md
│   ├── armazenamento.md
│   ├── seguranca.md
│   ├── integracao.md
│   └── etica-em-dados.md
└── diagramas/
    ├── arquitetura-final.pdf

```

| Documento | Assunto |
|---|---|
| empresa | Quem é a clínica e que problema o sistema resolve 
| arquitetura | Desenho geral e por que foi montado assim
| modelagem | Modelo relacional e modelo estrela 
| armazenamento | Bancos, documentos, arquivo físico e backup 
| seguranca | Acesso, criptografia e auditoria 
| integracao | Laboratório, nota fiscal e ETL 
| etica-em-dados | Discussão feita em sala 
