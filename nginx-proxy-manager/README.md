# Nginx Proxy Manager

Proxy reverso com interface web, usado para expor os serviços sob
subdomínios `*.home`.

## Portas

- `80:80` — HTTP
- `81:81` — interface de administração
- `443:443` — HTTPS

## Dependências

- Containers: nenhum.
- Rede: nenhuma rede compartilhada declarada; usa a rede padrão do Compose.
- Volumes do host: `./data:/data` (config, hosts proxeados, banco sqlite),
  `./letsencrypt:/etc/letsencrypt` (certificados).

## Variáveis de ambiente

Nenhuma declarada no compose.

## Gotchas

- Sem `user:` nem `PUID`/`PGID` — o container roda como root; arquivos
  criados em `./data` e `./letsencrypt` ficam com dono `root` no host
  (ex.: `letsencrypt/accounts/`, ilegível para o usuário `lukas`).
- `data/keys.json` (par de chaves RSA usado para assinar sessões) é gerado
  pela própria aplicação, não vem do compose.
- <!-- VERIFICAR: a porta 443 está mapeada no compose, mas o estado
  documentado no CLAUDE.md diz "HTTP apenas, sem TLS". Confirmar se há uso
  real da 443 ou se é só a porta padrão da imagem, sem certificado
  configurado. -->
