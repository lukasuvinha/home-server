# Hardware

Fonte: `hostnamectl`, `lscpu`, `free -h`, `lsblk`, `df -h`, `/etc/fstab`,
rodados no host em 2026-08-09.

## Máquina

- Vendor/modelo: Acer Aspire E1-571 (hardware SKU `Aspire E1-571_064B_V2.15`)
- Chassis: laptop
- Firmware: versão V2.15, datado de 2013-03-11
- Hostname: `melchior`

## CPU

- Intel Core i5-3230M @ 2.60GHz
- 2 núcleos físicos, 4 threads (hyper-threading)
- Cache: L1d 64KiB, L1i 64KiB, L2 512KiB, L3 3MiB
- Virtualização: VT-x presente

## Memória

- RAM total: 7,1Gi (`free -h`; comercialmente 8GB)
- Swap: 4,0Gi — arquivo `/swap.img` (não é partição), montado via `/etc/fstab`
- No momento da checagem: swap com 2,0Gi em uso

## Armazenamento

| Disco | Tamanho | Partições / uso |
|---|---|---|
| `sda` | 223,6G | `sda1` (1G) → `/boot/efi`; `sda2` (2G) → `/boot`; `sda3` (220,5G) → volume group LVM `ubuntu-vg`, contendo o LV `ubuntu-lv` de 100G montado em `/` |
| `sdb` | 1,8T | `sdb1` (1,8T) → `/mnt/hd2tb`, montado via UUID no fstab com opção `nofail` |

A partição `sda3` tem 220,5G mas o LV `ubuntu-lv` montado em `/` usa só
100G — sobram cerca de 120G no volume group `ubuntu-vg` sem LV alocado.
Não configurado de propósito, pelo que se sabe hoje — a única
configuração feita manualmente na instalação do Ubuntu foi o IP fixo
local e os servidores DNS (antes do Pi-hole entrar em ação). O espaço
sobrando no VG provavelmente não tem uso planejado, mas isso não está
confirmado.

## Sistema operacional

- Ubuntu Server 26.04 LTS
- Kernel `7.0.0-29-generic`
- Timezone: America/Sao_Paulo, relógio sincronizado via NTP
