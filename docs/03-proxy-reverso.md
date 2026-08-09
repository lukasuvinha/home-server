# Proxy reverso — Nginx Proxy Manager

Fonte: `nginx-proxy-manager/docker-compose.yml` e
`nginx-proxy-manager/data/nginx/proxy_host/*.conf`, lidos em 2026-08-09.

## O que é

Proxy reverso com interface web (`jc21/nginx-proxy-manager`), expõe os
outros serviços sob subdomínios `*.home`.

## Portas

- `80` — HTTP
- `81` — interface de administração
- `443` — HTTPS

Conforme `CLAUDE.md`, o stack está **ativo em HTTP apenas, sem TLS
configurado e sem Access List**, mesmo com a porta 443 mapeada no compose.

## Mapeamento atual (19 proxy hosts)

Todos apontam para `192.168.18.195`, cada um numa porta diferente:

| Host | Destino |
|---|---|
| `pihole.home` | `:8080` |
| `npm.home` | `:81` |
| `jellyfin.home` | `:8096` |
| `sonarr.home` | `:8989` |
| `radarr.home` | `:7878` |
| `bazarr.home` | `:6767` |
| `seerr.home` | `:5055` |
| `qbittorrent.home` | `:8090` |
| `prowlarr.home` | `:9696` |
| `beszel.home` | `:8095` |
| `homarr.home` | `:7575` |
| `glance.home` | `:8075` |
| `homepage.home` | `:3000` |
| `wylan.home` | `:8840` |
| `dockge.home` | `:5001` |
| `uptimekuma.home` | `:3001` |
| `navidrome.home` | `:4533` |
| `filebrowser.home` | `:8082` |
| `portainer.home` | `:9000` |

Todos os destinos são `192.168.18.195` — o próprio host `melchior`, já
que todos os serviços rodam nele.

## Dados e certificados

- `./data` — configuração dos proxy hosts, banco sqlite, chaves de sessão
  (`keys.json`).
- `./letsencrypt` — diretório para certificados; hoje sem certificado
  ativo, consistente com o stack estar em HTTP apenas.
- O container roda sem `user:`/`PUID`/`PGID` — os arquivos gerados em
  `./data` e `./letsencrypt` ficam com dono `root` no host (ver
  `nginx-proxy-manager/README.md`).
