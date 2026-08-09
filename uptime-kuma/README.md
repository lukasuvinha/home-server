# Uptime Kuma

Monitoramento de disponibilidade — checagens HTTP/ping. Não verifica
estado de container (ver `CLAUDE.md`).

## Portas

- `3001:3001`

## Dependências

- Containers: nenhum declarado no compose.
- Rede: nenhuma rede compartilhada declarada.
- Volumes do host: `./data:/app/data`.

## Variáveis de ambiente

Nenhuma declarada no compose.

## Gotchas

- `./data` contém um banco MariaDB completo (`data/mariadb/`), embora o
  compose não declare nenhum serviço de banco separado.
  `data/db-config.json` aponta para `"type": "embedded-mariadb"`.
  <!-- VERIFICAR: confirmar que esse MariaDB é gerenciado internamente
  pela própria imagem do Uptime Kuma 2.x, e não um serviço externo que
  ficou de fora do compose por engano. -->
- Sem `PUID`/`PGID`, sem `TZ`.
