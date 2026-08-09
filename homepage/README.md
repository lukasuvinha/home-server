# Homepage

Dashboard de serviços. Um dos três dashboards em avaliação simultânea
(junto com Glance e Homarr) — ver `CLAUDE.md`. Configuração ainda com os
valores padrão do template (`config/bookmarks.yaml`, `config/services.yaml`
etc. não foram customizados).

## Portas

- `3000:3000`

## Dependências

- Containers: nenhum. Monta `/var/run/docker.sock` — comentário no compose
  descreve como opcional, para integrações Docker.
- Rede: nenhuma rede compartilhada declarada.
- Volumes do host: `./config:/app/config`.

## Variáveis de ambiente

- `HOMEPAGE_ALLOWED_HOSTS` — lista de hosts autorizados a acessar a UI;
  hoje contém `homepage.home` e `192.168.18.195:3000`.
- `PUID`/`PGID` aparecem comentados no compose — não estão ativos.

## Gotchas

- Se o hostname ou o IP do host mudar, `HOMEPAGE_ALLOWED_HOSTS` precisa ser
  atualizado, senão a interface recusa a requisição.
- `PUID`/`PGID` comentados — o container roda com o usuário padrão da
  imagem, não com o UID do host.
- Quando este dashboard for configurado de fato, `config/services.yaml` e
  `config/widgets.yaml` deixarão de ter conteúdo padrão e passarão a
  guardar API keys dos *arr em texto puro (ver `CLAUDE.md`, seção 5).
