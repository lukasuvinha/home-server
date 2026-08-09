# home-server

Homelab pessoal de servidor único. Este repositório é a raiz dos stacks
Docker do host **melchior** e, ao mesmo tempo, a documentação da
infraestrutura.

Os serviços estão em uso diário. Não é laboratório descartável nem
demonstração montada para o repositório: o que está documentado aqui é o
que roda, incluindo o que está quebrado, o que está pela metade e o que
não existe.

Os compose files publicados são exatamente os que estão em produção, sem
etapa de sanitização. As credenciais foram extraídas para arquivos `.env`
não versionados (ver [ADR 004](docs/adr/004-estrategia-de-segredos.md)).

---

## Índice

- [Hardware](docs/00-hardware.md)
- [Setup do host](docs/01-host-setup.md) — Ubuntu, netplan, UFW, fstab, SSH
- [Rede e DNS](docs/02-rede-e-dns.md) — Pi-hole, wildcard `.home`, systemd-resolved, Tailscale
- [Proxy reverso](docs/03-proxy-reverso.md) — Nginx Proxy Manager e mapeamento de hosts
- [Convenção de stacks](docs/04-stacks.md) — estrutura, Dockge, PUID/PGID, redes
- [Backup](docs/05-backup.md) — atualmente inexistente
- [Decisões de arquitetura (ADR)](docs/adr/)
- [Postmortems](docs/postmortem/)
- [CLAUDE.md](CLAUDE.md) — contexto e regras operacionais para trabalho assistido por IA neste repositório

---

## O host

| | |
|---|---|
| Nome | `melchior` |
| Máquina | Acer Aspire E1-571 reaproveitado (notebook) |
| CPU | Intel Core i5-3230M, 2 núcleos / 4 threads |
| RAM | 8 GB DDR3 |
| Disco | 224 GB SSD (sistema, LVM) + 2 TB HD em `/mnt/hd2tb` |
| SO | Ubuntu Server 26.04 LTS |
| Rede | IP fixo via netplan, gateway na mesma `/24` |

A nomenclatura segue os supercomputadores MAGI de *Neon Genesis
Evangelion*. **Apenas o `melchior` existe.** `balthazar` (produção) e
`casper` (NAS) são planos sem hardware definido — não aparecem na
documentação como se existissem.

Detalhes em [docs/00-hardware.md](docs/00-hardware.md).

---

## Topologia

```
                      internet
                          |
                     roteador
                    (DHCP entrega
                   o Pi-hole como DNS
                    para a casa toda)
                          |
        LAN 192.168.x.0/24 ---- clientes da casa
                          |
                   melchior (host)
                          |
     +--------------------+--------------------+
     |                    |                    |
  Tailscale            Pi-hole           Nginx Proxy Manager
  (no host,          DNS + wildcard        HTTP apenas, :80/:81
   acesso remoto)      `.home`             roteia *.home para
                                           as portas locais
                                                  |
                        +-------------------------+-----------------+
                        |            |            |                 |
                   mediastack    dashboards    observabilidade    utilitários
                   (rede         Homepage      Beszel             Filebrowser
                   `mediastack`) Glance        Uptime Kuma        Navidrome
                                 Homarr                           WatchYourLAN
                                                                  Snowflake

                   Dockge + Portainer — gerência dos stacks
```

Não há exposição direta à internet: nenhuma porta é encaminhada no
roteador. O acesso remoto é feito exclusivamente por Tailscale
([ADR 003](docs/adr/003-tailscale-em-vez-de-port-forwarding.md)).

---

## Serviços

Um diretório por stack, estrutura plana, cada um com seu
`docker-compose.yml`. Os hostnames `.home` resolvem via wildcard do
Pi-hole e são roteados pelo Nginx Proxy Manager.

| Stack | Diretório | Porta(s) no host | Hostname | Estado |
|---|---|---|---|---|
| Pi-hole | `pihole/` | 53 (tcp/udp), 8080 | `pihole.home` | Ativo |
| Nginx Proxy Manager | `nginx-proxy-manager/` | 80, 81, 443 | `npm.home` | Ativo, **HTTP apenas** |
| Jellyfin | `jellyfin/` | 8096 | `jellyfin.home` | Ativo |
| qBittorrent | `jellyfin/` | 8090, 6881 | `qbittorrent.home` | Ativo |
| Prowlarr | `jellyfin/` | 9696 | `prowlarr.home` | Ativo |
| Sonarr | `jellyfin/` | 8989 | `sonarr.home` | Ativo |
| Radarr | `jellyfin/` | 7878 | `radarr.home` | Ativo |
| Bazarr | `jellyfin/` | 6767 | `bazarr.home` | Ativo |
| Seerr | `jellyfin/` | 5055 | `seerr.home` | Ativo |
| Dockge | `dockge/` | 5001 | `dockge.home` | Ativo — gerenciador principal |
| Portainer | `portainer/` | 9000, 9443 | `portainer.home` | Ativo — uso secundário (inspeção e limpeza de imagens) |
| Beszel | `beszel/` | 8095 | `beszel.home` | Ativo |
| Uptime Kuma | `uptime-kuma/` | 3001 | `uptimekuma.home` | Ativo, **básico** — só HTTP/ping, não checa estado de container |
| Filebrowser | `filebrowser/` | 8082 | `filebrowser.home` | Ativo |
| Snowflake | `snowflake/` | rede do host | — | Ativo |
| Homepage | `homepage/` | 3000 | `homepage.home` | **Em avaliação** |
| Glance | `glance/` | 8075 | `glance.home` | **Em avaliação** |
| Homarr | `homarr/` | 7575 | `homarr.home` | **Em avaliação** |
| Navidrome | `navidrome/` | 4533 | `navidrome.home` | **Instalado, não configurado** |
| WatchYourLAN | `watchyourlan/` | 8840 | `wylan.home` | **Quebrado** — funcionou por um período e parou; causa não investigada |
| Tailscale | host, não Docker | — | — | Ativo |
| Cockpit | host, não Docker | — | — | **Instalado, não configurado** |

Cada diretório tem um `README.md` próprio com portas, dependências,
variáveis de ambiente e gotchas.

Duas observações estruturais herdadas e ainda não resolvidas:

- O diretório `jellyfin/` contém a **mediastack inteira** (sete serviços
  numa rede `mediastack` compartilhada), não só o Jellyfin. Renomear é
  tarefa pendente.
- `filebrowser/` e `homarr/` têm um aninhamento acidental
  (`filebrowser/filebrowser/`, `homarr/homarr/`) causado por caminho
  relativo no compose. É ali que o estado real vive.

**Três dashboards rodam simultaneamente** (Homepage, Glance, Homarr).
Não é redundância por descuido: é comparação em andamento para escolher
qual será o definitivo. A decisão virará ADR.

---

## Como reproduzir

```
git clone https://github.com/lukasuvinha/home-server.git
cd home-server
```

Os stacks que precisam de credencial trazem um `.env.example` com as
variáveis e placeholders vazios. Para cada um:

```
cp pihole/.env.example pihole/.env
# preencher os valores
docker compose -f pihole/docker-compose.yml up -d
```

Hoje precisam de `.env`: `pihole`, `homarr`, `beszel` e `glance` (este
sem `.env.example` correspondente ainda). Os demais sobem sem
configuração adicional, mas os caminhos de bind mount presumem
`/mnt/hd2tb` e o usuário `uid 1000`.

---

## Decisões registradas

Cada ADR documenta uma escolha, o motivo, as consequências aceitas e as
alternativas descartadas.

| ADR | Assunto |
|---|---|
| [001](docs/adr/001-dockge-como-gerenciador-principal.md) | Dockge como gerenciador principal de stacks |
| [002](docs/adr/002-host-nao-usa-pihole-como-dns.md) | O host não usa o próprio Pi-hole como resolvedor |
| [003](docs/adr/003-tailscale-em-vez-de-port-forwarding.md) | Tailscale em vez de port forwarding |
| [004](docs/adr/004-estrategia-de-segredos.md) | Segredos em `.env`, repositório único, whitelist no `.gitignore` |
| [005](docs/adr/005-git-descartavel-antes-de-refatoracao-assistida.md) | Git local descartável como snapshot antes de refatoração assistida por IA |

---

## Incidentes

| Data | Incidente |
|---|---|
| [2026-07-31](docs/postmortem/2026-07-31-perda-de-dns-pos-apagao.md) | Perda total de resolução DNS no host após restabelecimento da energia |

O postmortem de 31/07 é o documento mais útil deste repositório para
entender como a infra está montada. O apagão não foi a causa raiz — ele
expôs uma cadeia de quatro camadas de configuração quebrada que existia
desde a instalação do Pi-hole, meses antes, e que funcionava por
coincidência.

---

## Limitações conhecidas

Lista deliberada. Nenhum destes itens está resolvido.

- **Não existe backup.** Nenhum mecanismo, nenhuma ferramenta instalada,
  nenhum job agendado. Falha de disco perde configuração de Pi-hole,
  bancos das dashboards, configuração do proxy reverso e a biblioteca
  inteira em `/mnt/hd2tb`. É o maior gap da infraestrutura. Ver
  [docs/05-backup.md](docs/05-backup.md).
- **Sem TLS.** O Nginx Proxy Manager roda em HTTP apenas, sem certificado
  e sem Access List, mesmo com a 443 mapeada. Credenciais administrativas
  (Dockge, Portainer, Filebrowser, Pi-hole) trafegam em claro na LAN.
- **Superfície administrativa ampla.** Dockge sobe com console habilitado
  e `docker.sock` montado — na prática, equivalente a root no host. O
  Filebrowser monta a raiz do filesystem em somente leitura, o que inclui
  os próprios `.env` que o ADR 004 tirou do git. Candidato a ADR.
- **Ponto único de falha.** Servidor único, notebook de 2013, sem
  nobreak. A bateria foi removida por incerteza sobre o estado — não há
  shutdown gracioso em queda de energia.
- **Monitoramento raso.** Uptime Kuma só faz HTTP e ping; não detecta
  container em restart loop nem serviço que responde 200 estando quebrado.
- **Sem convenção única de PUID/PGID.** Vários containers rodam como root
  e deixam arquivos com dono `root` nos bind mounts. Ver
  [docs/04-stacks.md](docs/04-stacks.md).
- **Pendências abertas do incidente de 31/07** estão listadas no próprio
  postmortem e ainda não foram todas fechadas.

---

## Próximos passos

Em ordem de prioridade, não de facilidade.

1. Backup com restic para disco externo dedicado. Restaurar do backup
   pelo menos uma vez antes de considerar o item resolvido.
2. Endurecer o acesso SSH: autenticação por chave, senha desabilitada.
3. TLS interno no Nginx Proxy Manager e Access List nos serviços
   administrativos.
4. Reduzir o escopo do bind mount do Filebrowser.
5. Escolher um dos três dashboards e remover os outros dois. Registrar em
   ADR.
6. Investigar e corrigir o WatchYourLAN, ou removê-lo.
7. Configurar o Cockpit ou desinstalá-lo.
8. Renomear `jellyfin/` para `mediastack/` e corrigir o aninhamento
   acidental em `filebrowser/` e `homarr/`.

---

## Convenções do repositório

- Estrutura plana, um diretório por stack. O Dockge depende desses
  caminhos.
- Segredos sempre como `${VARIAVEL}` no compose, com `env_file: .env` no
  serviço e um `.env.example` de placeholders vazios versionado ao lado.
- O `.gitignore` usa **whitelist**: ignora tudo e libera explicitamente
  markdown, compose files e arquivos `.example`. Falha fechada — arquivo
  novo desconhecido fica de fora por padrão.
- A documentação descreve o que está no arquivo, não o que a boa prática
  recomenda. Se algo está quebrado, está escrito "quebrado".
- Português do Brasil.
