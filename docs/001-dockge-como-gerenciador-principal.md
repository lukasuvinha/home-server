# ADR 001 — Dockge como gerenciador principal, Portainer retido

**Status:** aceito
**Data:** <!-- VERIFICAR: quando você decidiu -->

## Contexto

Os stacks do melchior são definidos em arquivos `docker-compose.yml`, um
por diretório em `~/docker`. Era preciso uma interface para gerenciá-los
sem depender exclusivamente da linha de comando.

## Decisão

Dockge como gerenciador principal. Portainer mantido em execução para
inspeção de containers e limpeza de imagens órfãs.

## Justificativa

<!-- VERIFICAR: o motivo real é seu. O que sei é que você escolheu Dockge
     como principal e manteve o Portainer como "opção mais completa".
     Preencha com o critério que pesou de fato. Pontos possíveis:
     - Dockge edita os compose files reais no disco; Portainer abstrai
       stacks para o próprio banco, afastando a UI da fonte da verdade
     - Composes em disco são versionáveis por git; stacks no banco do
       Portainer não são
     - Portainer resolve melhor inspeção pontual e limpeza de imagens -->

## Consequências

- A fonte da verdade de cada stack continua sendo o arquivo em disco,
  o que torna o repositório git significativo.
- Dois gerenciadores rodando consomem recursos em uma máquina com 8GB.
- Alterações feitas pelo Portainer podem divergir do arquivo em disco.
  <!-- VERIFICAR: isso já aconteceu? Vale registrar se sim. -->

## Alternativas descartadas

- **Só Portainer:** stacks vivendo no banco, não em arquivo.
- **Só linha de comando:** viável, mas sem visão rápida de estado.
