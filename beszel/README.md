# Beszel

Monitoramento de recursos do host. Dois serviços no mesmo compose: `beszel`
(hub, interface web) e `beszel-agent` (coleta métricas e reporta ao hub).

## Portas

- `beszel`: `8095:8090` — interface web
- `beszel-agent`: nenhuma porta publicada (`network_mode: host`)

## Dependências

- `beszel-agent` fala com o hub via `HUB_URL=http://localhost:8095`. Como o
  agent roda em `network_mode: host`, esse `localhost` é o host físico, não
  o container do hub — funciona porque a porta do hub está publicada no
  host.
- Os dois serviços compartilham um socket via volume `./beszel_socket`,
  montado em ambos.
- Volumes do host: `./beszel_data` (dados do hub, inclui a chave
  `id_ed25519`), `./beszel_agent_data` (dados do agent),
  `/var/run/docker.sock:ro` (agent monitora containers),
  `/mnt/hd2tb/.beszel:ro` (filesystem extra monitorado pelo agent).

## Variáveis de ambiente

- `beszel`: `APP_URL` (fixo no compose, não é segredo).
- `beszel-agent`: usa `env_file: .env`. Variável presente:
  `BESZEL_AGENT_TOKEN` (token de enrollment). `LISTEN`, `HUB_URL` e `KEY`
  ficam fixos no compose — `KEY` é a chave pública SSH do hub, não é
  segredo.

## Gotchas

- `beszel-agent` usa `network_mode: host` — não está na rede padrão do
  Compose; a comunicação com o hub depende da porta estar publicada no
  host, não de resolução de nome entre containers.
- Healthcheck de ambos os serviços está comentado no compose (desativado).
- A chave privada SSH do hub (`beszel_data/id_ed25519`) tem permissão
  `600` e dono `root:root` no host — ilegível para o usuário `lukas`.
