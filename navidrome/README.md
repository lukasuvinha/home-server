# Navidrome

Servidor de streaming de música. Instalado, sem biblioteca configurada
(ver `CLAUDE.md`).

## Portas

- `4533:4533`

## Dependências

- Containers: nenhum.
- Rede: nenhuma rede compartilhada declarada.
- Volumes do host: `/mnt/hd2tb/media-server/navidrome/data:/data`,
  `/mnt/hd2tb/media-server/musicas:/music:ro`.

## Variáveis de ambiente

Nenhuma ativa — o bloco `environment` (`ND_LOGLEVEL`) está comentado no
compose.

## Gotchas

- Usa `user: 1000:1000` diretamente no compose, não o padrão
  `PUID`/`PGID` via variável de ambiente.
- Sem `TZ` definida.
