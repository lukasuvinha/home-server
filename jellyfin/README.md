# Mediastack

O diretório se chama `jellyfin/` mas contém a stack de mídia inteira: um
único `docker-compose.yml` define sete serviços — Jellyfin, qBittorrent,
Prowlarr, Sonarr, Radarr, Bazarr e Seerr.

## Serviços e portas

| Serviço | Papel | Porta |
|---|---|---|
| `jellyfin` | player de mídia | `8096:8096` |
| `qbittorrent` | cliente de download | `8090:8080` (webui), `6881:6881` tcp/udp (torrent) |
| `prowlarr` | indexer central | `9696:9696` |
| `sonarr` | gerenciamento de séries | `8989:8989` |
| `radarr` | gerenciamento de filmes | `7878:7878` |
| `bazarr` | legendas | `6767:6767` |
| `seerr` | portal de pedidos | `5055:5055` |

## Dependências

- Rede: todos os sete serviços compartilham a rede `mediastack` (bridge),
  declarada no próprio compose.
- `depends_on` explícito: só `seerr`, que depende de `jellyfin`, `sonarr` e
  `radarr`. Os demais serviços não têm `depends_on` entre si no compose,
  mesmo havendo relação funcional entre eles (prowlarr alimenta sonarr e
  radarr, que entregam para bazarr).
- Volumes do host: cada serviço monta seu próprio
  `/mnt/hd2tb/media-server/<serviço>/config`. `jellyfin`, `sonarr` e
  `bazarr` montam `media/series` e `media/filmes`; `sonarr`, `radarr` e
  `qbittorrent` compartilham `media-server/downloads`.

## Variáveis de ambiente

- `jellyfin`, `qbittorrent`, `prowlarr`, `sonarr`, `radarr`, `bazarr`
  compartilham `PUID`, `PGID`, `TZ` via um bloco YAML ancorado
  (`x-common-env`). `qbittorrent` soma `WEBUI_PORT`.
- `seerr` não usa o bloco comum: define `LOG_LEVEL`, `TZ`, `PORT`
  separadamente.

## Gotchas

- `version: "3.8"` no topo do arquivo está obsoleto — o `docker compose`
  atual ignora essa chave e avisa no `config`/`up`.
- Comentário no início do compose orienta ajustar `PUID`/`PGID`/`TZ` e os
  caminhos de host antes de subir.
- `seerr` não recebe `PUID`/`PGID` — roda com o usuário padrão da própria
  imagem, diferente dos demais serviços da stack.
- `qbittorrent` publica a porta de torrent `6881` (tcp e udp) diretamente
  no host.
- Renomear o diretório `jellyfin/` para refletir que é a stack inteira é
  tarefa futura — não fazer agora (ver `CLAUDE.md`).
