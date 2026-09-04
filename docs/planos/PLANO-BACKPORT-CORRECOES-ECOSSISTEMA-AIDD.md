# PLANO DE BACKPORT — Correções feitas no monorepo ecossistema-aidd

> **Origem:** `C:\Users\trcnologia\Desktop\ecossistema-aidd\tools\aidd-master`
> **Repositório:** `heverton-dev/aidd-master` (padrão neste repositório)
> **Status:** PRONTO PARA EXECUÇÃO
> **Dificuldade:** Alta — o único dos 4 com um bug estrutural (pasta inteira faltando), além de precisar de Docker

---

## 1. CONTEXTO

Mesma origem dos outros dois planos. Este repositório tem o problema mais sério dos 4: `templates/v2/` está listado no `.gitignore` (`templates/v2/` na linha correspondente) **e a pasta nunca existiu aqui** — confirmado que nem no clone atual nem em nenhum outro lugar deste repositório existe esse diretório. Isso quebra a coleta de 2 arquivos de teste completos.

## 2. O QUE MUDA

### 2.1. `.gitignore` — remover a exclusão de `templates/v2/`

`templates/v2/` não é artefato de build — é biblioteca de templates que `scripts/scaffold_infra.py`/`scripts/compose_suite.py` usam em runtime e que os testes importam diretamente via `sys.path`. Estar no `.gitignore` significa que **um `git clone` deste repositório nunca vem com essa pasta**, e `tests/unit/test_database_adapter.py` + `tests/unit/test_cqrs_local_first.py` sempre falham na coleta (`ModuleNotFoundError: No module named 'database'`) em qualquer clone novo.

**Mudança:** remover a linha `templates/v2/` do `.gitignore`.

### 2.2. Restaurar `templates/v2/` (43 arquivos)

A fonte mais confiável é `aidd-master-enterprise` (mesma linhagem, mesma origem) — **mas com uma ressalva importante**: o `database.py` de lá também está desatualizado (301 linhas, sem `RLSConnection`), enquanto o `src/core/database.py` deste próprio repositório (`aidd-master`) tem a versão certa (545 linhas, com `RLSConnection`). Copiar `templates/v2/database.py` direto do enterprise reintroduz o mesmo bug que já foi corrigido no monorepo.

**Passo a passo:**
1. Copiar toda a pasta `templates/v2/` de `C:\Users\trcnologia\Desktop\proj_aidd\aidd-master-enterprise\templates\v2\` pra cá (ou de `ecossistema-aidd/tools/aidd-master-enterprise/templates/v2/`, que já está com o `database.py` velho — tanto faz, o resultado é o mesmo).
2. **Depois**, sobrescrever só `templates/v2/database.py` com o conteúdo de `src/core/database.py` deste mesmo repositório (`aidd-master`) — é o arquivo certo, mais completo.

### 2.3. `requirements.txt` — dependência real faltando

Igual ao plano do enterprise: `redis>=5.0.0` faltando, necessário pro teste `test_redis_driver_raises_clear_error_message_when_url_invalid_scheme`.

### 2.4. `scripts/scaffold_infra.py` — mesmo bug de template Helm do enterprise (linha ~249)

```
resources: {{ toYaml .Values.resources }}
```
→
```
resources:
  {{- toYaml .Values.resources | nindent 12 }}
```

### 2.5. `tests/unit/test_events_driver.py` — mesma race condition do enterprise

Adicionar `driver._redis.ping()` dentro do loop de retry antes de prosseguir, pro mesmo motivo descrito no plano do enterprise.

## 3. COMO APLICAR

Na ordem:
1. Editar `.gitignore` (remover `templates/v2/`).
2. Copiar as 43 arquivos de `templates/v2/` (de qualquer um dos dois lugares citados em 2.2, passo 1).
3. Sobrescrever `templates/v2/database.py` com `src/core/database.py` (passo 2 de 2.2).
4. Copiar `requirements.txt`, `scripts/scaffold_infra.py` e `tests/unit/test_events_driver.py` de `ecossistema-aidd/tools/aidd-master/`.

## 4. VALIDAÇÃO (critério de aceite — nesta ordem)

```bash
pip install -r requirements.txt
python -m pytest -q
```
Esperado: sem nenhum erro de coleta (`ModuleNotFoundError` não pode mais aparecer). `helm`/`terraform` como `SKIPPED` se não instalados é normal.

**Se o Docker Desktop estiver rodando**, mesma validação de race condition do plano do enterprise:
```bash
docker rmi redis:7
python -m pytest tests/unit/test_events_driver.py::test_event_emitted_on_instance_a_is_processed_on_instance_b_via_redis -v
```
Repetir 2-3 vezes com `docker rmi redis:7` entre cada.

## 5. COMMIT E PUSH

Pode ser 1 commit só ou 2 (um pro `templates/v2` + `.gitignore`, outro pros 3 arquivos de teste/infra) — o segundo formato deixa mais claro no histórico que são dois problemas diferentes. Depois, `git push origin main`.
