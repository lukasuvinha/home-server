# Runbook: proteger o acesso ao WatchYourLAN

Data da elaboração: 2026-08-18.

Este documento descreve dois planos de ação alternativos para colocar
autenticação na frente do WatchYourLAN (`wylan.home`, hoje acessível sem
login em `192.168.18.195:8840`). Os dois planos **não são cumulativos** —
escolha um. Este arquivo existe pra revisão antes de qualquer execução.
Nenhum passo aqui foi executado ainda.

Convenção de marcação de cada passo:

- `[CLAUDE]` — edição de arquivo ou checagem que posso executar diretamente.
- `[CLAUDE - PEDIR CONFIRMAÇÃO]` — envolve subir/derrubar/recriar container.
  Não faço sem você autorizar explicitamente aquele passo específico, mesmo
  que o plano geral já tenha sido escolhido.
- `[MANUAL - SUDO]` — comando que mexe no host (UFW). Você roda por fora,
  eu não tenho sudo interativo nesta sessão de qualquer forma.
- `[MANUAL]` — ação na interface web da NPM ou decisão que só você deve
  tomar (senha, nome de usuário, hostname).
- `[TESTE - CLAUDE]` — eu confiro depois do passo.
- `[TESTE - MANUAL]` — só você consegue confirmar (navegador, Tailscale
  ligado/desligado).

---

## 1. Por que nenhum dos dois planos usa só o firewall

Antes dos dois planos, o porquê da arquitetura de base ser igual nos dois:

- O WatchYourLAN faz varredura ARP de camada 2. Isso exige enxergar o
  tráfego real da LAN, o que só funciona com `network_mode: host` (ou
  macvlan). Não dá pra isolar ele numa rede bridge interna do Docker sem
  quebrar a função principal do container.
- O WatchYourLAN não tem autenticação própria (confirmado na documentação
  oficial do projeto: "WatchYourLAN does not have built-in auth option").
- Qualquer autenticação colocada num proxy reverso (NPM ou Authelia) só
  vale para quem passa pelo proxy. Enquanto a porta `8840` do
  WatchYourLAN estiver alcançável direto pela rede, dá pra pular o proxy
  simplesmente digitando `IP:8840` no navegador — isso já foi confirmado
  na prática nesta conversa.
- Tentamos fechar isso só com UFW (regra `8840/tcp ALLOW IN Anywhere`,
  criada em 2026-08-13 pra resolver o monitoramento do Uptime Kuma).
  Só que descobrimos, também nesta conversa, que **o tráfego do Tailscale
  neste host não passa pelas regras normais do UFW** — foi assim que o
  acesso via Tailscale funcionava mesmo antes de existir qualquer regra
  para a porta 8840. Isso significa que restringir a origem da regra do
  UFW (por exemplo, liberar só a rede da NPM) não tem garantia de barrar
  quem está no Tailscale. Não testei esse cenário específico porque o
  teste arriscaria derrubar seu acesso atual sem necessidade.

Conclusão: a única forma de fechar o acesso direto de forma que não
dependa do comportamento do UFW ou do Tailscale é fazer o WatchYourLAN
escutar **só no loopback do host** (`HOST=127.0.0.1`, variável já
documentada pelo projeto). Isso é reforçado pelo kernel, não por regra de
firewall — nada fora do próprio host alcança um socket em `127.0.0.1`,
não importa a interface de origem.

Consequência direta: a NPM (hoje em rede bridge, só com portas publicadas
`80`, `81`, `443`) precisa passar a rodar em `network_mode: host` também,
porque é a única forma dela alcançar `127.0.0.1:8840` do próprio host.
Essa mudança de rede é **comum aos dois planos**, não é específica de
usar NPM Access List ou Authelia.

### Auditoria feita antes de escrever este plano

Antes de propor a troca de rede da NPM, chequei (somente leitura, sem
alterar nada) o banco `nginx-proxy-manager/data/database.sqlite` pra ver
se algum Proxy Host depende de resolução de nome de container (o que
quebraria ao sair da rede bridge). Resultado: **todos os 20 proxy hosts
configurados hoje apontam para o IP real do host, `192.168.18.195`**, nunca
para nome de container ou IP de rede bridge:

```
pihole.home        -> 192.168.18.195:8080
portainer.home      -> 192.168.18.195:9000
jellyfin.home       -> 192.168.18.195:8096
sonarr.home         -> 192.168.18.195:8989
radarr.home         -> 192.168.18.195:7878
bazarr.home         -> 192.168.18.195:6767
seerr.home          -> 192.168.18.195:5055
qbittorrent.home    -> 192.168.18.195:8090
prowlarr.home       -> 192.168.18.195:9696
npm.home            -> 192.168.18.195:81
beszel.home         -> 192.168.18.195:8095
homarr.home         -> 192.168.18.195:7575
glance.home         -> 192.168.18.195:8075
homepage.home       -> 192.168.18.195:3000
wylan.home          -> 192.168.18.195:8840
dockge.home         -> 192.168.18.195:5001
uptimekuma.home     -> 192.168.18.195:3001
navidrome.home      -> 192.168.18.195:4533
filebrowser.home    -> 192.168.18.195:8082
router.home         -> 192.168.18.1:1 (https)
```

Isso é uma boa notícia: **trocar a NPM para `network_mode: host` não deve
quebrar nenhum proxy host existente**, porque nenhum depende da rede
interna do Docker — todos já falam com o IP real do host. Também
confirmei que **não existe nenhuma Access List configurada hoje** (tabela
`access_list` vazia), batendo com o que já estava documentado no
CLAUDE.md.

### Fora do escopo dos dois planos (limitações conhecidas, não resolvidas aqui)

- **TLS.** A NPM continua HTTP puro nos dois planos. Isso importa
  principalmente para o Plano A: Basic Auth sobre HTTP trafega usuário e
  senha em texto claro dentro da LAN. Pelo Tailscale isso é menos grave
  (o túnel já é criptografado por fora), mas na LAN local, sem TLS,
  qualquer um capturando tráfego vê a senha. Resolver isso é um projeto
  separado (certificado, domínio próprio ou self-signed).
- **Resolução de `.home` via Tailscale.** Você mencionou que hoje acessa
  os endereços da NPM pelo Tailscale digitando o IP da VPN puro, não o
  hostname `algo.home`. Isso é porque o DNS wildcard `.home` só existe no
  Pi-hole, que só é consultado por dispositivos usando ele como DNS — o
  que normalmente não é o caso quando você está fora da LAN, mesmo
  conectado ao Tailscale (a não ser que o MagicDNS ou os nameservers do
  Tailscale estejam apontando pro Pi-hole, o que não foi verificado aqui).
  **Isso afeta os dois planos igualmente**: se você continuar digitando o
  IP puro em vez do hostname, o Nginx da NPM não sabe pra qual Proxy Host
  rotear (roteamento é por `Host` header/domínio) e a autenticação
  configurada pode nem entrar em ação corretamente. Testar isso está nos
  passos de teste manual de cada plano abaixo — se falhar, é sinal de que
  esse ponto precisa ser resolvido à parte (dá pra fazer outro plano só
  pra isso, se quiser, depois).

---

## 2. Passo 0 — preparação comum aos dois planos

Faça isso antes de escolher entre A e B — é reversível e não muda
comportamento nenhum ainda.

1. `[CLAUDE]` Backup dos arquivos que serão editados:
   - `watchyourlan/docker-compose.yml` → `docker-compose.yml.bak-20260818`
   - `nginx-proxy-manager/docker-compose.yml` → `docker-compose.yml.bak-20260818`
2. `[CLAUDE]` Confirmar mais uma vez, na hora de executar, que a versão do
   WatchYourLAN em uso (`2.1.4`, verificado em 2026-08-13) ainda documenta
   a variável `HOST` — projetos mudam, e este plano foi escrito numa data
   específica.

---

## 3. Plano A — NPM Access List (Basic Auth)

### Arquitetura resultante

```
Cliente (LAN ou Tailscale)
   │  http://wylan.home
   ▼
NPM (network_mode: host, escuta 80/443 direto no host)
   │  Access List: pede usuário/senha antes de encaminhar
   ▼
WatchYourLAN (HOST=127.0.0.1, só escuta em 127.0.0.1:8840)
```

Ninguém de fora do host alcança `8840` diretamente. O único caminho é
passar pela NPM, que agora roda no mesmo namespace de rede do host e por
isso consegue falar com `127.0.0.1:8840`.

### Passo a passo

1. `[CLAUDE]` Editar `watchyourlan/docker-compose.yml`: adicionar
   `HOST: 127.0.0.1` em `environment`.
2. `[CLAUDE]` Rodar `docker compose config` no diretório `watchyourlan/`
   pra validar sintaxe (não sobe nada).
3. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Recriar o container do WatchYourLAN
   (`docker compose up -d`) pra aplicar a nova variável.
4. `[TESTE - CLAUDE]` Depois de recriar:
   - `curl 127.0.0.1:8840` do host → esperado `200`.
   - `curl --max-time 3 192.168.18.195:8840` do host → esperado falhar
     (recusa ou timeout, já não deve responder pela interface da LAN).
   - Repetir o teste de conectividade que já fiz do container do Kuma até
     `192.168.18.195:8840` → esperado **falhar agora** (isso é o sinal de
     que o endurecimento funcionou; se ainda responder, algo saiu errado).
   - Conferir logs do WatchYourLAN sem erro (`docker logs
     watchyourlan-wyl-1 --tail 30`) e rodar de novo o `PRAGMA
     integrity_check` no `scan.db` (mesma checagem já feita antes,
     somente leitura).
5. `[MANUAL - SUDO]` Remover a regra de firewall que não faz mais falta
   (a porta não precisa mais estar aberta pra rede nenhuma):
   ```
   sudo ufw delete allow 8840/tcp
   ```
6. `[CLAUDE]` Editar `nginx-proxy-manager/docker-compose.yml`: remover o
   bloco `ports:` (80/81/443) e adicionar `network_mode: host`.
7. `[CLAUDE]` Rodar `docker compose config` no diretório
   `nginx-proxy-manager/` pra validar.
8. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Recriar o container da NPM. Aviso: o
   painel administrativo (porta 81) fica fora do ar por alguns segundos
   durante a troca — e, brevemente, **todos os outros serviços proxied
   pela NPM também ficam inacessíveis via `*.home`** até o container
   voltar (não é específico do WatchYourLAN, é a NPM inteira reiniciando).
9. `[TESTE - CLAUDE]` Depois de recriar:
   - Painel da NPM responde em `192.168.18.195:81`.
   - `docker inspect nginx-proxy-manager-app-1` confirma
     `NetworkMode: host`.
   - Testar cada um dos 20 proxy hosts listados na auditoria acima com
     `curl -H "Host: <dominio>" http://192.168.18.195/` e reportar
     qualquer um que não responda mais como antes.
10. `[MANUAL]` No painel da NPM:
    a. Criar uma **Access List** nova, com usuário e senha à sua escolha
       (não vou gerar nem sugerir senha — isso é credencial, fica com
       você).
    b. Editar o Proxy Host `wylan.home` já existente: trocar
       "Forward Hostname/IP" de `192.168.18.195` para `127.0.0.1`
       (a porta continua `8840`).
    c. Anexar a Access List criada em (a) a esse Proxy Host.
11. `[TESTE - MANUAL]` Pela LAN, acessar `http://wylan.home` — deve
    aparecer um prompt de usuário/senha antes de mostrar a interface.
12. `[TESTE - MANUAL]` Pela LAN, tentar `http://192.168.18.195:8840`
    direto — deve dar erro de conexão recusada, sem pedir nada (a porta
    não existe mais pra fora do host).
13. `[TESTE - MANUAL]` Repetir os dois testes acima **conectado ao
    Tailscale**, sem estar na LAN. Preste atenção em qual endereço você
    digita — pelo ponto 1 (limitação de DNS), talvez `wylan.home` não
    resolva fora da LAN; se não resolver, esse é o sinal de que a
    limitação de DNS via Tailscale citada na seção 1 precisa ser
    resolvida à parte antes desse plano funcionar remotamente.
14. `[MANUAL ou CLAUDE, a seu critério]` Reconfigurar o monitor do
    Uptime Kuma: trocar a URL monitorada de `http://192.168.18.195:8840`
    para `http://wylan.home` (ou, se o DNS não resolver de dentro do
    container do Kuma, usar `http://192.168.18.195/` com o campo "Host
    Header" da NPM preenchido como `wylan.home` — o Kuma permite
    customizar isso em monitores HTTP(s)). Preencher usuário e senha da
    Access List no monitor (o Kuma tem campos nativos de Basic Auth).
15. `[TESTE - CLAUDE]` Confirmar que o monitor do Kuma volta a reportar
    "up".

### Prós

- Reaproveita a NPM, que já está em produção — nenhum serviço novo pra
  manter ou atualizar.
- Configuração inteira pela interface web da NPM, sem editar YAML de
  autenticação.
- Menos passos, implementação mais rápida.

### Contras

- Sem 2FA — é só usuário e senha. Se a senha vazar, não há segunda
  camada.
- Access List é "tudo ou nada": uma senha só, compartilhada por qualquer
  um que precise entrar. Sem contas individuais, sem log de quem acessou.
- Sem TLS (fora do escopo, seção 1), a senha trafega em texto claro
  dentro da LAN.
- A NPM passa a rodar em `network_mode: host`, perdendo o isolamento de
  rede que tinha (fica exposta com a mesma superfície de um processo
  nativo do host). Mitigado pelo fato de já termos confirmado que nenhum
  proxy host depende da rede interna do Docker.

### Rollback

1. Restaurar os dois `.bak` gerados no passo 0.
2. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Recriar os dois containers com a
   configuração original.
3. `[MANUAL - SUDO]` `sudo ufw allow 8840/tcp` (repor a regra removida).
4. Remover a Access List do Proxy Host `wylan.home` pela interface da
   NPM (ou deixar — sem porta acessível de fora, ela fica sem efeito,
   mas o ideal é desfazer pra não confundir depois).

---

## 4. Plano B — Authelia (portal de autenticação com sessão/2FA)

### Arquitetura resultante

```
Cliente (LAN ou Tailscale)
   │  http://wylan.home
   ▼
NPM (network_mode: host)
   │  bloco "Advanced" com auth_request apontando pro Authelia
   ▼
Authelia (127.0.0.1:9091, verifica sessão/cookie)
   │  se não autenticado: redireciona pra http://auth.home (login)
   │  se autenticado: libera
   ▼
WatchYourLAN (HOST=127.0.0.1, só escuta em 127.0.0.1:8840)
```

Authelia é um novo stack (`authelia/`), separado do WatchYourLAN e da
NPM. Assim como o WatchYourLAN, ele também fica restrito a `127.0.0.1`
(mesma lógica de trava por loopback) — só a NPM em modo host o alcança.

### Passo a passo

Os passos 1 a 9 são **idênticos aos do Plano A** (bloquear o
WatchYourLAN em loopback, remover a regra de UFW, colocar a NPM em modo
host) — não repito o detalhamento aqui, só a lista:

1. `[CLAUDE]` `HOST: 127.0.0.1` no `watchyourlan/docker-compose.yml`.
2. `[CLAUDE]` `docker compose config` (watchyourlan).
3. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Recriar container do WatchYourLAN.
4. `[TESTE - CLAUDE]` Mesma bateria do Plano A passo 4.
5. `[MANUAL - SUDO]` `sudo ufw delete allow 8840/tcp`.
6. `[CLAUDE]` `network_mode: host` no `nginx-proxy-manager/docker-compose.yml`.
7. `[CLAUDE]` `docker compose config` (nginx-proxy-manager).
8. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Recriar container da NPM.
9. `[TESTE - CLAUDE]` Mesma bateria do Plano A passo 9 (20 proxy hosts).

A partir daqui, específico do Authelia:

10. `[CLAUDE]` Criar o diretório `authelia/` com:
    - `docker-compose.yml` — imagem `authelia/authelia`, volume
      `./config:/config`, publicando **só em loopback**:
      `127.0.0.1:9091:9091` (mesma lógica de trava do WatchYourLAN — não
      faz sentido abrir isso pra rede nenhuma, só a NPM local precisa
      alcançar).
    - `.env.example` com placeholders vazios para os três segredos que o
      Authelia exige: `AUTHELIA_JWT_SECRET`, `AUTHELIA_SESSION_SECRET`,
      `AUTHELIA_STORAGE_ENCRYPTION_KEY` (seguindo o padrão de segredos já
      usado no resto do repositório).
    - `config/configuration.yml` — domínio `auth.home`, backend de
      sessão e storage em SQLite local (adequado pro tamanho deste
      homelab), regra de `access_control` liberando autenticação só para
      `wylan.home` (dá pra estender pra outros domínios depois).
    - `config/users_database.yml` — placeholder vazio; a senha real
      **você** gera e cola (passo 11), não eu.
11. `[MANUAL]` Gerar o hash da senha do seu usuário Authelia (o Authelia
    nunca guarda a senha em texto puro, só o hash):
    ```
    docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'SUA_SENHA_AQUI'
    ```
    Cole o hash gerado em `authelia/config/users_database.yml` — pode
    fazer você mesmo diretamente no arquivo, já que é credencial.
12. `[CLAUDE]` `docker compose config` no diretório `authelia/`.
13. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Subir o container do Authelia pela
    primeira vez.
14. `[TESTE - CLAUDE]` `curl 127.0.0.1:9091/api/health` do host →
    esperado responder `200`/`OK`. Conferir logs do Authelia sem erro de
    configuração (chave de sessão ausente, YAML inválido, etc.).
15. `[MANUAL]` Na NPM, criar um Proxy Host novo `auth.home` → Forward
    Hostname/IP `127.0.0.1`, porta `9091` (esse é o portal de login).
16. `[MANUAL]` No Proxy Host `wylan.home` já existente:
    a. Trocar "Forward Hostname/IP" para `127.0.0.1` (igual Plano A).
    b. Na aba "Advanced", colar o snippet de integração (fornecido
       separadamente na hora de executar este passo, porque depende da
       versão exata do Authelia instalada — a sintaxe do `auth_request`
       muda entre versões major do projeto).
    <!-- VERIFICAR: confirmar a versão do Authelia a instalar antes de escrever o snippet final do Advanced, a sintaxe do endpoint /api/verify mudou entre v4.37 e v4.38+ -->
17. `[TESTE - MANUAL]` Pela LAN, acessar `http://wylan.home` — deve
    redirecionar pra `http://auth.home`, pedir login, e ao autenticar
    voltar pro WatchYourLAN.
18. `[TESTE - MANUAL]` Pela LAN, tentar `http://192.168.18.195:8840`
    direto — deve dar conexão recusada (igual Plano A).
19. `[TESTE - MANUAL]` Repetir os dois testes acima pelo Tailscale (mesma
    ressalva de DNS da seção 1).
20. `[MANUAL ou CLAUDE]` Reconfigurar o monitor do Kuma. **Atenção**:
    diferente da Access List da NPM (Basic Auth simples, que o Kuma
    preenche em dois campos), o Authelia usa cookie de sessão — o
    monitor HTTP do Kuma não consegue "logar" sozinho pra checar se o
    WatchYourLAN está de pé atrás da autenticação. O jeito realista de
    manter algum monitoramento é apontar o Kuma pra `http://auth.home` e
    aceitar como "up" qualquer resposta HTTP válida (200 ou redirect de
    login) — isso confirma que a cadeia NPM→Authelia está viva, mas
    **não** confirma que o WatchYourLAN por trás dela está respondendo.
    É uma perda real de granularidade de monitoramento, listada nos
    contras abaixo.
21. `[TESTE - CLAUDE]` Confirmar que o monitor ajustado do Kuma reporta
    "up" com o critério definido no passo 20.

### Prós

- Autenticação de verdade: sessão com cookie, suporte nativo a 2FA
  (TOTP), múltiplos usuários com contas individuais.
- Dá pra estender pra proteger outros `*.home` no futuro com o mesmo
  portal de login (SSO), sem duplicar Access Lists uma por serviço.
- Fica registro de tentativas de login (logs do próprio Authelia).

### Contras

- Serviço novo em produção: mais uma peça pra manter atualizada, mais
  YAML de configuração (`configuration.yml`, `users_database.yml`,
  segredos), mais superfície de erro de configuração.
- Integração com a NPM é por snippet manual colado na aba "Advanced" —
  mais frágil que a Access List nativa; se você editar o Proxy Host
  depois e esquecer de manter o snippet, a proteção cai silenciosamente
  sem aviso.
- Setup inicial bem mais longo que o Plano A: gerar hash de senha,
  escrever `access_control`, criar dois Proxy Hosts em vez de editar um.
- Quebra a integração simples com o Uptime Kuma (ver passo 20) — Basic
  Auth do Plano A funciona nativamente no monitor HTTP do Kuma; sessão do
  Authelia, não.
- Mesma dependência de rede host pra NPM e loopback pro WatchYourLAN que
  o Plano A — Authelia não muda nada na camada de rede, só troca *quem*
  verifica a credencial.

### Rollback

1. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Derrubar o container do Authelia
   (`docker compose down` no diretório `authelia/` — sem `-v`).
2. Restaurar os `.bak` do WatchYourLAN e da NPM (passo 0).
3. `[CLAUDE - PEDIR CONFIRMAÇÃO]` Recriar os dois containers com a
   configuração original.
4. `[MANUAL - SUDO]` `sudo ufw allow 8840/tcp`.
5. Remover o Proxy Host `auth.home` e o snippet "Advanced" do
   `wylan.home` pela interface da NPM.
6. Se decidir não usar Authelia de jeito nenhum, o diretório `authelia/`
   pode ser apagado (nada mais no repositório depende dele).

---

## 5. Comparação rápida

| | Plano A — NPM Access List | Plano B — Authelia |
|---|---|---|
| Serviço novo pra manter | Não | Sim (`authelia/`) |
| Esforço de setup | Baixo | Alto |
| 2FA | Não | Sim |
| Contas individuais | Não (senha única) | Sim |
| Integração nativa com Kuma (Basic Auth) | Sim | Não (perde granularidade) |
| Reaproveita infra existente | Sim (só NPM) | Parcial (NPM + serviço novo) |
| Risco de quebrar proxy hosts existentes | Baixo (auditado: nenhum depende de rede interna) | Igual ao Plano A (mesma mudança de rede na NPM) |
| Extensível pra proteger outros `*.home` depois | Sim, mas uma Access List por serviço | Sim, com um único login central |

Passos 0 (preparação), e os passos 1–9 de cada plano (bloquear
WatchYourLAN em loopback + colocar NPM em modo host) são **idênticos**
nos dois planos — essa parte da decisão já está tomada independente de
qual plano você escolher no fim. A diferença real de esforço e
manutenção está nos passos específicos de cada um (10 em diante).
