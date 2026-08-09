# CLAUDE.md

Contexto e regras para o Claude Code neste repositório.

---

## 1. Contexto

Homelab pessoal. Servidor único chamado **melchior** — notebook Acer Aspire
E1-571 (i5-3230M, 8GB DDR3) reaproveitado, rodando **Ubuntu Server 26.04 LTS**.
IP fixo `192.168.18.195` via netplan, sem reserva no DHCP.

Nomenclatura MAGI (Neon Genesis Evangelion): melchior (atual),
balthazar (produção, futuro), casper (NAS, futuro). Balthazar e casper
**não existem** — são planos. Nunca documente como se existissem.

Este diretório (`~/docker`) é a raiz de todos os stacks Docker.
Um subdiretório por stack, cada um com seu `docker-compose.yml`.
Gerenciados via Dockge.

Os serviços estão **em produção e em uso diário**. Não são descartáveis.

O repositório será publicado no GitHub como portfólio técnico. A audiência
é dupla: recrutadores de infraestrutura e amigos técnicos. A documentação
deve ser honesta sobre o que está incompleto — isso vale mais do que
parecer terminado.

---

## 2. Estado real dos serviços

Fonte da verdade. Se encontrar algo diferente no filesystem, me avise em
vez de assumir que a tabela está desatualizada.

| Stack | Diretório | Estado |
|---|---|---|
| Pi-hole | `pihole/` | Ativo. DNS wildcard `.home` apontando para .195. Entregue à casa toda via DHCP do roteador |
| Nginx Proxy Manager | `nginx-proxy-manager/` | Ativo, **HTTP apenas**. Sem TLS, sem Access List |
| Mediastack | `jellyfin/` | Ativo. **O diretório se chama `jellyfin` mas contém a stack inteira**: Jellyfin, qBittorrent (8090:8080), Prowlarr, Sonarr, Radarr, Bazarr, Seerr. Rede compartilhada `mediastack`. Renomear o diretório é tarefa futura — não faça agora |
| Dockge | `dockge/` | Ativo. Gerenciador principal de stacks |
| Portainer | `portainer/` | Ativo. Uso secundário: inspeção de containers e limpeza de imagens |
| Homepage | `homepage/` | **Em avaliação.** Instalado, configuração padrão |
| Beszel | `beszel/` | Ativo. Monitoramento de recursos |
| Uptime Kuma | `uptime-kuma/` | Ativo, mas **básico**: só checagens HTTP/ping. Não verifica estado de container |
| Filebrowser | `filebrowser/` | Ativo |
| Glance | `glance/` | **Em avaliação.** Instalado |
| Homarr | `homarr/` | **Em avaliação.** Instalado |
| Snowflake | `snowflake/` | Ativo. Proxy voluntário do projeto Tor (`snowflake-proxy`). Retransmite tráfego de usuários em regiões sob censura |
| Navidrome | `navidrome/` | **Instalado, não configurado.** Container sobe, sem biblioteca de música |
| WatchYourLAN | `watchyourlan/` | **Instalado, não configurado.** Funcionou por um período e parou; causa não investigada |
| Cockpit | (host, não Docker) | **Instalado, não configurado** |
| Tailscale | (host, não Docker) | Ativo. Instalado no host, não containerizado |

**Três dashboards em avaliação simultânea** (Homepage, Glance, Homarr).
Não é redundância por escolha — é comparação em andamento para decidir
qual será o dashboard definitivo. Documente assim. A decisão virará ADR.

**Não existe backup.** Nenhum. É o maior gap conhecido da infra e deve
constar assim na documentação.

Não existem e não devem aparecer como se existissem: Vaultwarden,
interface dummy para split-DNS, servidor MCP, n8n, bot de Telegram,
Ansible, Grafana, Prometheus, k3s, Fail2Ban, CrowdSec.

---

## 3. Regras de segurança operacional

Absolutas.

- **NUNCA** rodar `docker compose down -v`. A flag `-v` destrói volumes.
- **NUNCA** rodar `rm -rf` fora do diretório de trabalho.
- **NUNCA** subir, derrubar ou reiniciar containers sem confirmação
  explícita minha. Nem para "testar se funciona".
- **NÃO** modificar arquivos em `*/data/`, `*/config/`, `*/etc-pihole/`,
  `*/appdata/` sem perguntar. São dados vivos, não código.
- **NÃO** tocar em bancos: `*.db`, `*.sqlite`, `*.db-wal`, `*.db-shm`,
  diretórios `mariadb/` e `redis/`.
- **NÃO** fazer commit. Eu reviso e commito.
- **NÃO** adicionar remote git nem fazer push. Nunca.

---

## 4. Regras de documentação

**A regra que mais importa:** documente o que ESTÁ no arquivo, não o que
a boa prática recomenda. Se algo não está configurado, escreva
"não configurado". Se está quebrado, escreva "quebrado". Nunca preencha
lacuna com suposição plausível — invenção plausível é pior que omissão,
porque não dá para distinguir do resto.

- Se não souber, escreva `<!-- VERIFICAR: pergunta -->` e me avise.
- **NÃO escreva ADRs nem postmortems.** O raciocínio por trás das decisões
  não está no filesystem — está na minha cabeça e em conversas. Você não
  tem acesso a ele e reconstruiria por suposição. Se identificar um
  candidato a ADR, me diga qual e por quê, sem escrever.
- README por stack: o que é, portas expostas, dependências entre
  containers, gotchas conhecidos. Curto.
- IPs privados (192.168.18.x) podem aparecer. Não são segredo e ajudam
  o texto a fazer sentido.
- Nomes de indexers específicos do Prowlarr **não** entram na
  documentação. Indexers são configuração externa não versionada.

---

## 5. Extração de segredos

Objetivo: tirar credenciais de dentro dos composes, deixando-os
publicáveis sem alteração posterior.

Padrão por stack:

```
pihole/
  docker-compose.yml    <- usa ${PIHOLE_PASSWORD}
  .env.example          <- PIHOLE_PASSWORD=      (placeholder VAZIO)
  .env                  <- valor real, ignorado pelo git
```

- `${VARIAVEL}` no compose, com `env_file: .env` no serviço.
- Um `.env` por stack, no diretório do stack.
- **O `.env.example` NUNCA recebe o valor real.** Placeholder vazio ou
  descritivo. Este é o erro mais caro possível nesta tarefa.
- Nome de variável em MAIÚSCULA, prefixado pelo stack.

Segredos conhecidos a extrair:

- `WEBPASSWORD` do Pi-hole
- Credenciais do banco do Nginx Proxy Manager
- API keys de Sonarr, Radarr, Prowlarr, Bazarr, Seerr
- Credenciais do OpenSubtitles (Bazarr)
- Senha do qBittorrent
- `TS_AUTHKEY` do Tailscale, se existir em algum compose

Lista não exaustiva. Varra tudo e reporte o que encontrar além disso,
inclusive em arquivos que não são compose: YAMLs de dashboard,
`settings.json`, `.toml`, `.conf`.

Nota: quando o Homepage for configurado, `config/services.yaml` e
`config/widgets.yaml` passarão a conter API keys dos *arr em texto puro.
Hoje estão com conteúdo padrão. Reavaliar depois da configuração.

---

## 6. Estilo

- Ao remover ou substituir um bloco, **comente em vez de deletar**.
  Deixe nota de onde estava e como reativar. Se poluir no lugar, mova o
  bloco comentado para o fim do arquivo com a referência.
- Português do Brasil.
- Markdown simples. Sem badges, sem emoji, sem "Getting Started" decorado.

---

## 7. Estrutura do repositório

Estrutura plana (um diretório por stack) **fica como está** — o Dockge
depende desses caminhos. Acrescente apenas `docs/`:

```
~/docker/
├── README.md
├── CLAUDE.md
├── .gitignore
├── docs/
│   ├── 00-hardware.md
│   ├── 01-host-setup.md        # Ubuntu, netplan, UFW, fstab, usuário, SSH
│   ├── 02-rede-e-dns.md        # Pi-hole, .home, systemd-resolved, Tailscale
│   ├── 03-proxy-reverso.md     # NPM
│   ├── 04-stacks.md            # convenção de diretórios, Dockge, PUID/PGID
│   ├── 05-backup.md            # atualmente: não existe
│   ├── adr/                    # EU escrevo
│   ├── postmortem/             # EU escrevo
│   └── runbooks/
├── pihole/
├── jellyfin/
└── ...
```

---

## 8. Antes de terminar qualquer tarefa

1. `docker compose config` em cada stack alterado — valida sintaxe sem subir.
2. Listar em texto tudo que foi alterado e por quê.
3. Não commitar.
