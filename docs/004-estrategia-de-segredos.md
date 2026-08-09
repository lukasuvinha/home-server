# ADR 004 — Segredos em `.env`, repositório único, whitelist no `.gitignore`

**Status:** aceito
**Data:** 2026-08-09

## Contexto

O repositório precisa cumprir dois papéis conflitantes: reconstruir o
servidor a partir de um clone, e ser publicável como portfólio. Os
compose files continham credenciais escritas diretamente.

A primeira tentativa de versionamento capturou 588 arquivos com chaves
privadas, senhas e bancos de dados — ver postmortem de 2026-08-09.

## Decisão

Repositório único em `~/docker`. Compose files usam `${VARIAVEL}`, com
os valores reais em `.env` por stack, ignorados pelo git. Cada stack
versiona um `.env.example` com placeholders vazios.

O `.gitignore` adota estratégia de **whitelist**: ignora tudo, libera
explicitamente compose files, markdown e arquivos `.example`.

Dados de serviço (bancos, logs, estado) ficam fora do git e são cobertos
por backup dedicado.

## Justificativa

O mesmo compose file serve aos dois papéis sem alteração. Um clone mais
o preenchimento dos `.env` reconstrói o servidor; o repositório publicado
não contém segredo algum.

A whitelist foi escolhida sobre a blacklist porque cada serviço novo
inventa arquivos de configuração novos. Uma lista de exclusão exige
revisão a cada adição e falha silenciosamente; uma lista de inclusão
falha fechada.

## Consequências

- Os `.env` não estão versionados e precisam de proteção própria
  (tarball local; futuramente restic).
- Arquivos legitimamente publicáveis podem ficar de fora por omissão
  na whitelist. Correção consciente, não automática.
- Compose files publicados são exatamente os que rodam em produção,
  sem etapa de sanitização — não há divergência entre o documentado e
  o executado.

## Alternativas descartadas

- **Dois repositórios**, um privado com segredos e um público
  sanitizado: duplicação, sincronização manual, e o público divergiria
  do que roda de fato.
- **Repositório privado no GitHub:** não resolve o armazenamento de
  segredos em terceiro, e o histórico acompanha uma eventual mudança
  de visibilidade.
- **Blacklist no `.gitignore`:** falha aberta diante do desconhecido.
