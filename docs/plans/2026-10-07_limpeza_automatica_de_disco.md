# Limpeza automática de disco no deploy

**Data:** 2026-10-07
**Status:** Aprovado em 2026-10-07 ("aprovar") e implementado

## Projetos Afetados
- `assistant_bot`: `scripts/deploy_compose.sh` (limpeza em dois níveis e `--so-liberar`), `update_base_image.yml`,
  testes e CLAUDE.md.
- `assistant_api`, `assistant_schedule`, `assistant_promotion`, `carros_app`, `metro_bot`, `soro_bot`, `posto_bot`,
  `strava_bot` e `weednotifier`: o `deploy.yml` passa a chamar o script único; CLAUDE.md.

## 1. Objetivo
O *Update Base Image* com redeploy de 07/10/2026 parou os 9 repos com "só 4 GB livres (mínimo 5 GB)". A base nova
ocupa alguns GB, a antiga continua presa às imagens dos serviços até eles serem reconstruídos, e a limpeza leve
(imagens sem nome e cache de build acima de 10 GB) não bastava. O deploy passa a resolver isso sozinho.

## 2. O que muda
1. **`deploy_compose.sh`, limpeza em dois níveis:**
   - **nível 1 (como antes):** imagens sem nome e cache de build acima de 10 GB;
   - **nível 2 (só se ainda faltar espaço):**
     - todo o cache de build (`builder prune -af`);
     - containers parados;
     - imagens que nenhum container usa (`docker rmi` sem `-f`), menos as protegidas: `assistant-base-python`,
       `assistant-imagem-python`, `assistant-schedule` e as dos `docker-compose.yml` dos repos;
   - **volume nunca;**
   - **sem 5 GB nem assim:** falha sem mexer em nada e mostra o `docker system df` e as 10 maiores imagens no log.
2. **`--so-liberar [pasta]`:** só as limpezas, sem deploy. O *Update Base Image* usa antes de construir a base (falha
   se não houver espaço) e de novo antes do redeploy.
3. **Script único:** o `deploy.yml` dos 9 repos troca o bloco copiado por
   `git -C ../assistant_bot pull -q || true` + `bash ../assistant_bot/scripts/deploy_compose.sh .`.

## 3. Decisões e riscos

| Decisão | Por quê |
|---|---|
| `docker rmi` sem `-f`, em vez de `image prune -a` | O docker recusa apagar imagem em uso por container; a lista de protegidas cobre a base e a do schedule, que não têm container próprio |
| Nível 2 só quando falta espaço | Apagar todo o cache de build deixa o próximo build mais lento |
| Script pelo clone do `assistant_bot` no servidor | Uma regra só para os 10 repos; o clone já existe e é atualizado pelo `git pull` antes |

## 4. Testes
- `assistant_bot/tests/test_deploy_compose.py`, com o docker trocado por um dublê:
  - o nível 2 só entra quando o nível 1 não basta;
  - quais imagens saem (as protegidas, as dos composes e as `<none>` ficam);
  - nada de `rmi -f`, `image prune -a` ou volume;
  - a falha final mostra o diagnóstico;
  - o `--so-liberar`;
  - o workflow chama o script antes da base e antes do redeploy.
- YAML dos 10 workflows conferido.

## 5. Checklist
- [x] Script, testes, workflow e CLAUDE.md no `assistant_bot`
- [x] `deploy.yml` e CLAUDE.md nos 9 repos
- [x] Plano em `docs/plans/2026-10-07_limpeza_automatica_de_disco.md` nos 10 repos
- [ ] Push e *Update Base Image* com redeploy


## Adendo (aprovado em 2026-10-07): um deploy por vez, sinal de vida e limite de tempo

O primeiro uso, com os 10 deploys disparados juntos, mostrou dois problemas:
- **`assistant_bot`:** o build ficou 1h43 parado em "exporting layers", enquanto os outros deploys apagavam o cache
  de build.
- **`assistant_schedule`:** a conexão SSH caiu (`ECONNRESET`) depois de minutos sem nenhuma saída no log.

Correção:
1. **Trava (`flock` em `/tmp/deploy_compose.lock`):** um deploy por vez no servidor. Quem chega espera até 30 min,
   escrevendo no log, e depois desiste sem mexer em nada.
2. **`com_sinal`:** a limpeza do cache, o build e o `up` rodam com `timeout` (10, 25 e 10 min) e escrevem
   "... ainda rodando" a cada 30 s. Estourou o tempo, o deploy falha sem trocar os containers.
3. **Limite do job:** `timeout-minutes: 40` no `deploy.yml` dos 10 repos. O *Update Base Image* fica com 150 e
   `command_timeout` de 140 min, porque faz o redeploy dos 10 repos em série.
4. **Testes:** a trava ocupada (espera e desistência), o sinal de vida e o build que passa do limite.
