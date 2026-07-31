# Pomin Contábil — Relatórios de Revisão (públicos)

Este repositório é **público** e contém **apenas relatórios de revisão** (`RELATORIO_*.md`) do
projeto Pomin Contábil: descrições de alteração de cada lote e evidências técnicas (decisões de
arquitetura, provas de comportamento, checkpoints de validação).

## O que este repositório NÃO contém

- Código-fonte ou lógica de negócio
- Credenciais, senhas, tokens ou chaves de API
- Dados de clientes reais, dados pessoais ou sigilosos

Cada relatório é gerado ao final de um lote de trabalho e passa por uma varredura automática de
padrões sensíveis (senha/token/secret/api-key/bearer/chave privada/CNPJ/CPF/cartão) antes do push.
Se algo sensível for necessário para explicar uma mudança, ele é descrito de forma abstrata — nunca
o valor real.

## Índice

Os arquivos seguem o padrão `RELATORIO_<TEMA>.md`, um por tema/ciclo. Consulte a lista de arquivos
neste repositório; cada um traz data, escopo e evidências do respectivo lote.
