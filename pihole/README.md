# Pi-hole

Servidor DNS com bloqueio de anúncios (sinkhole) e interface web de
administração.

## Portas

- `192.168.18.195:53/tcp` e `192.168.18.195:53/udp` — DNS, publicado apenas
  nesse IP, não em wildcard (`0.0.0.0`)
- `8080:80/tcp` — interface web

## Dependências

- Containers: nenhum.
- Rede: nenhuma rede compartilhada declarada; usa a rede padrão do Compose.
- Volumes do host: `./etc-pihole:/etc/pihole` (config, gravity, banco de
  queries), `./etc-dnsmasq.d:/etc/dnsmasq.d` (config extra do dnsmasq).

## Variáveis de ambiente

- `TZ`
- `FTLCONF_webserver_api_password` (via `.env`, ver `.env.example`)

## Gotchas

- `cap_add: NET_ADMIN` está declarado no compose.
- Porta 53 publicada só no IP do host, não em wildcard — ver
  `docs/2026-07-31-perda-de-dns-pos-apagao.md` para o incidente relacionado
  a bind de porta DNS.
- Sem `PUID`/`PGID` — a imagem oficial do Pi-hole não usa esse padrão.
- Precisa de `.env` (não versionado) para subir com a senha correta da API
  web; sem ele, `FTLCONF_webserver_api_password` fica vazio.
