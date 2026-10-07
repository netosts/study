# OpenCode com Docker

Este projeto utiliza o [OpenCode](https://opencode.ai/) dentro de um container Docker para manter o ambiente do agente isolado da máquina host.

A ideia principal é permitir que o OpenCode tenha acesso apenas ao projeto atual, sem montar todo o diretório pessoal (`$HOME`) dentro do container.

## Como funciona

O `compose.yaml` monta apenas:

```text
Projeto atual
    │
    └── /workspace

Volume global do OpenCode
    └── /root/.local/share/opencode

Configurações globais do OpenCode
    └── /root/.config/opencode
```

O projeto atual é montado através de:

```yaml
- .:/workspace
```

Isso significa que o OpenCode consegue acessar os arquivos da pasta onde o `compose.yaml` está localizado, mas não recebe acesso automático ao restante do computador.

Os dados e configurações do OpenCode ficam em volumes Docker persistentes, permitindo reutilizar login e configurações entre diferentes execuções.

---

## Pré-requisitos

### 1. Docker

É necessário ter Docker instalado.

Verifique com:

```bash
docker --version
```

E:

```bash
docker compose version
```

---

### 2. Criar os volumes globais

O `compose.yaml` utiliza dois volumes externos:

```yaml
opencode-global-data
opencode-global-config
```

Eles precisam ser criados uma única vez:

```bash
docker volume create opencode-global-data
docker volume create opencode-global-config
```

Verifique se foram criados:

```bash
docker volume ls
```

Você deverá encontrar:

```text
opencode-global-data
opencode-global-config
```

Como são volumes `external`, eles não são apagados ao remover containers ou ao executar:

```bash
docker compose down
```

---

## Context7

O container carrega variáveis de ambiente através de:

```yaml
env_file:
  - ${HOME}/.config/opencode/context7.env
```

Portanto, o arquivo precisa existir no host:

```text
~/.config/opencode/context7.env
```

Exemplo:

```bash
mkdir -p ~/.config/opencode
nano ~/.config/opencode/context7.env
```

Coloque nesse arquivo somente as variáveis necessárias para o Context7.

> Não coloque o arquivo `context7.env` dentro do repositório.

Como ele está fora do projeto, credenciais ou tokens não serão acidentalmente versionados junto com o código.

---

## Executando o OpenCode

Na raiz do projeto, onde está o `compose.yaml`:

```bash
docker compose run --rm opencode
```

Esse é o comando principal.

### O que ele faz

`docker compose run`

Cria um novo container usando a configuração do serviço `opencode`.

`--rm`

Remove o container automaticamente quando o OpenCode for encerrado.

Os dados importantes continuam persistidos nos volumes:

```text
opencode-global-data
opencode-global-config
```

Portanto, remover o container não remove as configurações ou sessões persistidas nesses volumes.

---

## Encerrando

Dentro do OpenCode, encerre normalmente a aplicação.

Como o container foi iniciado com:

```bash
docker compose run --rm opencode
```

ele será removido automaticamente depois da saída.

É possível conferir containers ativos com:

```bash
docker ps
```

E containers antigos com:

```bash
docker ps -a
```

---

## Atualizando o OpenCode

A imagem utilizada é:

```yaml
image: ghcr.io/anomalyco/opencode
```

Para baixar a versão mais recente:

```bash
docker compose pull
```

Depois basta iniciar novamente:

```bash
docker compose run --rm opencode
```

---

## Isolamento

O ponto mais importante dessa configuração é:

```yaml
volumes:
  - .:/workspace
```

Somente o diretório atual é disponibilizado ao OpenCode como workspace.

Por exemplo, se o projeto estiver em:

```text
~/desktop/study/django-drf
```

o container verá esse diretório como:

```text
/workspace
```

Mas diretórios como:

```text
~/Documents
~/Downloads
~/Pictures
~/Desktop/outro-projeto
~/.ssh
```

não são automaticamente montados no container.

Isso reduz bastante o risco de um agente acessar arquivos de outros projetos ou informações pessoais da máquina.

### Importante

Docker não transforma o OpenCode em um ambiente completamente offline.

O container ainda pode acessar a internet, o que é necessário para utilizar os modelos, APIs e MCPs configurados.

O isolamento aqui é principalmente de **filesystem**:

```text
Host
│
├── projeto-atual ───────────────► /workspace
│
├── outros-projetos              ✕
├── ~/.ssh                       ✕
├── Documents                    ✕
├── Downloads                    ✕
└── outros arquivos pessoais     ✕
```

---

## Segurança adicional

O Compose utiliza:

```yaml
security_opt:
  - no-new-privileges:true
```

Essa opção impede que processos dentro do container adquiram novos privilégios através de mecanismos como `setuid` ou `setgid`.

Ela é uma camada adicional de proteção, mas não substitui o isolamento correto dos volumes.

---

## Persistência

Existem dois volumes globais.

### Dados

```text
opencode-global-data
```

Montado em:

```text
/root/.local/share/opencode
```

Utilizado para dados persistentes do OpenCode.

### Configurações

```text
opencode-global-config
```

Montado em:

```text
/root/.config/opencode
```

Utilizado para configurações persistentes.

Isso permite utilizar o mesmo ambiente do OpenCode em vários projetos sem montar o `$HOME` do host dentro do container.

---

## Usando em outro projeto

Para utilizar esse mesmo ambiente em um novo projeto, basta copiar o `compose.yaml` para a raiz dele.

Exemplo:

```text
projects/
├── projeto-a/
│   ├── compose.yaml
│   └── ...
│
└── projeto-b/
    ├── compose.yaml
    └── ...
```

Ao executar:

```bash
cd projeto-a
docker compose run --rm opencode
```

o OpenCode verá:

```text
projeto-a → /workspace
```

Ao executar:

```bash
cd projeto-b
docker compose run --rm opencode
```

ele verá:

```text
projeto-b → /workspace
```

Os dois projetos podem compartilhar as configurações globais do OpenCode através dos volumes externos, mas cada execução recebe como workspace somente o projeto atual.

---

## `compose.yaml`

Configuração utilizada:

```yaml
services:
  opencode:
    image: ghcr.io/anomalyco/opencode
    stdin_open: true
    tty: true
    working_dir: /workspace

    volumes:
      - .:/workspace
      - opencode-global-data:/root/.local/share/opencode
      - opencode-global-config:/root/.config/opencode

    security_opt:
      - no-new-privileges:true

    env_file:
      - ${HOME}/.config/opencode/context7.env

volumes:
  opencode-global-data:
    external: true

  opencode-global-config:
    external: true
```

---

## Comandos úteis

Iniciar:

```bash
docker compose run --rm opencode
```

Atualizar a imagem:

```bash
docker compose pull
```

Ver containers ativos:

```bash
docker ps
```

Ver todos os containers:

```bash
docker ps -a
```

Ver volumes:

```bash
docker volume ls
```

Inspecionar o volume de dados:

```bash
docker volume inspect opencode-global-data
```

Inspecionar o volume de configuração:

```bash
docker volume inspect opencode-global-config
```

---

## Resumo

Fluxo normal de uso:

```bash
cd meu-projeto

docker compose pull       # opcional, para atualizar

docker compose run --rm opencode
```

Na primeira configuração da máquina:

```bash
docker volume create opencode-global-data
docker volume create opencode-global-config

mkdir -p ~/.config/opencode
```

Depois disso, normalmente será necessário apenas:

```bash
docker compose run --rm opencode
```

### Objetivo da configuração

- manter o OpenCode dentro de Docker;
- expor somente o projeto atual ao agente;
- evitar montar o `$HOME` inteiro;
- persistir configurações entre execuções;
- compartilhar configuração do OpenCode entre projetos;
- manter secrets fora dos repositórios;
- reduzir o risco de acesso acidental a arquivos de outros projetos.
