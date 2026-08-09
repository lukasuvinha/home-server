# Portainer

Interface de gerenciamento de containers Docker. Uso secundário neste
homelab: inspeção de containers e limpeza de imagens órfãs (ver
`CLAUDE.md`).

## Portas

- `9443:9443` — HTTPS
- `9000:9000` — HTTP

## Dependências

- Containers: nenhum.
- Rede: define rede própria `portainer_network` (nome fixo).
- Volumes: `/var/run/docker.sock` (bind mount), `portainer_data` (volume
  nomeado, gerenciado pelo Docker — não é bind mount para o diretório do
  stack).

## Variáveis de ambiente

Nenhuma declarada no compose.

## Gotchas

- `restart: always`, diferente da maioria dos outros stacks, que usam
  `unless-stopped`.
- O `docker.sock` montado dá ao container controle total sobre o Docker
  do host.
- `portainer_data` é volume nomeado, não bind mount — os dados não ficam
  visíveis dentro de `~/docker/portainer/` e não são removidos por um
  `docker compose down` normal (só com `-v`).
