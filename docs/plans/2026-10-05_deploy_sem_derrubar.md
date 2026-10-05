# Deploy que não derruba serviço quando o build falha

**Data:** 2026-10-05
**Status:** Aprovado

## Projetos Afetados

| Repo | Arquivos |
|---|---|
| `assistant_bot` | `.github/workflows/deploy.yml`, `.github/workflows/update_base_image.yml`, `scripts/deploy_compose.sh` (novo), `CLAUDE.md`, `docs/` |
| `assistant_schedule` | `.github/workflows/deploy.yml`, `CLAUDE.md` |
| `carros_app` | `.github/workflows/deploy.yml`, `CLAUDE.md`/`AGENTS.md` |
| `assistant_api`, `assistant_promotion`, `metro_bot`, `soro_bot`, `posto_bot`, `strava_bot`, `weednotifier` | `.github/workflows/deploy.yml` (mesma troca; entram na sessão com `add_repo` na implementação) |

Sem mudança de código de aplicação, model, contrato de API ou regra de acesso.

## 1. Objetivo

Em 05/10/2026 dois problemas de deploy tiraram o ecossistema do ar:

1. **Disco cheio + `down` antes do build.** O redeploy do *Update Base Image* (e cada `deploy.yml`) faz
   `docker-compose down` e só depois `up -d --build`. Com o disco cheio, o build falhou (`no space left on device`)
   e 9 serviços ficaram parados: `assistant_api`, `assistant_schedule`, `assistant_promotion`, `carros_app`,
   `metro_bot`, `soro_bot`, `posto_bot`, `strava_bot`, `weednotifier`.
2. **Imagem compartilhada do `assistant_schedule`.** Só o `pessoal_schedule` tem `build`; os outros 14 serviços
   usam `image: assistant-schedule`. Depois de um `docker image prune -af`, o `up` tenta baixar essa imagem do
   Docker Hub antes de construir e falha com `pull access denied`.

Objetivo: um deploy que falha deixa os containers antigos rodando, o deploy recusa começar sem espaço em disco, e a
imagem compartilhada sempre é construída antes de subir.

## 2. Fluxo novo (o mesmo nos `deploy.yml` e no redeploy)

```
git pull
→ espaço livre >= MIN (5 GB) em /var/lib/docker?  não → aborta sem tocar nos containers (avisa no Telegram)
→ docker-compose --env-file ../.env build          falhou → aborta; containers antigos seguem no ar
→ docker-compose --env-file ../.env up -d --remove-orphans   (recria só o que mudou; sem `down`)
→ docker image prune -f; docker builder prune -f --keep-storage 10GB
```

- **Sem `down`:** o `up -d` depois do `build` já recria os containers cuja imagem ou config mudou. O
  `--remove-orphans` tira serviço que saiu do compose (o que o `down` fazia por nós).
- **`build` antes do `up`:** resolve o item 2 (o `pessoal_schedule` gera `assistant-schedule` antes de os outros
  14 precisarem dela) e garante que a imagem nova existe antes de trocar qualquer container.
- **Script único:** `assistant_bot/scripts/deploy_compose.sh <pasta>` com a checagem de espaço, o build, o up e a
  limpeza. O redeploy do *Update Base Image* chama o script para cada repo. Cada `deploy.yml` traz as mesmas
  linhas inline, porque o servidor pode não ter o `assistant_bot` atualizado no momento do deploy de outro repo; o
  CLAUDE.md de cada repo diz que as duas cópias andam juntas.
- **Base:** o *Update Base Image* também checa o espaço antes do `docker build` da base e aborta sem redeploy se
  faltar.
- **Limpeza:** o `builder prune --keep-storage 10GB` segura o cache de build, que foi o que encheu o disco, sem
  zerar o cache (o build da base continua rápido). Nunca `image prune -a` no workflow: apaga a `assistant-schedule`
  e a `assistant-base-python`.

## 3. Decisões e riscos

| Decisão | Por quê |
|---|---|
| Sem `down` | É o `down` que deixa o serviço parado quando o build falha |
| `build` explícito antes do `up` | Ordem garantida para a imagem compartilhada; falha de build não toca nos containers |
| Limite de 5 GB, configurável por variável no topo do script | O build da base precisa de alguns GB; abaixo disso falhar cedo é melhor |
| `pull_policy` não usado | Depende da versão do `docker-compose` do servidor (confirmar com `docker-compose version`); o `build` antes já resolve |

Riscos:
- **Sem `down`, a rede e os volumes não são recriados.** Mudança de rede/volume no compose passa a precisar de
  `down` manual; isso fica escrito no CLAUDE.md do `assistant_bot`.
- **Redis:** o deploy do `assistant_bot` deixa de reiniciar o `redis` quando ele não muda. O cache das buscas
  deixa de zerar a cada deploy, o que é melhor; o CLAUDE.md que diz que ele "recomeça vazio" é atualizado.
- **WordPress e volumes externos:** continuam iguais (`external`, nome fixo).
- A primeira execução depois do merge ainda usa o fluxo antigo nos repos que não receberam o `deploy.yml` novo.

## Notas da implementação

- O espaço livre é medido na pasta do Docker (`docker info -f '{{.DockerRootDir}}'`, ou `/var/lib/docker`).
- Os `deploy.yml` dos outros repos mantêm o jeito de cada um chamar o compose (sem `--env-file`, como antes); só o
  do `assistant_bot` usa `--env-file ../.env`, pela interpolação das senhas.
- O ensaio do script virou teste permanente: `assistant_bot/tests/test_deploy_compose.py` roda o script com
  `docker` e `docker-compose` falsos (caminho feliz, sem espaço, build com falha) e confere que os workflows usam o
  script e não fazem `down`.
- `shellcheck` não está instalado no container; o script passou no `bash -n` e no teste.

## 4. Testes

Workflow não tem suíte de teste nos repos. Verificação:
- `bash -n scripts/deploy_compose.sh` e `shellcheck` (quando instalado) no `assistant_bot`.
- Dry-run local do script com `docker-compose` falso no `PATH` (registra os comandos) cobrindo: espaço baixo
  aborta antes de tudo; build com falha não chama `up`; caminho feliz chama `build`, `up -d --remove-orphans` e a
  limpeza, sem `down`.
- Depois do push (quando pedido): um deploy normal de um repo pequeno (`weednotifier` ou `assistant_schedule`) e o
  *Update Base Image* com `redeploy=true`, conferindo no log que não há `down` e que todos sobem.

## 5. Deploy

Push nos repos (cada um faz o próprio deploy, já com o fluxo novo). O `assistant_bot` primeiro, porque o
redeploy do *Update Base Image* passa a chamar o script dele.

## 6. Checklist

- [x] `scripts/deploy_compose.sh` + chamada no `update_base_image.yml` (checagem de espaço também antes da base)
- [x] `deploy.yml` sem `down` em `assistant_bot`, `assistant_schedule`, `carros_app`, `assistant_api`,
      `assistant_promotion`, `metro_bot`, `soro_bot`, `posto_bot`, `strava_bot`, `weednotifier`
- [x] Dry-run do script com os três cenários
- [x] CLAUDE.md: `assistant_bot` (seção Infra: fluxo novo, `down` manual para rede/volume, Redis não zera mais a
      cada deploy, nunca `image prune -a`), `assistant_schedule` (imagem compartilhada exige `build` antes),
      `carros_app` (Deploy e infra)
- [x] Plano em `docs/plans/2026-10-05_deploy_sem_derrubar.md` em cada repo afetado, no mesmo commit
- [x] Um commit por repo, em português (`fix: deploy só troca os containers depois do build e não começa sem
      espaço em disco`)
