# Convenção de stacks

Fonte: os 14 `docker-compose.yml` de `~/docker`, lidos em 2026-08-09.

## Estrutura de diretórios

- Um diretório por stack, direto em `~/docker`, sem aninhamento — estrutura
  plana. Cada um com seu próprio `docker-compose.yml`.
- Exceção documentada: `jellyfin/` contém a mediastack inteira (sete
  serviços), não só o Jellyfin — ver `jellyfin/README.md`.
- Dois stacks têm um aninhamento acidental por causa de caminho relativo
  no compose (`./nome-do-stack/...` de dentro de um diretório já chamado
  assim): `filebrowser/filebrowser/` e `homarr/homarr/`.

## Dockge

- Gerenciador principal dos stacks (`dockge/docker-compose.yml`).
- Monta `/home/lukas/docker` no **mesmo caminho** dentro e fora do
  container, e usa `DOCKGE_STACKS_DIR=/home/lukas/docker` para localizar
  os stacks. Mover o repositório no host exige atualizar os dois.
- Monta `/var/run/docker.sock` — controla o Docker do host diretamente.

## PUID/PGID

Não há convenção única aplicada a todos os stacks. O que existe hoje:

| Padrão | Stacks |
|---|---|
| `PUID=1000`/`PGID=1000` via variável de ambiente | `dockge`, `filebrowser`, e (via bloco `x-common-env` compartilhado) `jellyfin`, `qbittorrent`, `prowlarr`, `sonarr`, `radarr`, `bazarr` |
| `user: 1000:1000` direto no compose (não variável) | `navidrome` |
| Sem `PUID`/`PGID` definido | `beszel`, `glance`, `homarr`, `homepage` (aparece comentado, inativo), `nginx-proxy-manager`, `pihole`, `portainer`, `seerr`, `snowflake`, `uptime-kuma`, `watchyourlan` |

Nos stacks sem `PUID`/`PGID`, os containers que não declaram `user:`
rodam com o usuário padrão da própria imagem — em várias delas (NPM,
Beszel), isso é root, e os arquivos gerados nos bind mounts ficam com
dono `root` no host (detalhado nos READMEs de cada stack).

## Redes

- Só a mediastack (`jellyfin/`) declara uma rede compartilhada
  (`mediastack`, bridge) entre seus próprios sete serviços.
- Nenhum outro stack compartilha rede com outro. Cada um sobe na rede
  bridge padrão que o Compose cria automaticamente para o projeto
  (`<nome-do-stack>_default`).
- Consequência observável (`ip addr` no host): mais de dez redes bridge
  `172.x.0.0/16` isoladas, uma por stack, além da `mediastack`.

## Segredos

- `pihole`, `homarr` e `beszel` usam `${VARIAVEL}` no compose, com
  `env_file: .env` no serviço — `.env` com o valor real (fora do git),
  `.env.example` com o nome da variável e placeholder vazio.
- `glance` já tinha um `.env` com `env_file: .env` antes dessa extração,
  mas sem `.env.example` correspondente.
- Os demais dez stacks não têm credencial escrita no compose atualmente.
