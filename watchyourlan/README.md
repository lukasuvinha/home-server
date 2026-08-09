# WatchYourLAN

Scanner de dispositivos na rede local. Instalado, não configurado —
funcionou por um período e parou; causa não investigada (ver `CLAUDE.md`).

## Portas

Nenhuma publicada explicitamente — `network_mode: host`.

## Dependências

- Containers: nenhum.
- Rede: nenhuma; usa a rede do host diretamente (necessário para escanear
  a LAN pela interface física).
- Volumes do host: `~/docker/watchyourlan:/data/WatchYourLAN` — o próprio
  diretório do stack é montado dentro do container.

## Variáveis de ambiente

- `TZ`
- `IFACES` — nome da interface de rede a escanear (hoje `enp2s0f0`).

## Gotchas

- `IFACES` é um nome fixo de interface. Se esse nome mudar no host
  (troca de placa, renomeação via udev/netplan), o scan para de funcionar
  sem erro óbvio no compose.
- `config_v2.yaml`, no mesmo diretório, está vazio (0 bytes).
- <!-- VERIFICAR: por que o serviço parou de funcionar depois de rodar por
  um tempo? Não é visível a partir do compose — precisa checar logs do
  container ou o estado de `IFACES` no momento em que parou. -->
