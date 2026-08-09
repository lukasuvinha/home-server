# ADR 005 — Git local descartável como snapshot antes de refatoração assistida por IA

**Status:** aceito
**Data:** 2026-08-09

## Contexto

A extração de segredos dos compose files seria executada por um agente de
IA (Claude Code) com acesso de escrita e execução no servidor. Os serviços
estão em produção e em uso diário.

Isso exigia uma forma de reverter alterações. Mas o repositório destinado
à publicação não pode conter os segredos que precisavam estar protegidos
durante a operação — o repositório final e a rede de segurança da
operação têm requisitos opostos.

## Decisão

Dois repositórios git em sequência, no mesmo diretório:

1. **Snapshot descartável.** `git init` sem exclusões relevantes,
   capturando o estado completo, inclusive segredos. Sem remote.
2. Executada a refatoração e validado o resultado, o histórico é
   preservado em `git bundle` fora do diretório e o `.git` é removido.
3. **Repositório de publicação.** Novo `git init` com o `.gitignore`
   whitelist, nascendo sem segredos no histórico.

## Justificativa

O histórico do git é imutável: limpar arquivos em um commit posterior não
os remove dos anteriores. Um repositório que já contém segredos nunca
poderá ser publicado sem reescrita de histórico.

Descartar o `.git` resolve isso sem `filter-repo`: o histórico sujo deixa
de existir, e o repositório de publicação nasce limpo desde o primeiro
commit.

O `git bundle` preserva o snapshot para consulta posterior, cobrindo o
caso de uma regressão só percebida dias depois.

## Consequências

- O repositório publicado não tem histórico anterior à refatoração.
  Aceitável: o histórico útil começa na configuração já organizada.
- Existe uma janela em que o `.git` contém segredos. Mitigada por não
  configurar remote e pela vida curta do repositório.
- Requer disciplina de sequência. Um `git remote add` durante a fase 1
  anularia a decisão inteira.

## Alternativas descartadas

- **`git filter-repo` após o fato:** exige rotação de todas as
  credenciais e não garante remoção de objetos em clones existentes.
- **Publicar sem snapshot:** deixaria a refatoração sem reversão.
- **Snapshot só por tarball:** perde granularidade de `git diff`, que é
  justamente o que permite revisar o que o agente alterou.
