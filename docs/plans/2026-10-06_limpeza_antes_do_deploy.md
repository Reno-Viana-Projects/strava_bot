# Deploy limpa o lixo do Docker antes de medir o espaço

**Data:** 2026-10-06
**Status:** Aprovado e implementado

## Projetos Afetados

- `assistant_bot`: `scripts/deploy_compose.sh`, `.github/workflows/update_base_image.yml`, teste e CLAUDE.md.
- `.github/workflows/deploy.yml` em `assistant_schedule`, `assistant_api`, `assistant_promotion`, `carros_app`,
  `metro_bot`, `soro_bot`, `posto_bot`, `strava_bot` e `weednotifier`.

## 1. Objetivo

Em 06/10/2026 o deploy do `carros_app` parou com 4 GB livres, abaixo do mínimo de 5 GB. A limpeza (`docker image
prune -f` e `docker builder prune --keep-storage 10GB`) só rodava **no fim** de um deploy bem-sucedido. Cada *Update
Base Image* deixa a base antiga sem nome no disco. Com o disco perto do limite, o deploy desistia antes da limpeza e
o espaço nunca voltava sozinho.

## 2. O que muda

- A mesma limpeza segura roda **antes** da checagem de espaço:
  - no `deploy_compose.sh`, que o redeploy e o deploy do `assistant_bot` usam;
  - no *Update Base Image*, antes do build da base;
  - nos `deploy.yml` dos outros 9 repositórios.
- A limpeza do fim continua (tira o lixo do build novo).
- **Nunca** a limpeza com `-a`, que apagaria a `assistant-base-python` e a `assistant-schedule`.
- Se faltar espaço mesmo depois da limpeza, o deploy continua desistindo sem tocar nos containers.

## 3. Testes

`assistant_bot/tests/test_deploy_compose.py`:
- a limpeza vem antes do build;
- sem espaço, só a limpeza roda (nenhum build, up ou down);
- o build que falha não troca os containers;
- o *Update Base Image* limpa antes de medir e nunca usa `-a`.

Os workflows dos outros repos foram conferidos como YAML válido.

## 4. Checklist

- [x] `deploy_compose.sh` e `update_base_image.yml` com a limpeza antes, e o teste
- [x] `deploy.yml` dos 9 repos
- [x] CLAUDE.md (`assistant_bot`)
- [x] Plano em `docs/plans/2026-10-06_limpeza_antes_do_deploy.md` em cada repo, no mesmo commit
