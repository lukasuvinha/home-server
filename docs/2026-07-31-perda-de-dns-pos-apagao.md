# Postmortem: perda total de resolução DNS no host após restabelecimento da energia

**Data do incidente:** 2026-07-31
**Duração:** ~3 horas de diagnóstico
**Impacto:** host melchior sem resolução de nomes. Serviços em container e
demais dispositivos da rede não afetados.
**Status:** resolvido e validado após reboot

---

## Resumo

Ribeirão Preto sofreu uma microexplosão atmosférica que deixou a região
dois dias sem energia elétrica e seis dias sem internet. Ao restabelecer
a conectividade, o melchior havia perdido completamente a capacidade de
resolver nomes de domínio, enquanto a conectividade IP permanecia normal.

**O apagão não foi a causa raiz.** Ele apenas expôs uma configuração
quebrada que existia desde a instalação do Pi-hole, meses antes, e que
funcionava por coincidência.

A causa real foi uma cadeia de quatro camadas de configuração, nenhuma
delas suficiente sozinha para derrubar o serviço.

---

## Sintomas

```
$ ping google.com
ping: google.com: Temporary failure in name resolution

$ ping 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=1 ttl=116 time=12.9 ms      # funciona

$ sudo apt update
Falha temporária ao resolver 'archive.ubuntu.com'           # todos os mirrors

$ nslookup google.com
;; Got SERVFAIL reply from 100.100.100.100, trying next server
** server can't find google.com: SERVFAIL
```

Quadro clínico:

- Conectividade IP íntegra — ping por IP, SSH e o túnel Tailscale funcionando
- Resolução de nome quebrada em 100% das consultas, no host
- Demais dispositivos da casa navegando normalmente
- Container do Pi-hole rodando e reportando `healthy`

A combinação "tudo funciona, menos nome" apontava para o resolvedor do
host, não para rede nem para o Pi-hole em si. Mas o fato de o Pi-hole
estar saudável e a casa inteira navegando tornou o diagnóstico
contraintuitivo desde o início.

---

## A cadeia causal

Quatro problemas empilhados. Isolados, nenhum derrubaria nada.

### Camada 1 — Pi-hole ocupando a porta 53 em wildcard

```yaml
ports:
  - "53:53/tcp"      # sem IP especificado = bind em 0.0.0.0 E [::]
  - "53:53/udp"
```

Sem IP na declaração, o `docker-proxy` ocupa a porta 53 em **todos** os
endereços IPv4 do host — incluindo `127.0.0.53`, exatamente onde o
`systemd-resolved` quer escutar.

### Camada 2 — `DNSStubListener=no`

Como os dois não coexistem, a receita padrão de Pi-hole em Docker
(inclusive a da documentação oficial do próprio Pi-hole) manda desligar
o stub listener:

```
/etc/systemd/resolved.conf:  DNSStubListener=no
```

Isso foi feito na instalação, meses antes, e funcionou.

O efeito colateral não é óbvio: o `resolv.conf` passa a operar em **modo
`uplink`**. As aplicações falam direto com os upstreams, sem passar pelo
`127.0.0.53`. O `systemd-resolved` continua rodando, mas fora do caminho
de resolução.

### Camada 3 — Tailscale caindo no "direct manager"

O `tailscaled` precisa injetar DNS no sistema para o MagicDNS funcionar.
No Linux ele detecta o ambiente e escolhe um gerenciador, nesta ordem:

```
systemd-resolved → resolvconf → openresolv → NetworkManager → direct
```

Sem o stub listener ativo, ele conclui que o `systemd-resolved` não está
no comando da resolução e cai no último da lista: o **direct manager**,
que sobrescreve `/etc/resolv.conf` na marra:

```
nameserver 100.100.100.100
```

Dois sintomas visíveis disso, ambos ignorados por meses: `resolv.conf
mode: foreign` no `resolvectl status`, e `/etc/resolv.conf` deixando de
ser symlink para virar arquivo comum.

Isso era **determinístico**, não intermitente. Acontecia a cada início do
`tailscaled`. Só era invisível porque o `100.100.100.100` respondia.

### Camada 4 — netplan sem fallback

```yaml
nameservers:
  addresses: []              # vazio
  search: [1.1.1.1, 1.0.0.1] # IPs no campo errado
```

O campo `search` é para sufixos de domínio, não para servidores. Na
prática, não havia nameserver configurado — e portanto nenhum fallback
quando o `100.100.100.100` parou de responder.

Agravante descoberto no processo: o gateway `192.168.18.1` **não roda
forwarder de DNS**, contrariando a suposição comum de que o roteador
sempre serve como resolvedor de emergência.

### Como as quatro se somaram

O apagão quebrou o fetch de configuração de DNS do Tailscale. O
`100.100.100.100` passou a retornar SERVFAIL para tudo. Como ele era o
único nameserver no `resolv.conf` sequestrado, e o netplan não tinha
fallback, e o gateway não resolve — o host ficou sem nenhum caminho.

---

## Hipóteses descartadas

Registradas porque o caminho até a causa foi mais longo que a causa:

| Hipótese | Como foi descartada |
|---|---|
| Mirrors do Ubuntu fora do ar | `nala fetch` também falhou, e falhava antes de qualquer mirror |
| Container do Pi-hole caído ou com IP trocado | `docker ps` mostrava healthy, IP correto |
| Rede de containers quebrada | Outros containers comunicando normalmente |
| Unbound na porta 5335 | Nenhum Unbound instalado |
| Firewall pfSense bloqueando 53 | Não existe pfSense nesta rede |
| Provedor bloqueando porta 53 de saída | `dig` novo resolveu `debian.org` com TTL real |
| Clock dessincronizado quebrando TLS | Sintoma real, mas consequência — não causa |
| Loop de atualização do gravity travado | Sintoma, não causa |

---

## Erro de diagnóstico

Uma resposta de `dig` trouxe a flag EDE **"Stale Answer"**. Isso foi lido
como evidência de que o upstream estava acessível.

Era o contrário: o Pi-hole serviu uma resposta do cache **porque** o
upstream falhou. A flag existe justamente para sinalizar que o dado é
velho e que a consulta ao upstream não teve sucesso.

Essa leitura invertida custou tempo e desviou a investigação para a
camada errada — foi procurar problema de conectividade externa quando o
problema estava no caminho de resolução local.

<!-- VERIFICAR: vale dizer o que você faria diferente. Ler a RFC da EDE
     antes de interpretar? Testar upstream direto com dig @1.1.1.1 em vez
     de inferir do cache? Escreva o que você concluiu de fato. -->

---

## Correção

A ordem importa. Reativar o stub antes de liberar a porta faz o
`systemd-resolved` tentar bindar `127.0.0.53:53` com o `docker-proxy`
ainda em wildcard, resultando em `EADDRINUSE` e serviço subindo quebrado.

**1. Restringir o bind do Pi-hole ao IP do host**

```yaml
ports:
  - "192.168.18.195:53:53/tcp"
  - "192.168.18.195:53:53/udp"
  - "8080:80/tcp"
```

Especificar o IP também impede o Docker de criar o binding `[::]`. Isso
importa: com `net.ipv6.bindv6only=0` (padrão no Linux), um socket em
`[::]:53` pode capturar tráfego IPv4 e continuar atropelando o
`127.0.0.53`.

```bash
docker compose up -d --force-recreate
sudo ss -ulpn | grep :53        # deve mostrar só 192.168.18.195:53
```

**2. Reativar o stub listener**

```bash
sudo sed -i 's/^DNSStubListener=no/#DNSStubListener=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

Comentado, não removido — o histórico da linha é a documentação de por
que ela existia.

**3. Restaurar o `resolv.conf` como symlink**

```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

**4. Corrigir o netplan**

```yaml
nameservers:
  addresses: [1.1.1.1, 1.0.0.1]
  search: []
dhcp6: false
```

**5. Reiniciar o `tailscaled` — não basta `tailscale set`**

```bash
sudo systemctl restart tailscaled
sudo tailscale set --accept-dns=true
```

Este foi o passo crítico e o menos óbvio. O daemon escolhe o gerenciador
de DNS **na inicialização**, não a cada chamada de `tailscale set`.
Enquanto ele não reiniciasse, continuaria em direct manager mesmo com o
stub já ativo.

---

## Validação

Confirmado após reboot limpo — o reboot é parte da correção, não
formalidade, porque prova que a configuração persiste em vez de depender
do estado da sessão:

- `tailscale0` com `Current Scopes: DNS`, e `100.100.100.100` escopado
  apenas àquele link
- Global limpo, em `resolv.conf mode: stub`
- MagicDNS resolvendo peers por nome curto
- Relógio do sistema sincronizando automaticamente (o `timesyncd`
  precisava resolver `ntp.ubuntu.com`)

---

## Lições

<!-- Esta seção é a mais importante do documento e a que um entrevistador
     vai puxar. O que está abaixo é ponto de partida — reescreva com o que
     você concluiu de fato. -->

**Configuração que funciona não é o mesmo que configuração correta.**
O `resolv.conf mode: foreign` estava visível desde o primeiro dia. Nada
quebrou por meses porque o caminho errado funcionava. Um evento externo
não criou o problema; removeu a coincidência que o escondia.

**Receita oficial não é receita segura no seu contexto.** O
`DNSStubListener=no` vem da documentação do Pi-hole. Está certo no escopo
dela e errado no meu, porque ela não sabe que existe Tailscale na
máquina. Documentação de um componente não conhece a interação com os
outros.

**Ausência de fallback só aparece quando o caminho principal cai.** O
netplan com `nameservers.addresses` vazio era invisível enquanto qualquer
outro caminho respondia.

**Daemon que decide na inicialização precisa de restart, não de reload.**
`tailscale set` parecia estar aplicando a configuração. Não estava — a
decisão do gerenciador já tinha sido tomada.

---

## Checklist derivado para servidores futuros

Aplicável a balthazar e casper quando existirem:

- [ ] Todo bind de porta em compose especifica o IP. Nunca wildcard em
      serviço de infraestrutura.
- [ ] `resolvectl status` conferido após instalar qualquer coisa que mexa
      em DNS. `mode` deve ser `stub`; `foreign` é sinal de alerta.
- [ ] `/etc/resolv.conf` conferido como symlink, não arquivo comum.
- [ ] `nameservers.addresses` preenchido no netplan como fallback real.
      Não assumir que o gateway resolve.
- [ ] Host nunca depende de container hospedado nele para resolver nomes.
- [ ] Reboot de validação após qualquer alteração em DNS. Configuração que
      não sobrevive ao boot não está aplicada, está em memória.

---

## Pendências que o incidente revelou

<!-- VERIFICAR: marque o que já foi resolvido desde 31/07 -->

- [ ] `fe80::1` ainda na lista de DNS do `enp2s0f0`, vindo de Router
      Advertisement — `dhcp6: false` não bloqueia RA. Se houver lentidão,
      `accept-ra: false` no netplan.
- [ ] Clientes IPv6 do Pi-hole: ao restringir o bind ao IPv4, o Pi-hole
      deixou de responder DNS por IPv6. Dispositivos que o alcançavam por
      RA passaram a usar outro DNS e perderam filtragem silenciosamente.
      Conferir contagem de clientes no dashboard.
- [ ] Senha do Pi-hole exposta em texto puro durante a investigação —
      rotacionar e migrar para `.env`.
- [ ] Interface dummy para recuperar `.home` no host. Adiada
      conscientemente; agora viável porque o stub está ativo.
- [ ] Sem nobreak. A bateria do notebook foi removida por incerteza sobre
      seu estado. Inspecionar por inchaço; se íntegra, recolocar para
      shutdown gracioso.
