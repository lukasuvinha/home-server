# Homarr

Dashboard com suporte a integrações com outros serviços. Um dos três
dashboards em avaliação simultânea (junto com Homepage e Glance) — ver
`CLAUDE.md`.

## Portas

- `7575:7575`

## Dependências

- Containers: nenhum. Monta `/var/run/docker.sock` — comentário no compose
  descreve como opcional, para integração com Docker.
- Rede: nenhuma rede compartilhada declarada.
- Volumes do host: `./homarr/appdata:/appdata` (banco sqlite e Redis
  embutido).

## Variáveis de ambiente

- Usa `env_file: .env`. Variável presente: `HOMARR_SECRET_ENCRYPTION_KEY`
  (chave usada pelo Homarr para criptografar credenciais de integrações
  salvas no próprio banco).

## Gotchas

- O `docker.sock` montado dá ao container visibilidade e controle sobre os
  containers do host.
- Sem `PUID`/`PGID` — roda com o usuário padrão da imagem.
- O volume `./homarr/appdata` cria aninhamento `homarr/homarr/appdata/`,
  já que o compose está dentro de um diretório também chamado `homarr/`.
