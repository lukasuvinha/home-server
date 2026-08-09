# Setup do host

Fonte: leitura direta do host (`melchior`) em 2026-08-09. Onde o arquivo não
era legível pelo usuário `lukas`, está marcado.

## Usuário

- `lukas`, uid 1000, shell `/usr/bin/zsh`
- Grupos: `adm`, `cdrom`, `sudo`, `dip`, `plugdev`, `users`, `lxd`, `docker`
- Membro do grupo `sudo` (`getent group sudo` lista `lukas`)

## Rede — netplan

`/etc/netplan/00-installer-config.yaml` existe (e um `.bkp` ao lado), mas
ambos têm permissão `600` root — ilegível pelo usuário `lukas`.

<!-- VERIFICAR: conteúdo exato do netplan. O que segue é o estado efetivo
observado via `ip addr` / `ip route` / `resolvectl`, não o arquivo em si. -->

Estado efetivo:

- `enp2s0f0`: IP estático `192.168.18.195/24`, gateway padrão
  `192.168.18.1`
- `wlp3s0` existe como interface mas sem escopo de rota ativo no momento da
  checagem (sem uso)
- `resolvectl status`: `resolv.conf mode: stub`; DNS efetivo em `enp2s0f0`
  é `1.1.1.1` / `1.0.0.1` (não o Pi-hole) — ver
  `docs/002-host-nao-usa-pihole-como-dns.md` e
  `docs/2026-07-31-perda-de-dns-pos-apagao.md`

## fstab

```
/dev/disk/by-id/dm-uuid-LVM-... /              ext4  defaults        0 1
/dev/disk/by-uuid/...            /boot          ext4  defaults        0 1
/dev/disk/by-uuid/...            /boot/efi      vfat  defaults        0 1
/swap.img                        none           swap  sw              0 0
UUID=...                         /mnt/hd2tb     ext4  defaults,nofail 0 2
```

`/mnt/hd2tb` usa `nofail` — o boot não trava se o disco não estiver
presente.

## UFW

- `systemctl is-active ufw` → `active`
- `systemctl is-enabled ufw` → `enabled`
- Regras específicas (`/etc/ufw/user.rules`) têm permissão `640` root —
  ilegível pelo usuário `lukas`; `ufw status` exige root.

<!-- VERIFICAR: quais portas o UFW libera de fato? Não consegui ler
`/etc/ufw/user.rules` nem rodar `ufw status` sem senha de sudo interativa. -->

Fato observável sem UFW: as portas dos containers (ver READMEs de cada
stack) e a porta 22 (SSH) aparecem em `ss -tlnp` como escutando em
`0.0.0.0`/`[::]`. Se estão de fato acessíveis de fora da rede local depende
das regras do UFW e do roteador, nenhum dos dois verificável a partir daqui.

## SSH

- `sshd_config` é majoritariamente o padrão do Ubuntu — a maior parte das
  diretivas está comentada. Linhas ativas (não comentadas):
  `KbdInteractiveAuthentication no`, `X11Forwarding yes`, `PrintMotd no`,
  `UsePAM yes`, `AcceptEnv LANG LC_* COLORTERM NO_COLOR`,
  `Subsystem sftp /usr/lib/openssh/sftp-server`.
- `PasswordAuthentication` não aparece definida explicitamente no arquivo
  principal (fica no padrão do OpenSSH, que é `yes`).
- Existe `/etc/ssh/sshd_config.d/50-cloud-init.conf`, incluído antes do
  restante por ordem lexical, mas com permissão `600` root — ilegível.
  <!-- VERIFICAR: o que esse snippet do cloud-init define? Pode
  sobrescrever `PasswordAuthentication` ou `PermitRootLogin`. -->
- `/home/lukas/.ssh/authorized_keys` existe mas está vazio (0 bytes) —
  nenhuma chave pública autorizada para login por chave nesse usuário.
- `ssh.service` está com `enabled=disabled`, mas `ssh.socket` está
  `active`/`enabled` — o serviço sobe via ativação por socket, não
  diretamente no boot.

## Serviços do host (não containerizados)

| Serviço | Ativo | Habilitado no boot |
|---|---|---|
| `ssh` / `ssh.socket` | sim | socket: sim / serviço: não |
| `ufw` | sim | sim |
| `tailscaled` | sim | sim |
| `docker` | sim | sim |
| `cockpit.socket` | sim | sim |
| `systemd-resolved` | sim | sim |
| `systemd-networkd` | sim | sim |

`tailscale` (1.102.2) e `cockpit-ws` (360-1) estão instalados via pacote
`apt`/`dpkg`, não via snap (`snap list` não retorna nenhum pacote
instalado).

Docker: `29.7.2` / Docker Compose `v5.4.0`.
