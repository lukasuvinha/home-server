# Dockge

Interface web para gerenciar os stacks Docker deste repositório — editar,
subir e derrubar `docker-compose.yml` pela UI.

## Portas

- `5001:5001` — interface web

## Dependências

- Containers: nenhum.
- Rede: nenhuma rede compartilhada declarada.
- Volumes do host: `/var/run/docker.sock` (controle do Docker do host),
  `/opt/dockge/data` (dados do próprio Dockge), `/home/lukas/docker`
  montado no **mesmo caminho** dentro e fora do container.

## Variáveis de ambiente

- `DOCKGE_STACKS_DIR`
- `PUID`, `PGID`
- `DOCKGE_ENABLE_CONSOLE`

## Gotchas

- O bind mount do `docker.sock` dá ao container controle total sobre o
  Docker do host.
- O volume `/home/lukas/docker:/home/lukas/docker` precisa ter o mesmo
  caminho dentro e fora do container — é assim que o Dockge consegue
  editar os composes no lugar certo. Mover o repositório no host exige
  atualizar esse volume e o `DOCKGE_STACKS_DIR` junto.
- Existe um `docker-compose.yaml.save` no mesmo diretório — não é
  referenciado por nenhum comando; conteúdo idêntico ao template padrão do
  Dockge, não ao compose em uso.
