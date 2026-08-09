# ADR 003 — Tailscale para acesso remoto, sem exposição por port forwarding

**Status:** aceito
**Data:** <!-- VERIFICAR -->

## Contexto

Acesso aos serviços do melchior fora da rede local. As opções
convencionais são redirecionamento de portas no roteador, VPN
tradicional, ou rede mesh overlay.

O melchior roda serviços com autenticação fraca ou inexistente
(Nginx Proxy Manager em HTTP puro, Dockge, Portainer, Filebrowser),
além de SSH com autenticação por senha na porta 22.

## Decisão

Tailscale instalado no host, sem containerização. Nenhum redirecionamento
de portas configurado no roteador.

## Justificativa

Com port forwarding, cada serviço exposto é uma superfície de ataque
independente e permanente. Com o Tailscale, o acesso remoto depende de
identidade autenticada, e a superfície pública é zero.

Isso torna aceitáveis, no curto prazo, configurações que seriam
inaceitáveis com exposição pública — em particular o NPM sem TLS.
É uma dívida consciente, não um descuido.

## Consequências

- Serviços acessíveis remotamente sem nenhuma porta aberta.
- Dependência de um serviço de terceiros para o plano de controle.
- O TLS no NPM continua pendente. Sem exposição pública, a urgência é
  baixa; a dívida permanece registrada.
- Necessário verificar se o UPnP no roteador está abrindo portas por
  conta própria — um cliente BitTorrent solicita isso por padrão, o que
  contornaria a premissa desta decisão.
  <!-- VERIFICAR: confira UPnP, DMZ e Servidor Virtual no roteador e
       registre o resultado aqui. Se houver porta aberta, esta decisão
       está parcialmente furada e o SSH vira urgente. -->

## Alternativas descartadas

- **Port forwarding direto:** expõe serviços sem autenticação adequada.
- **WireGuard próprio:** funcionalmente equivalente, mas exige porta
  aberta e gestão manual de chaves e NAT traversal.
