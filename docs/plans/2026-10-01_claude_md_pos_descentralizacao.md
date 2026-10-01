# CLAUDE.md conferidos após a descentralização + pontos de melhoria

**Data:** 2026-10-01  
**Status:** Aprovado e implementado

## Projetos Afetados

- [x] `CLAUDE.md` da pasta raiz do ecossistema (não versionado)
- [x] Bloco `regras-ecossistema` nos 9 `assistant_*`, `carros_app` e `carros-mcp`
- [x] `strava_bot`, `metro_bot`, `soro_bot`, `posto_bot`, `weednotifier`
- [x] `assistant_bot` (compose, `deploy.yml`, `.env.example`, docs)
- [x] `carros_app` (testes, README, `.env.example`, `CLAUDE.md`)
- [x] `carros-mcp` (`CLAUDE.md`)

## 1. Objetivo

A descentralização do compose (um `docker-compose.yml` e um `deploy.yml` por repo) está concluída. Os `CLAUDE.md`
passam a tratá-la assim, todo repo com deploy próprio passa a ter as regras do ecossistema, e os pontos de melhoria
levantados na revisão são corrigidos.

## 2. Escopo das Alterações

### 2.1 `CLAUDE.md` (documentação)
- Bloco comum: a lista de projetos inclui os repos irmãos (`carros_app`, `carros_mcp`, `metro_bot`, `soro_bot`,
  `posto_bot`, `strava_bot`, `weednotifier`); o marcador diz "todos os repos do ecossistema". Idêntico nos 16.
- `strava_bot`: ganha o bloco e a seção de deploy. `metro_bot`, `soro_bot`, `posto_bot` e `weednotifier` ganham
  `CLAUDE.md` (bloco, visão geral, legado conhecido, deploy e testes).
- Raiz: "Pendências da descentralização" vira "Pontos de atenção"; legado `reset.yml`/`update_assistant.sh` fora
  de uso; lista correta de `TEMPLATE.md`; repos irmãos no texto do bloco; armadilha do `--env-file`.
- Trechos pontuais: `assistant_bot` (legado fora de uso), `carros_app` (o `assistant_bot` fornece `/carro`, Redis
  e imagem base), `carros-mcp` (dependência nova = *Update Base Image* + deploy dos consumidores).

### 2.2 Pontos de melhoria (código e configuração)
1. **`carros_app`:** os testes do visitante passam a seguir `config.visitante_buscas` (o padrão virou 3), com testes
   novos para o 403 no lugar do 429 quando o limite por minuto estoura sem consultas (visitante e free) e para o
   VIP, que continua recebendo 429. `.env.example` e README com 3 buscas.
2. **`metro_bot`:** `docker-compose.yml` versionado (serviço `metro_bot`, rede `webnet`, `env_file: ../.env`), no
   mesmo padrão dos outros bots; antes o `deploy.yml` rodava sem compose no repo.
3. **`assistant_bot`:** senhas do WordPress/MariaDB saem do compose e vêm de `MYSQL_ROOT_PASSWORD` e
   `WORDPRESS_DB_PASSWORD` (com `:?`, o compose para se faltarem); chaves no `.env.example`.
4. **`assistant_bot`:** `deploy.yml` roda `docker-compose --env-file ../.env`, para a interpolação
   (`REDIS_PASSWORD`, senhas do WordPress) ler o mesmo `.env` que os containers.

## 3. Fluxo

```
push na main (repo de serviço) → deploy.yml → git pull → docker-compose [--env-file ../.env] down/up -d --build
```

## 4. Decisões e Riscos

| Decisão | Justificativa |
|---|---|
| Testes leem `config.visitante_buscas` | O número é configurável; o teste confere o mecanismo, não o valor |
| `:?` nas senhas do WordPress | Melhor o deploy parar com erro claro do que subir o WordPress com senha vazia |
| Não trocar a senha do MariaDB agora | Com o volume já criado, a variável é ignorada; trocar exige `ALTER USER` e é passo manual |

**Riscos:** antes do próximo push do `assistant_bot`, `MYSQL_ROOT_PASSWORD` e `WORDPRESS_DB_PASSWORD` precisam estar
em `assistant_project/.env` (com os valores atuais), senão o deploy dele para. A senha antiga está no histórico do
git: trocar no banco e no `.env` é recomendado.

## 5. Testes

- `carros_app/backend`: `python -m pytest` (181 passando).
- Composes e workflows alterados: YAML validado; sem Docker na máquina local para `docker-compose config`.
- Bloco comum: hash idêntico nos 16 `CLAUDE.md`.

## 6. Deploy

- Commits locais, um por repo, **sem push**: push na `main` de repo de serviço é deploy.
- Antes do push do `assistant_bot`: senhas do WordPress no `assistant_project/.env`.
- O `carros_app` já está em produção com as 3 buscas; o push dele só leva testes e docs.

## 7. Checklist de Entrega

- [x] Bloco idêntico nos 16 `CLAUDE.md`
- [x] `CLAUDE.md` novos nos bots
- [x] Pontos de melhoria 1 a 4, com testes do `carros_app` passando
- [x] Plano arquivado em `docs/plans/` de cada repositório afetado
