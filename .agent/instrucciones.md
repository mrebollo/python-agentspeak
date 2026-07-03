# Instrucciones de Desarrollo para Agentes de IA

Este documento contiene las reglas de desarrollo y directrices arquitectónicas para cualquier agente de IA que trabaje en el repositorio de **python-agentspeak**.

---

## 🏛️ Estructura del Proyecto y Módulos Clave

-   **`agentspeak/stdlib.py`**: Contiene la biblioteca de acciones estándar internas del lenguaje (comienzan con un punto, ej. `.print`, `.concat`).
-   **`agentspeak/ext_stdlib.py`**: Acciones estándar extendidas adicionales.
-   **`agentspeak/runtime.py`**: El motor del ciclo de razonamiento BDI. Define clases clave como:
    -   `Agent`: Administra creencias (`beliefs`), reglas (`rules`), planes (`plans`) e intenciones (`intentions`).
    -   `Environment`: Gestiona el tiempo y el ciclo global de ejecución de los agentes.
    -   `Intention`: Representa un hilo de ejecución de metas con su propia pila (`stack`) y variables locales (`scope`).
    -   `Plan`, `Event`, `Instruction`, `Waiter`.
-   **`agentspeak/__init__.py`**: Contiene las definiciones de AST, literales (`Literal`), variables (`Var`), y funciones fundamentales de unificación (`unify`, `unifies`, `grounded`, `freeze`, etc.).

---

## 🛠️ Guía de Codificación para Acciones de la Stdlib

Al agregar o modificar acciones en `agentspeak/stdlib.py`, se deben seguir estas convenciones:

### 1. Registro de Acciones
Las acciones se registran en el objeto global `actions` usando decoradores:
```python
# Para acciones con número fijo de argumentos
@actions.add(".mi_accion", 2)
def _mi_accion(agent, term, intention):
    # Lógica...
    yield
```

También se pueden registrar mediante funciones auxiliares para simplificar firmas:
-   `actions.add_function(functor, arg_specs, f)`: Para funciones puras que unifican su último argumento con el resultado.
-   `actions.add_predicate(functor, arg_specs, f)`: Para condiciones lógicas (deben retornar `True`/`False`).
-   `actions.add_procedure(functor, arg_specs, f)`: Para efectos colaterales sencillos.

### 2. Evaluación y Unificación de Términos
-   **Grounded/Evaluate**: Siempre se deben resolver las variables en los términos antes de operar con ellos usando `agentspeak.grounded(term, scope)` o `agentspeak.evaluate(term, scope)`.
-   **Unificación**: Para asignar valores a variables pasadas como argumento, utiliza siempre la función `agentspeak.unify(left, right, scope, stack)`. Si tiene éxito, se debe ejecutar `yield`.
-   **Efecto de Retorno**: Las funciones decoradas con `@actions.add` son generadores de Python y **deben hacer `yield`** para indicar éxito de ejecución.

### 3. Backtracking (Búsqueda de alternativas)
Si una acción soporta múltiples resultados posibles (como `.substring` o `.member`), se debe manejar la pila de backtracking del intérprete usando un `choicepoint`:
```python
choicepoint = object()
for valor in opciones:
    intention.stack.append(choicepoint)
    if agentspeak.unify(term.args[X], valor, intention.scope, intention.stack):
        yield
    agentspeak.reroll(intention.scope, intention.stack, choicepoint)
```

---

## ⚠️ Reglas Críticas de Comportamiento para Agentes

1.  **NO ejecutar comandos automáticamente**: Bajo ninguna circunstancia se deben ejecutar comandos del sistema (como `pytest` o ejecuciones de prueba) de forma automática o mediante `run_command` sin el consentimiento explícito del usuario en el chat actual. El usuario prefiere validar la estabilidad ejecutando los tests de forma manual.
2.  **Desarrollo incremental**: No intentes realizar cambios masivos que afecten a múltiples fases a la vez. Consulta siempre el archivo `.agent/plan_trabajo.md` y realiza las modificaciones paso a paso por bloques lógicos.
3.  **Preservación de comentarios**: Mantén intactos todos los comentarios y docstrings originales que no tengan relación con el cambio directo.
