# Rede e DNS

Fonte: composes dos stacks, `/etc/dnsmasq.d/02-wildcard.conf`,
`resolvectl status`, `tailscale status`, rodados no host em 2026-08-09.

## Pi-hole

- Container `pihole/`, imagem `pihole/pihole:2026.07.2`.
- DNS publicado só em `192.168.18.195:53` (tcp e udp), não em wildcard —
  ver `pihole/README.md`.
- Interface web em `192.168.18.195:8080`, atrás do NPM em
  `pihole.home`.
- Wildcard `.home`: `pihole/etc-dnsmasq.d/02-wildcard.conf` contém
  `address=/home/192.168.18.195` — qualquer nome `*.home` resolve para o
  IP do host.
- Distribuição para a rede: conforme `CLAUDE.md`, entregue via DHCP do
  roteador. <!-- VERIFICAR: não dá pra confirmar a config do DHCP do
  roteador a partir do filesystem deste host. -->

## O host não usa o próprio Pi-hole como resolvedor

- `resolvectl status` (link `enp2s0f0`): `Current DNS Server: 1.1.1.1`,
  com `1.0.0.1` e `fe80::1` como demais servidores. Não é
  `192.168.18.195`.
- `resolv.conf mode: stub` — `systemd-resolved` ativo no caminho de
  resolução do host.
- Decisão e motivo documentados em
  `docs/002-host-nao-usa-pihole-como-dns.md` (ADR). Incidente relacionado
  a essa configuração em `docs/2026-07-31-perda-de-dns-pos-apagao.md`.

## systemd-resolved — portas em escuta

- `192.168.18.195:53` — Pi-hole (container)
- `127.0.0.53:53` — stub padrão do `systemd-resolved`
- `127.0.0.54:53` — segundo listener do `systemd-resolved`. Propósito não
  identificado; não investigado.

## Tailscale

- Ativo no host (não containerizado), versão `1.102.2`, instalado via
  `apt`.
- IP no tailnet: `100.69.232.108` (interface `tailscale0`).
- DNS: `resolvectl status` mostra `tailscale0` com resolver
  `100.100.100.100`, domínio `tailb3a6d6.ts.net` (MagicDNS).
- Demais dispositivos no tailnet (`tailscale status`):

  | Dispositivo | SO | Estado |
  |---|---|---|
  | `melchior` (este host) | linux | ativo |
  | `galaxy-a52s-do-lukas` | android | offline (últ. visto há 10d) |
  | `ti-jmc-lnvs145` | windows | offline (últ. visto há 4d) |
  | `xcssetcodeghost` | linux | ativo |

- Decisão de usar Tailscale em vez de port forwarding documentada em
  `docs/003-tailscale-em-vez-de-port-forwarding.md` (ADR).

## Rede local

- Gateway: `192.168.18.1` (rota padrão via `enp2s0f0`)
- IP do host: `192.168.18.195/24`, fixo via netplan (arquivo não legível,
  ver `docs/01-host-setup.md`)
- Interface Wi-Fi `wlp3s0` existe no host mas sem rota/escopo DNS ativo no
  momento da checagem.
