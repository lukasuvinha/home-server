# Filebrowser

Navegador de arquivos via web.

## Portas

- `8082:80`

## Dependências

- Containers: nenhum.
- Rede: nenhuma rede compartilhada declarada.
- Volumes do host: `/home/lukas:/srv/home`, `/:/srv/root:ro` (raiz do
  host inteira, somente leitura), `/mnt/hd2tb:/srv/hd2tb`,
  `./filebrowser/database.db:/database/filebrowser.db`,
  `./filebrowser/config:/config`.

## Variáveis de ambiente

- `PUID`, `PGID`, `TZ`

## Gotchas

- Bind mount de `/` inteiro (somente leitura) — a interface web enxerga
  todo o filesystem do host, não só as pastas de mídia.
- Os volumes usam caminho relativo `./filebrowser/...` de dentro do
  próprio diretório `filebrowser/`, o que cria um subdiretório aninhado
  `filebrowser/filebrowser/` — é ali que o banco e o `config/` realmente
  ativos ficam, não na raiz do stack.
- <!-- VERIFICAR: existe um arquivo `catalogo-betor.yml` neste diretório,
  não referenciado por este compose e com conteúdo quase idêntico a ele.
  Qual é o propósito desse arquivo? -->
