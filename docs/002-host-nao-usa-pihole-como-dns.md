# ADR 002 — O host não usa o Pi-hole como resolvedor

**Status:** aceito
**Data:** <!-- VERIFICAR -->

## Contexto

O Pi-hole roda em container no melchior e serve DNS para toda a rede
doméstica via DHCP do roteador, incluindo o domínio wildcard `.home`
apontando para 192.168.18.195.

Se o próprio host apontasse para o Pi-hole, criaria dependência circular:
o Docker precisa de DNS para subir containers, e o container que fornece
DNS ainda não subiu.

## Decisão

O melchior resolve DNS por servidores externos, não pelo Pi-hole que ele
hospeda. `systemd-resolved` com `DNSStubListener` ativo, `resolv.conf`
como symlink para o stub.

## Justificativa

Quebra da dependência circular. Um host que depende do próprio container
para resolver nomes não consegue se recuperar sozinho de um boot frio.

<!-- VERIFICAR: você chegou nessa configuração antes ou depois do
     incidente de DNS pós-apagão? A ordem muda a narrativa. -->

## Consequências

- O tráfego do próprio host não passa pelo bloqueio de anúncios.
- O host se recupera de reinício sem intervenção manual.
- Coexistência entre `systemd-resolved`, o Pi-hole na porta 53 e o
  gerenciador de DNS do Tailscale exige configuração explícita e é
  frágil a mudanças — ver o postmortem da falha de DNS pós-apagão.

## Alternativas descartadas

- **Host apontando para o Pi-hole:** dependência circular.
- **Desativar `systemd-resolved`:** quebra a integração do Tailscale
  com DNS, que depende dele.
