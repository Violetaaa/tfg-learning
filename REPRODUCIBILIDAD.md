# Guía de reproducibilidad: trazas, versiones y experimentos

Notas de diseño para el desarrollo y la experimentación del TFG.
Objetivo: que cada número de la memoria se pueda rastrear hasta la ejecución
concreta que lo produjo, y que esa ejecución se pueda repetir.

---

## 1. Principio general

- La lógica del agente vive en una biblioteca (`agente/`) con un único punto
  de entrada: `run(pregunta, estrategia, modelo) -> Resultado`.
- `run()` **siempre** registra la traza, la llame quien la llame: CLI,
  scripts de experimentos o interfaz.
- La interfaz (si existe) solo **muestra** el `Resultado`. No contiene lógica
  del agente ni es responsable de guardar nada.

```
CLI ──────────┐
Experimentos ─┼──► run(pregunta, estrategia, modelo) ──► Resultado
Interfaz ─────┘                │
                               └──► resultados/<experimento>.jsonl
```

---

## 2. Trazas de ejecución

### Qué son
El registro completo de lo que hizo el agente para resolver una pregunta.
Son los **datos experimentales**: de ellas salen las métricas, el análisis
de errores y los ejemplos de la memoria.

No confundir con la **persistencia de conversaciones** (historial de chats
para el usuario), que queda **fuera del alcance**.

### Formato
- Un fichero **JSONL por experimento** (una traza por línea).
  Ejemplo: `resultados/exp01_react_<modelo>.jsonl`
- Las trazas de los experimentos finales **no se editan nunca**. Los análisis
  las leen; no las modifican.

### Campos obligatorios

| Campo | Para qué |
|---|---|
| `id_ejecucion` | Identificador único |
| `fecha` | Cuándo se ejecutó |
| `pregunta_id`, `pregunta` | Qué se preguntó |
| `conjunto` | `dev` o `test` |
| `estrategia` | `sin_herramientas`, `react`, `plan_execute`… |
| `modelo` | Nombre y versión **exactos** del modelo |
| `parametros` | Temperatura, máx. tokens, límite de pasos |
| `repeticion` | Nº de repetición (1..N) |
| `git_commit` | Commit del código ejecutado |
| `repo_sucio` | `true` si había cambios sin commitear |
| `hash_prompt` | Huella del prompt de sistema + plantilla de estrategia |
| `hash_herramientas` | Huella de las definiciones de herramientas |
| `plan` | Plan generado (si la estrategia lo produce) |
| `pasos[]` | Cada llamada: herramienta, argumentos, resultado, error, ms |
| `respuesta_final` | Texto devuelto |
| `tokens` | Entrada y salida |
| `duracion_s` | Tiempo total |
| `error` | Error global (no termina, límite de pasos…) o `null` |

---

## 3. Versionado de prompts

### Qué cuenta como "prompt"
Todo el texto fijo que recibe el modelo además de la pregunta:
1. Prompt de sistema (instrucciones generales).
2. Plantilla de cada estrategia (ReAct, plan-and-execute…).
3. **Descripciones y esquemas de las herramientas.** También influyen en el
   comportamiento y también se versionan.

### Por qué
El prompt es una variable del experimento, igual que el modelo. Sin
registrar su versión no se puede saber si una mejora viene del prompt, del
modelo o del código, ni reproducir resultados antiguos.

### Cómo
- Prompts en ficheros, no incrustados en el código:
  ```
  prompts/sistema.txt
  prompts/react.txt
  prompts/plan_execute.txt
  agente/herramientas/definiciones.py
  ```
- Todo bajo git.
- En cada traza: `git_commit`, `repo_sucio`, `hash_prompt`, `hash_herramientas`.
- El hash se calcula sobre el **texto realmente enviado**, así detecta cambios
  aunque se olvide renombrar la versión.

```python
import hashlib, json, subprocess

def huella(texto: str) -> str:
    return hashlib.sha256(texto.encode()).hexdigest()[:10]

def info_git() -> dict:
    commit = subprocess.check_output(
        ["git", "rev-parse", "--short", "HEAD"]).decode().strip()
    sucio = bool(subprocess.check_output(
        ["git", "status", "--porcelain"]).decode().strip())
    return {"git_commit": commit, "repo_sucio": sucio}

traza.update(info_git())
traza["hash_prompt"] = huella(prompt_sistema + prompt_estrategia)
traza["hash_herramientas"] = huella(json.dumps(tools, sort_keys=True))
```

- Opcional: nombre legible (`react_v3`) para tablas, pero el hash es la
  referencia fiable.

---

## 4. Desarrollo frente a test

| Fase | Conjunto | Prompts | Regla |
|---|---|---|---|
| Desarrollo | `dev` | Se cambian libremente | Nunca mirar resultados de `test` |
| Congelación | — | Se fija una versión por estrategia | Tag de git, p. ej. `experimento-final` |
| Test | `test` | Congelados | Se ejecuta con el tag; todas las trazas comparten hashes |

- El conjunto de `test` se **congela antes** de empezar a ajustar prompts.
- Mismas condiciones para todas las estrategias: mismo modelo, mismas
  herramientas, mismo límite de pasos y tokens, esfuerzo de ajuste similar.
- Si tras el test se cambia algo, es un **experimento nuevo** y se documenta
  como tal; no se sobrescriben resultados anteriores.

---

## 5. Preguntas independientes (un solo turno)

- Cada pregunta se resuelve sin contexto de preguntas anteriores.
- Las preguntas del benchmark son autocontenidas.
- Motivo: ejecuciones independientes y repetibles, sin una variable de
  confusión adicional.
- Conversaciones multiturno = posible ampliación fuera del alcance.

---

## 6. Checklist antes de lanzar un experimento

- [ ] Cambios commiteados (`repo_sucio = false`)
- [ ] Tag de git creado si es un experimento final
- [ ] Modelo con versión exacta (no alias genéricos)
- [ ] Parámetros fijados y registrados (temperatura, límites)
- [ ] Conjunto correcto (`dev` / `test`)
- [ ] Nº de repeticiones decidido de antemano
- [ ] Fichero de salida nuevo (no se sobrescribe uno existente)
- [ ] Prueba corta (2–3 preguntas) para comprobar que las trazas se escriben bien
- [ ] Estimación de coste: preguntas × estrategias × repeticiones × modelos
- [ ] Límite de gasto configurado en el proveedor

## 7. Para la memoria

- Anexo con los prompts finales completos y las definiciones de herramientas.
- Indicar commit/tag de los experimentos finales.
- Indicar modelo exacto, fechas de ejecución y parámetros.
- Toda tabla o gráfica debe poder regenerarse con un script a partir de los
  JSONL.
