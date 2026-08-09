# Glance

Dashboard com widgets (RSS, clima, mercado, releases do GitHub etc.). Um
dos três dashboards em avaliação simultânea (junto com Homepage e Homarr) —
ver `CLAUDE.md`.

## Portas

- `8075:8080`

## Dependências

- Containers: nenhum. O bind mount do `docker.sock` para o widget de
  containers Docker está comentado no compose, ou seja, desabilitado.
- Rede: nenhuma rede compartilhada declarada.
- Volumes do host: `./config:/app/config`, `./assets:/app/assets`,
  `/etc/localtime:/etc/localtime:ro`.

## Variáveis de ambiente

- Usa `env_file: .env`. Variável presente: `MY_SECRET_TOKEN`.

## Gotchas

- Sem `PUID`/`PGID`.
- <!-- VERIFICAR: `MY_SECRET_TOKEN` está definida no `.env`, mas não
  encontrei `${MY_SECRET_TOKEN}` referenciada em `config/glance.yml` nem
  `config/home.yml`. É usada em algum widget ou é sobra do template
  padrão do Glance? -->
