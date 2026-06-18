# RKLLama — Pontos de Melhoria (para implementação)

> Documento de tarefas. Cada item tem: **Problema → Solução → Arquivos/Linhas → Ganho**.
> Prioridade 1 = pedidos explícitos do dono. Prioridade 2 = bugs/robustez encontrados na análise.

---

## PRIORIDADE 1 — Comportamento de carga de modelos

### 1.1 Modo "um modelo por vez" (descarregar o anterior ao carregar um novo)

**Problema:**
O rkllama mantém vários modelos em memória ao mesmo tempo (padrão `max_number_models_loaded_in_memory = 9`). Um modelo só sai da memória se expirar (30 min) ou se faltar RAM. Não existe forma de forçar "apenas 1 modelo carregado por vez".

**Solução:**
1. Adicionar nova opção de config em `config_schema.py` (seção `model`):
   ```python
   model.boolean("single_model_mode", True,
       "Se True, descarrega todos os modelos antes de carregar um novo (apenas 1 modelo em memória por vez)")
   ```
   - Também adicionar a chave correspondente no `config/rkllama.ini` (seção `[model]`):
     `single_model_mode = True`

2. Em `worker.py`, método `WorkerManager.add_worker` (linha ~905), **antes** de qualquer
   verificação de memória/domínio, se o modo estiver ativo, descarregar todos os workers existentes:
   ```python
   if model_name not in self.workers.keys():
       # Single-model mode: garante apenas 1 modelo em memória
       if rkllama.config.get("model", "single_model_mode").lower() in ("true","1","yes","on"):
           if self.workers:
               logger.info("single_model_mode ativo: descarregando modelos atuais antes de carregar %s", model_name)
               self.stop_all()
       ...
   ```
   - `stop_all()` já existe (linha ~1140) e itera chamando `stop_worker`.

**Arquivos:** `src/rkllama/config/config_schema.py`, `config/rkllama.ini`, `src/rkllama/api/worker.py:905`

**Ganho:** Uso de RAM previsível; nunca dois modelos disputando NPU/memória ao mesmo tempo; evita swap/lentidão num Orange Pi de RAM limitada.

---

### 1.2 Recusar carga quando não há memória suficiente (com mensagem clara)

**Problema:**
Em `add_worker` (worker.py:925-936), depois de tentar liberar memória com
`unload_oldest_models_from_memory`, o código **não re-verifica** se agora cabe — segue direto
para `create_worker_process`. Se nem descarregando tudo o modelo couber, tenta carregar mesmo
assim, causando crash/OOM ou um erro genérico pouco útil.

**Solução:**
1. Em `worker.py` `add_worker`, após a tentativa de liberar memória, **re-verificar** e abortar
   com motivo explícito:
   ```python
   if not self.is_memory_available_for_model(worker_model.worker_model_info.size):
       self.unload_oldest_models_from_memory(worker_model.worker_model_info.size)

   # Re-verificação: se ainda não cabe, não carrega
   if not self.is_memory_available_for_model(worker_model.worker_model_info.size):
       avail = psutil.virtual_memory().available
       needed = int(worker_model.worker_model_info.size * 1.20)
       msg = (f"Memória insuficiente para carregar '{model_name}': "
              f"necessário ~{needed // (1024*1024)} MB, "
              f"disponível ~{avail // (1024*1024)} MB.")
       logger.error(msg)
       return False, msg
   ```

2. **Mudar a assinatura de retorno** de `add_worker` para `(bool, str|None)` para propagar o motivo.
   Hoje retorna apenas `bool` (e `None` implícito quando o modelo já está carregado — ver item 2.1).
   Padronizar todos os `return`:
   - sucesso: `return True, None`
   - falha de carga genérica: `return False, "mensagem"`
   - já carregado: `return True, None`

3. Em `server.py` `load_model` (linha 164), repassar a mensagem:
   ```python
   model_loaded, load_error = variables.worker_manager_rkllm.add_worker(...)
   if not model_loaded:
       return None, load_error or f"Erro inesperado ao carregar o modelo {model_name}."
   return None, None
   ```
   A mensagem já flui para o cliente nos endpoints (Ollama/OpenAI) que retornam `error`.

**Arquivos:** `src/rkllama/api/worker.py:905-946`, `src/rkllama/server/server.py:164-169`

**Ganho:** Sem crashes por OOM; o cliente recebe HTTP com mensagem explicando que faltou memória,
em vez de erro genérico ou queda do worker.

---

## PRIORIDADE 2 — Bugs e robustez encontrados na análise

### 2.1 `add_worker` retorna `None` quando o modelo já está carregado

**Problema:** worker.py `add_worker` só tem `return` dentro do `if model_name not in self.workers`.
Se o modelo **já está** carregado, cai no fim e retorna `None` (implícito). Em `load_model`,
`if not model_loaded:` trata `None` como falha → retorna erro mesmo com o modelo já em memória.

**Solução:** Adicionar no início, quando já existe:
```python
if model_name in self.workers.keys():
    return True, None
```
(combinar com a nova assinatura `(bool, str|None)` do item 1.2).

**Arquivo:** `src/rkllama/api/worker.py:905`

**Ganho:** Re-requisitar um modelo já carregado não gera erro falso.

---

### 2.2 Off-by-one em `get_available_base_domain_id`

**Problema:** worker.py:885 — na ordem normal usa `range(1, max_domain_id)`, que **exclui** o último
domínio. Com `max=9`, a ordem normal dá 1..8 (8 domínios) e a reversa (`range(max,0,-1)`) dá 9..1
(9 domínios). Contagem inconsistente e um domínio a menos no caminho normal.

**Solução:**
```python
candidates_range = range(1, max_domain_id + 1)   # ordem normal
```

**Arquivo:** `src/rkllama/api/worker.py:880-885`

**Ganho:** Permite de fato o número configurado de modelos; comportamento consistente.
(Menos relevante se `single_model_mode` estiver ligado, mas corrige a lógica.)

---

### 2.3 Cálculo de memória disponível superestimado

**Problema:** worker.py:1019 — `is_memory_available_for_model` soma
`psutil.virtual_memory().available + psutil.virtual_memory().free`. O campo `available` **já inclui**
a memória `free` mais a recuperável (cache). Somar os dois **conta `free` duas vezes** e superestima
a RAM livre, podendo aprovar uma carga que não cabe.

**Solução:**
```python
return psutil.virtual_memory().available > (model_size * 1.20)
```

**Arquivo:** `src/rkllama/api/worker.py:1013-1019`

**Ganho:** Estimativa de memória correta; menos risco de OOM. Reforça o item 1.2.

---

### 2.4 Opções numéricas declaradas como `string` no schema

**Problema:** Em `config_schema.py`, campos como `max_number_models_loaded_in_memory`,
`max_minutes_loaded_in_memory`, `max_seconds_waiting_worker_response`, `max_days_prompt_cache`
são declarados com `.string(...)` e depois convertidos com `int(...)` em vários pontos. Funciona,
mas é frágil (sem validação de faixa, erro só estoura em runtime).

**Solução:** Trocar para `.integer(..., min_value=...)` no schema e remover os `int(...)` espalhados
(ou mantê-los — são inofensivos). Ex.:
```python
model.integer("max_number_models_loaded_in_memory", 9,
    "Máx. de modelos simultâneos em memória", min_value=1, max_value=10)
```

**Arquivo:** `src/rkllama/config/config_schema.py:258-276`

**Ganho:** Validação na carga da config; erros de configuração detectados cedo, não em runtime.

---

## Ordem sugerida de implementação

1. **1.2 + 2.3** (memória) — juntos, pois 1.2 depende de a verificação de 2.3 ser confiável.
2. **2.1** (retorno `None`) — necessário para a nova assinatura `(bool, str)` funcionar bem.
3. **1.1** (single_model_mode) — o pedido principal do dono.
4. **2.2** e **2.4** — correções de robustez, independentes.

## Observações de teste

- Testar em hardware ARM64 real (Orange Pi/RK3588) ou emulado: a inferência NPU exige libs nativas.
- Cenários a validar:
  - Carregar modelo A, depois B → com `single_model_mode=True`, A deve sair antes de B entrar.
  - Carregar modelo maior que a RAM total → deve retornar mensagem de "Memória insuficiente",
    sem derrubar o worker.
  - Re-carregar um modelo já em memória → não deve retornar erro.
