# Snowflake

Proxy voluntário do projeto Tor (`snowflake-proxy`). Retransmite tráfego
de usuários em regiões sob censura (ver `CLAUDE.md`).

## Portas

Nenhuma publicada explicitamente — `network_mode: host`.

## Dependências

- Containers: nenhum.
- Rede: nenhuma; usa a rede do host diretamente.
- Volumes: nenhum.

## Variáveis de ambiente

Nenhuma. A capacidade é limitada via `command` (`-capacity 10`), não por
variável de ambiente.

## Gotchas

- `network_mode: host`.
- `version: '3.8'` no topo do arquivo está obsoleto para o `docker compose`
  atual.
