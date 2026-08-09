# Backup

## Estado atual: não existe

Nenhum mecanismo de backup está configurado neste host. Verificado em
2026-08-09:

- Nenhuma ferramenta de backup instalada: `restic`, `borgbackup`/`borg` e
  `duplicity` ausentes (`which` não encontra nenhum). `rsync` está
  presente, mas é pacote padrão do Ubuntu Server — não há evidência de
  script ou rotina que o use para backup.
- `crontab -l` do usuário `lukas` não tem nenhum job de backup — as duas
  únicas linhas não comentadas do arquivo são para ligar/desligar o
  Snowflake em horário, e estão comentadas (inativas).
- `/etc/cron.d/` só tem os arquivos padrão do Ubuntu (`e2scrub_all` e um
  `.placeholder`).
- `systemctl list-timers --all` não mostra nenhum timer relacionado a
  backup — todos os listados são padrões do Ubuntu (`apt-daily`,
  `logrotate`, `fstrim`, `sysstat`, `man-db`, etc.).

Este é o maior gap conhecido da infraestrutura, conforme registrado no
`CLAUDE.md`.

## O que seria perdido em caso de falha de disco

Sem backup, uma falha do disco `sda` (sistema) ou `sdb` (`/mnt/hd2tb`)
perde, sem cópia em nenhum outro lugar:

- `pihole/etc-pihole/` — gravity (listas de bloqueio já processadas) e o
  banco de queries DNS (`pihole-FTL.db`)
- Bancos de dados das dashboards: `homarr/homarr/appdata/db/db.sqlite`,
  `uptime-kuma/data/mariadb/`, `beszel/beszel_data/`
- `nginx-proxy-manager/data/` — configuração dos proxy hosts e chaves de
  sessão
- Todo o conteúdo de `/mnt/hd2tb/media-server/` — biblioteca de mídia e
  configuração de Jellyfin/qBittorrent/Prowlarr/Sonarr/Radarr/Bazarr/Seerr
  (fora do escopo deste repositório, mas no mesmo disco físico `sdb`)

## Plano

Existe um plano: adicionar HDs externos dedicados a backup. Ainda não
foram configurados — sem hardware definido, sem ferramenta escolhida, sem
prazo.
