# Prompt maestro — POC Toolkit MLData: Workflow automático + Overrides

## 1. Contexto

Estoy trabajando como Machine Learning Engineer / MLOps Engineer en un equipo que mantiene un Toolkit interno de MLData sobre Databricks, MLflow, Python Wheels y GitHub.

El Toolkit busca estandarizar el flujo utilizado por Data Scientists para preparar variables, entrenar modelos, registrar resultados y mantener trazabilidad y gobierno.

Actualmente tengo disponibles dos repositorios:

1. **Repositorio Workshop**
   - Representa el proyecto utilizado por el Data Scientist.
   - Ya permite ejecutar el flujo actual de principio a fin.
   - Será la base para construir el POC.
   - Representa el escenario actual donde el DS acepta la recomendación automática de variables.

2. **Repositorio Toolkit**
   - Contiene el código fuente utilizado para construir el Python Wheel `toolkit-databricks`.
   - Contiene los Entry Points y la lógica interna que realmente ejecuta el Workflow.
   - Este repositorio debe considerarse la fuente principal para entender y modificar el comportamiento del Toolkit.

No quiero empezar modificando código.

Primero necesito entender exactamente cómo funciona actualmente el Toolkit y luego implementar el cambio mínimo necesario para el POC.

---

# 2. Problema que queremos resolver

Actualmente el Toolkit realiza una selección automática de variables.

Por ejemplo:

```text
Variables candidatas:
A B C D E F G X Y Z

        ↓

Feature Selection

        ↓

Recomendación automática:

A B C D E
```

El problema aparece cuando el Data Scientist revisa la recomendación y considera que quiere modificarla.

Por ejemplo:

```text
Automático:
A B C D E

DS quiere:
- quitar B
+ agregar X

Resultado esperado:
A C D E X
```

Actualmente el DS puede terminar saliendo del flujo estandarizado para realizar esta modificación manualmente.

Esto genera pérdida de:

- estandarización
- trazabilidad
- gobierno
- reproducibilidad
- adopción del Toolkit

El objetivo del POC es permitir que el DS pueda adaptar la selección automática de variables **sin abandonar el Toolkit**.

### Problemática adicional que debe quedar explícitamente documentada

El análisis debe considerar que la arquitectura actual genera una **dependencia de la selección automática**.

Actualmente, el flujo conceptual es:

```text
Configuración
      ↓
Preprocesamiento
      ↓
Selección automática de variables
      ↓
Punto de revisión del DS
      ↓
Training
```

Esto significa que el Data Scientist **no define directamente las variables finales desde el inicio**.

Primero debe ejecutar la etapa de selección automática, obtener la recomendación del Toolkit y recién después revisar las variables seleccionadas para decidir si:

* acepta la recomendación;
* elimina alguna variable;
* incorpora alguna variable adicional permitida;
* o realiza otra adaptación controlada.

Por lo tanto, la problemática no debe describirse únicamente como:

> "El DS necesita modificar variables después de la selección."

Debe analizarse también como:

> **"El DS depende de la selección automática como paso previo obligatorio para poder llegar a su conjunto final de variables."**

Esta dependencia es relevante porque condiciona el diseño del POC y de la alternativa **Workflow único + Overrides**.

### Punto de decisión que debe investigarse

El análisis debe determinar cómo representar técnicamente el momento en que el DS revisa la recomendación automática.

Conceptualmente:

```text
                  Selección automática
                         ↓
                  Variables A B C D E
                         ↓
                 ┌───────┴───────┐
                 │               │
              Acepta          Adapta
                 │               │
                 │          - B + X
                 │               │
                 │          A C D E X
                 │               │
                 └───────┬───────┘
                         ↓
                      Training
                         ↓
                     Best Model
                         ↓
                       MLflow
```

Sin embargo, **no asumir que esta pausa debe implementarse como un Workflow de Databricks que permanece ejecutándose esperando interacción humana**.

Durante Discovery se debe determinar:

1. Qué componente genera la selección automática.
2. Dónde queda persistida esa recomendación.
3. Qué componente consume actualmente las variables seleccionadas.
4. Dónde existe actualmente el límite entre Feature Selection y Training.
5. Si el Workflow puede representar el punto de decisión mediante Tasks, Steps, ejecuciones separadas u otro mecanismo.
6. Cómo permitir el Override sin duplicar ni alterar la lógica existente de Training.
7. Cómo mantener la trazabilidad entre:

   * selección automática;
   * decisión del DS;
   * variables excluidas;
   * variables incluidas;
   * conjunto final utilizado para Training;
   * Best Model;
   * MLflow.

### Regla funcional del POC

El POC debe demostrar que:

> **La selección automática continúa siendo el comportamiento por defecto, pero deja de ser una restricción rígida para llegar al Training.**

Por tanto:

```text
SIN OVERRIDE
Selección automática
        ↓
Training
        ↓
Best Model
```

debe continuar funcionando exactamente como hoy.

Mientras que:

```text
CON OVERRIDE
Selección automática
        ↓
Revisión / decisión del DS
        ↓
Override controlado
        ↓
Selección final
        ↓
Training
        ↓
Best Model
```

debe permitir adaptar las variables sin abandonar el Toolkit.

### Escenario adicional que debe comprobarse

El POC debe evaluar también si es posible comparar:

```text
Selección automática
A B C D E
    ↓
Training
    ↓
Best Model
```

contra:

```text
Selección adaptada
A C D E X
    ↓
Training
    ↓
Best Model
```

manteniendo la misma lógica existente de entrenamiento, evaluación y selección del Best Model.

**Importante:** el POC no debe crear una nueva lógica de Best Model. Debe reutilizar la existente.

La premisa principal es:

> **"La recomendación automática nunca se pierde; el DS puede aceptarla o adaptarla mediante excepciones controladas."**

Y la pregunta técnica que debe responder Discovery es:

> **¿Cómo permitimos que el DS revise y adapte la recomendación automática sin obligarlo a salir del Toolkit y sin romper el flujo estandarizado de Training, Best Model, MLflow y entregables?**


---

# 3. Comportamiento actual que NO debemos romper

Es importante aclarar que el Toolkit actual **sí realiza selección del Best Model**.

La configuración permite, entre otras cosas:

- definir el archivo/configuración de Training
- definir los algoritmos que se desean contemplar
- ejecutar entrenamiento con esos algoritmos
- evaluar sus resultados
- determinar automáticamente cuál es el Best Model
- generar/registrar los entregables correspondientes
- utilizar MLflow para trazabilidad

Por ejemplo, si el DS configura:

```text
Algoritmo A
Algoritmo B
Algoritmo C
```

el Toolkit actualmente realiza algo conceptualmente similar a:

```text
Training
   │
   ├── Algoritmo A
   ├── Algoritmo B
   └── Algoritmo C
          ↓
      evaluación
          ↓
      Best Model
          ↓
    MLflow / entregables
```

Este comportamiento debe mantenerse.

**El POC NO busca modificar la lógica existente de selección del Best Model.**

---

# 4. Objetivo funcional del POC

El objetivo es permitir dos posibilidades:

### Camino 1 — Automático

El DS acepta la recomendación automática.

```text
Feature Selection
       ↓
A B C D E
       ↓
Training
       ↓
Algoritmos configurados
       ↓
Evaluación
       ↓
Best Model
       ↓
MLflow / entregables
```

Este camino debe conservar el comportamiento actual.

---

### Camino 2 — Adaptado mediante Overrides

El DS revisa la recomendación y quiere modificarla.

Ejemplo:

```text
Automático:
A B C D E

Override:
exclude → B
include → X

Selección final:
A C D E X
```

Luego el Toolkit debería utilizar esa selección adaptada para ejecutar el mismo proceso de Training:

```text
A C D E X
     ↓
Training
     ↓
Algoritmos configurados
     ↓
Evaluación
     ↓
Best Model
     ↓
MLflow / entregables
```

La idea es que el Toolkit permita experimentar con una selección adaptada **sin que el DS tenga que salir del flujo estándar**.

---

# 5. Importante: cómo representar técnicamente los dos caminos

No quiero imponer de antemano si los dos caminos deben implementarse como:

- dos Tasks de Databricks
- dos Steps dentro de una Task
- dos ejecuciones del mismo componente de Training
- dos Runs de MLflow
- otra solución técnicamente más adecuada

Esto debe determinarse después de analizar el código real.

La pregunta que debemos responder es:

> **¿Cuál es la forma más sencilla y segura de reutilizar el Training actual para ejecutar tanto la selección automática como una selección adaptada mediante Overrides?**

La arquitectura final debe minimizar cambios sobre el Toolkit existente.

---

# 6. Escenario que quiero evaluar especialmente

Me interesa evaluar si dentro del mismo contexto de Workflow podemos conservar el camino automático y, cuando exista un Override, ejecutar también el camino adaptado.

Conceptualmente:

```text
                 Feature Selection
                        │
                        ▼
                   A B C D E
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
        CAMINO AUTOMÁTICO    CAMINO ADAPTADO
              │                   │
        A B C D E             Override
                                  │
                             - B + X
                                  │
                             A C D E X
              │                   │
              ▼                   ▼
           Training             Training
              │                   │
              ▼                   ▼
          Best Model           Best Model
              │                   │
              └─────────┬─────────┘
                        ▼
                      MLflow
```

Sin embargo, esto es **un objetivo funcional**, no una decisión técnica definitiva.

Quiero que el análisis determine si esta representación realmente tiene sentido con la arquitectura actual.

---

# 7. No duplicar innecesariamente el entrenamiento

Cuando no existen Overrides, debe mantenerse el comportamiento actual.

Es decir:

```text
Feature Selection
       ↓
Training automático
       ↓
Best Model
```

No deberíamos ejecutar dos entrenamientos idénticos solamente por implementar el POC.

Cuando existen Overrides, evaluar si tiene sentido ejecutar:

```text
Training automático
+
Training adaptado
```

para poder comparar ambas configuraciones.

La decisión debe considerar:

- costo computacional
- complejidad
- impacto en el Workflow
- reutilización del código existente
- trazabilidad
- experiencia del Data Scientist

---

# 8. MLflow y trazabilidad

Quiero entender primero cómo funciona actualmente MLflow.

Es posible que cada Task o proceso genere un Experiment/Run y sus respectivos entregables, pero esto debe confirmarse revisando el código.

Necesito identificar exactamente:

- cómo se crea el Experiment
- cómo se crean los Runs
- qué parámetros se registran
- qué métricas se registran
- qué artefactos se generan
- cómo se registra el modelo
- cómo se determina el Best Model
- cómo se relacionan los Runs con los Tasks
- cómo se relacionan los entregables con cada ejecución

Para el POC quiero poder reconstruir algo similar a:

```text
Recomendación automática:
A B C D E

Override:
exclude = B
include = X

Selección final:
A C D E X
```

Idealmente, estos elementos deberían quedar trazables en MLflow aprovechando los mecanismos existentes.

No quiero sobre-diseñar el logging si el Toolkit ya tiene mecanismos adecuados.

---

# 9. Escenarios funcionales que debemos probar

El POC debe considerar como mínimo:

### Caso A — Sin Override

```text
Automático:
A B C D E

Final:
A B C D E
```

Debe comportarse exactamente como hoy.

---

### Caso B — Excluir variable

```text
Automático:
A B C D E

Override:
exclude = B

Final:
A C D E
```

---

### Caso C — Incluir variable

```text
Automático:
A B C D E

Override:
include = X

Final:
A B C D E X
```

---

### Caso D — Excluir e incluir

```text
Automático:
A B C D E

Override:
exclude = B
include = X

Final:
A C D E X
```

---

### Caso E — Variable no autorizada

Ejemplo:

```text
include = VARIABLE_NO_PERMITIDA
```

Debe existir una validación controlada.

No debería permitirse introducir arbitrariamente cualquier columna o variable fuera del universo/candidatos autorizados.

---

### Caso F — Conflicto

Ejemplo:

```text
include = X
exclude = X
```

Debe generar un error controlado y comprensible.

---

# 10. Regla conceptual del Override

El Override debe representar una **modificación controlada de la recomendación automática**, no una vía para que el DS pueda ejecutar código arbitrario.

Conceptualmente:

```text
FINAL =
(AUTOMATIC_SELECTION - FORCE_EXCLUDE)
+
FORCE_INCLUDE
```

Pero **no asumir esta implementación ni este nombre de configuración**.

El contrato real debe definirse después de revisar cómo funciona actualmente el Toolkit.

---

# 11. Gobernanza

El POC debe conservar el gobierno existente.

Por ejemplo:

- las variables deben pertenecer al universo/candidatos autorizados
- no permitir fuentes arbitrarias
- no permitir código Python de producción escrito por el DS
- mantener los algoritmos permitidos definidos por configuración
- mantener la trazabilidad
- registrar las excepciones
- mantener la reproducibilidad

El concepto que buscamos es:

> **Automatizado por defecto, configurable por excepción.**

---

# 12. Análisis obligatorio antes de modificar código

Primero analizar ambos repositorios.

NO comenzar implementando.

Necesito construir el mapa real:

```text
Data Scientist
      ↓
Config
      ↓
Generator Notebook
      ↓
workflow.yaml
      ↓
Databricks Workflow
      ↓
Tasks
      ↓
Python Wheel
      ↓
Entry Points
      ↓
Toolkit source code
      ↓
Preprocessing
      ↓
Feature Selection
      ↓
Training
      ↓
MLflow
      ↓
Best Model
      ↓
Entregables
```

Para cada etapa necesito identificar el código real.

---

# 13. Análisis del Workshop Repository

Identificar:

1. archivo de configuración utilizado por el DS
2. parámetros configurables
3. población/universo
4. tablas candidatas
5. algoritmos
6. configuración de Feature Selection
7. Generator Notebook
8. cómo lee la configuración
9. cómo transforma la configuración
10. cómo genera el YAML
11. cómo define las Tasks
12. cómo define dependencias
13. cómo define parámetros
14. qué Entry Points utiliza
15. qué versión/package del Wheel utiliza

Explicar cómo cada pieza se relaciona con el Workflow.

---

# 14. Análisis del Toolkit Repository

Buscar los Entry Points utilizados realmente por el Workshop.

Para cada Entry Point encontrado documentar:

```text
ARCHIVO:
ruta real

COMPONENTE:
clase / función / Entry Point

QUÉ HACE HOY:
explicación sencilla

INPUT:
qué recibe

OUTPUT:
qué produce

RELACIÓN CON EL WORKFLOW:
cómo participa

RELACIÓN CON FEATURE SELECTION:
por qué es relevante

RELACIÓN CON TRAINING:
por qué es relevante

RELACIÓN CON MLFLOW:
qué registra

CAMBIO PROPUESTO:
si aplica

POR QUÉ:
justificación técnica

IMPACTO:
bajo / medio / alto

RIESGO:
qué podría romperse

PRUEBA:
cómo comprobarlo
```

No inventar nombres de archivos, clases, funciones ni rutas.

Si algo no puede determinarse con la información disponible, indicarlo explícitamente.

---

# 15. Pregunta técnica principal

Necesito identificar con precisión:

### 1. ¿Qué produce Feature Selection?

Por ejemplo:

- lista de variables
- DataFrame
- tabla Delta
- archivo
- YAML
- JSON
- manifest
- metadata
- otro objeto

### 2. ¿Dónde se guarda?

### 3. ¿Quién lo consume?

### 4. ¿Cómo Training sabe actualmente qué variables utilizar?

### 5. ¿Dónde podemos introducir el Override con el menor impacto?

### 6. ¿Cómo se puede reutilizar el Training existente para el camino adaptado?

### 7. ¿Dónde se determina actualmente el Best Model?

### 8. ¿Cómo se relacionan Experiments, Runs, Tasks y entregables?

Estas respuestas son prioritarias antes de escribir código.

---

# 16. Diseño esperado del POC

Una vez entendido el código, proponer el diseño mínimo.

Debe responder:

```text
¿Dónde se define el Override?
        ↓
¿Dónde se valida?
        ↓
¿Cómo se combina con la selección automática?
        ↓
¿Cómo llega la selección final al Training?
        ↓
¿Cómo se ejecuta el Training?
        ↓
¿Cómo se determina el Best Model?
        ↓
¿Cómo se registra en MLflow?
```

No modificar componentes que no sean necesarios.

---

# 17. Reutilización

Preferir:

```text
Código actual
     ↓
reutilización
     ↓
nuevo comportamiento
```

en lugar de:

```text
Código actual
     ↓
duplicación completa
     ↓
nuevo código paralelo
```

Si el Training existente puede recibir una selección de variables parametrizada, aprovecharlo.

Si actualmente existe una función interna que resuelve las variables, evaluar si puede extenderse.

No crear nuevas abstracciones sin necesidad.

---

# 18. Construcción del POC

Después del análisis:

1. modificar Toolkit
2. implementar Override
3. implementar validaciones
4. mantener camino automático
5. reutilizar Training existente
6. mantener selección del Best Model
7. incorporar trazabilidad necesaria
8. generar nueva versión del Wheel
9. construir el Wheel
10. integrar la nueva versión con Workshop
11. ejecutar pruebas

No modificar primero el Workshop para ocultar problemas del Toolkit.

El Toolkit debe ser el lugar principal donde viva la lógica de negocio del Override.

---

# 19. Pruebas técnicas

Validar como mínimo:

```text
A. Sin Override
B. Solo exclude
C. Solo include
D. Include + exclude
E. Variable no autorizada
F. Include/exclude conflictivo
```

Además comprobar:

- no se rompe el Workflow actual
- el Training sigue ejecutándose
- los algoritmos configurados siguen siendo respetados
- el Best Model sigue determinándose correctamente
- los entregables siguen generándose
- MLflow mantiene trazabilidad
- la selección automática original no se pierde
- la selección final utilizada para Training queda identificada

---

# 20. Comparación automático vs adaptado

Si la arquitectura lo permite, realizar una prueba donde podamos observar:

```text
RUN AUTOMÁTICO
Features:
A B C D E

Algoritmos:
A B C

Best Model:
...

Métricas:
...
```

y:

```text
RUN ADAPTADO
Automatic Features:
A B C D E

Override:
-B
+X

Final Features:
A C D E X

Algoritmos:
A B C

Best Model:
...

Métricas:
...
```

El objetivo es poder observar las diferencias de forma trazable.

**No se busca modificar ni reemplazar la lógica existente de selección del Best Model.**

---

# 21. Importante sobre el Best Model

El Toolkit actual ya determina el Best Model.

Por lo tanto:

```text
Selección automática
        ↓
Training
        ↓
Best Model
```

y:

```text
Selección adaptada
        ↓
Training
        ↓
Best Model
```

deben utilizar la misma lógica existente.

El POC no debe convertirse en un nuevo componente de selección de modelos.

El objetivo es:

> **Permitir ejecutar el mismo proceso actual de Training y selección de Best Model utilizando tanto la selección automática como una selección adaptada mediante Overrides.**

---

# 22. Criterio de éxito

El POC será exitoso si conseguimos demostrar:

> **“El Data Scientist puede aceptar la recomendación automática o modificarla de forma controlada, continuar dentro del Toolkit y obtener el mismo proceso estandarizado de entrenamiento, selección de Best Model, trazabilidad y entregables.”**

La frase conceptual del POC es:

> **“Flexibilidad para experimentar, sin perder gobierno, trazabilidad ni estandarización.”**

Y:

> **“Automatizado por defecto, configurable por excepción.”**

---

# 23. Lo que NO debemos hacer todavía

No implementar todavía:

- Feature Store
- Feature Registry completo
- nueva plataforma de Features
- UI nueva
- sistema completo de experimentación
- selección automática adicional de modelos
- champion/challenger avanzado
- refactor masivo del Toolkit
- cambios innecesarios al Training
- arquitectura B completa de desacoplamiento
- arquitectura C completa de plataforma de Features

El objetivo es un **POC mínimo sobre el Toolkit actual**.

---

# 24. Resultado esperado de tu análisis

Quiero que la respuesta inicial esté organizada así:

## Fase 1 — Entendimiento actual

Explicar cómo funciona realmente el Toolkit.

## Fase 2 — Mapa de ejecución

Mostrar:

```text
Config
→ Generator
→ YAML
→ Workflow
→ Task
→ Wheel
→ Entry Point
→ Feature Selection
→ Training
→ MLflow
→ Best Model
→ Entregables
```

## Fase 3 — Punto exacto de intervención

Explicar:

- dónde introducir Override
- por qué
- qué componente se modifica
- qué componentes no se deberían tocar

## Fase 4 — Diseño propuesto

Mostrar el flujo automático y el adaptado.

## Fase 5 — Impacto

Indicar:

- archivos afectados
- componentes afectados
- riesgo
- complejidad
- compatibilidad con comportamiento actual

## Fase 6 — Implementación

Solo después de completar las fases anteriores.

---

# 25. Regla principal

**No asumir cómo funciona el Toolkit.**

Quiero que las conclusiones estén basadas en el código real de los dos repositorios.

Si existe una diferencia entre lo que yo creo que hace el Toolkit y lo que realmente hace el código, señalarla claramente.

La prioridad es:

**Entender → Mapear → Diseñar → Implementar → Probar.**

No:

**Asumir → Codificar → Corregir.**

Sí. Yo definiría un **formato de salida obligatorio para cada fase**, para que la cuenta corporativa no mezcle descubrimiento con diseño ni empiece a implementar antes de tiempo.

Puedes añadir esta sección al final del prompt:

# 26. Formato de salida obligatorio por fase

El trabajo debe producir un entregable claramente separado para cada fase.

**No mezclar descubrimiento, diseño e implementación en una misma respuesta.**

Cada fase debe terminar con una sección de **Conclusiones y Gate de aprobación**.

---

# FASE 1 — DESCUBRIMIENTO

## Objetivo de la salida

La salida debe permitir entender **cómo funciona hoy el Toolkit**, sin proponer todavía cambios de implementación.

### Formato obligatorio

## 1. Resumen ejecutivo

Máximo 10-15 bullets.

Responder:

* ¿Cómo funciona actualmente el flujo?
* ¿Cuáles son los componentes principales?
* ¿Dónde ocurre Feature Selection?
* ¿Dónde ocurre Training?
* ¿Dónde se determina Best Model?
* ¿Cómo interviene MLflow?
* ¿Cuál es el punto actual donde se conectan Feature Selection y Training?

No proponer todavía la solución.

---

## 2. Arquitectura actual

Mostrar un diagrama basado en el código real:

```text
Data Scientist
      ↓
Config
      ↓
Generator
      ↓
workflow.yaml
      ↓
Databricks Workflow
      ↓
Task
      ↓
Python Wheel
      ↓
Entry Point
      ↓
Preprocessing
      ↓
Feature Selection
      ↓
[resultado]
      ↓
Training
      ↓
Algoritmos
      ↓
Evaluación
      ↓
Best Model
      ↓
MLflow
      ↓
Entregables
```

Si el flujo real es diferente, corregir el diagrama.

---

## 3. Tabla de componentes

Utilizar:

| # | Repositorio | Archivo | Componente | Responsabilidad | Input | Output | Siguiente componente |
| - | ----------- | ------- | ---------- | --------------- | ----- | ------ | -------------------- |

Solo incluir información confirmada por el código.

---

## 4. Flujo Workshop

Explicar:

```text
Config
→ Generator
→ YAML
→ Workflow
→ Tasks
```

Para cada paso indicar:

* archivo
* función/clase
* qué hace
* qué recibe
* qué produce

---

## 5. Flujo Toolkit

Explicar:

```text
Task
→ Wheel
→ Entry Point
→ función/clase
→ librerías
→ resultado
```

Identificar los Entry Points reales utilizados por el Workshop.

---

## 6. Feature Selection → Training

Esta sección es obligatoria y debe ser muy concreta.

Responder:

> **¿Qué produce exactamente Feature Selection y cómo llega ese resultado al Training?**

Mostrar:

```text
Feature Selection
      ↓
[formato real del resultado]
      ↓
[persistencia / almacenamiento]
      ↓
[componente consumidor]
      ↓
Training
```

---

## 7. Training → Best Model

Explicar:

* cómo se configura Training
* qué algoritmos recibe
* cómo se ejecutan
* cómo se evalúan
* dónde se determina Best Model
* qué resultado se genera

Mostrar el flujo real.

---

## 8. MLflow / Experiments / Runs / Entregables

Explicar exactamente:

```text
Workflow Task
      ↓
?
      ↓
MLflow Experiment
      ↓
Runs
      ↓
Metrics / Parameters / Artifacts
      ↓
Best Model
      ↓
Entregables
```

Si la relación real es diferente, mostrarla.

No asumir que:

> Task = Experiment

o:

> Task = Run

hasta comprobarlo en el código.

---

## 9. Fichas de componentes relevantes

Para cada componente importante:

```text
ARCHIVO:
ruta real

COMPONENTE:
nombre real

QUÉ HACE HOY:
...

INPUT:
...

OUTPUT:
...

DEPENDENCIAS:
...

RELACIÓN CON WORKFLOW:
...

RELACIÓN CON FEATURE SELECTION:
...

RELACIÓN CON TRAINING:
...

RELACIÓN CON MLFLOW:
...

RELACIÓN CON BEST MODEL:
...

IMPACTO POTENCIAL:
bajo / medio / alto

RIESGO:
...

EVIDENCIA:
archivo / función / línea o referencia disponible
```

---

## 10. Hallazgos

Separar explícitamente:

### Confirmado por código

* ...

### Inferido pero no confirmado

* ...

### Información faltante

* ...

Esto es importante para no convertir hipótesis en hechos.

---

## 11. Puntos de extensión candidatos

Identificar posibles puntos donde podría introducirse el Override.

Para cada uno:

| Punto | Archivo | Componente | Ventaja | Riesgo | Impacto |
| ----- | ------- | ---------- | ------- | ------ | ------- |

**No seleccionar todavía una solución definitiva.**

---

## 12. Conclusión de Discovery

Cerrar con:

### Lo que entendemos

...

### Lo que todavía necesitamos confirmar

...

### Punto recomendado para analizar en Diseño

...

### Decisión de Gate

```text
DISCOVERY STATUS:
[ ] Incompleto
[ ] Completo — listo para Diseño
```

Si está incompleto, indicar exactamente qué falta.

---

# FASE 2 — DISEÑO

## Objetivo de la salida

Convertir los hallazgos de Discovery en un diseño técnico mínimo.

**No escribir código todavía.**

---

## 1. Resumen de la solución propuesta

Máximo 10 bullets.

Responder:

* ¿Dónde estará el Override?
* ¿Cómo se calcula la selección final?
* ¿Cómo llega al Training?
* ¿Cómo se conserva el camino automático?
* ¿Cómo se ejecutaría el camino adaptado?
* ¿Cómo se mantiene Best Model?
* ¿Cómo se registra en MLflow?

---

## 2. Arquitectura propuesta

Mostrar claramente:

```text
                    Feature Selection
                           │
                           ▼
                    Automatic Features
                           │
                 ┌─────────┴─────────┐
                 │                   │
                 ▼                   ▼
           AUTOMÁTICO            OVERRIDE
                 │                   │
                 │              Final Features
                 │                   │
                 ▼                   ▼
              Training            Training
                 │                   │
                 ▼                   ▼
             Best Model          Best Model
                 │                   │
                 └─────────┬─────────┘
                           ▼
                         MLflow
```

El diagrama debe reflejar la solución técnicamente propuesta después de Discovery.

---

## 3. Decisión sobre Tasks / Steps / Runs

Responder explícitamente:

> **¿Cómo se implementarán técnicamente los dos caminos?**

Evaluar:

* dos Tasks
* dos Steps
* dos ejecuciones del Training existente
* parametrización del Training
* otro mecanismo

Explicar:

### Opción seleccionada

...

### Alternativas consideradas

...

### Por qué se selecciona

...

No crear una segunda implementación paralela si el código existente puede reutilizarse.

---

## 4. Contrato del Override

Definir:

### Input

```yaml
...
```

### Validaciones

* ...
* ...
* ...

### Transformación

```text
Automatic Selection
        ↓
Override
        ↓
Final Selection
```

### Output

```text
Final Features
```

Los nombres definitivos deben basarse en el código real.

---

## 5. Impacto sobre componentes

Tabla obligatoria:

| Archivo | Componente | Cambio | Motivo | Impacto | Riesgo |
| ------- | ---------- | ------ | ------ | ------- | ------ |

Separar:

### Se modifica

...

### Se reutiliza sin cambios

...

### No se debe tocar

...

---

## 6. Training y Best Model

Explicar explícitamente:

```text
Automatic Features
        ↓
Training actual
        ↓
Best Model actual
```

y:

```text
Final Features
        ↓
Training actual
        ↓
Best Model actual
```

Demostrar que la lógica de Best Model se reutiliza y no se reemplaza.

---

## 7. MLflow y trazabilidad

Definir cómo distinguir:

```text
AUTOMATIC
ADAPTED
```

y cómo registrar:

* automatic_features
* force_include
* force_exclude
* final_features
* algoritmos
* métricas
* Best Model

Solo incorporar nuevos campos si realmente son necesarios.

---

## 8. Compatibilidad hacia atrás

Explicar qué ocurre cuando:

```text
Override = vacío / inexistente
```

El resultado esperado debe ser:

```text
Comportamiento actual
```

---

## 9. Matriz de escenarios

| Caso | Automatic | Include | Exclude | Final       | Resultado esperado     |
| ---- | --------- | ------- | ------- | ----------- | ---------------------- |
| A    | A B C D E | —       | —       | A B C D E   | Actual                 |
| B    | A B C D E | —       | B       | A C D E     | Adaptado               |
| C    | A B C D E | X       | —       | A B C D E X | Adaptado               |
| D    | A B C D E | X       | B       | A C D E X   | Adaptado               |
| E    | A B C D E | Z       | —       | Error       | Variable no autorizada |
| F    | A B C D E | X       | X       | Error       | Conflicto              |

---

## 10. Plan de implementación

Definir pasos exactos:

```text
1. Modificar componente X
2. Agregar validación Y
3. Reutilizar componente Z
4. Ajustar configuración
5. Construir Wheel
6. Integrar Workshop
7. Ejecutar pruebas
```

No implementar todavía.

---

## 11. Criterios de aceptación

Definir condiciones verificables.

Ejemplo:

```text
[ ] Sin Override mantiene comportamiento actual
[ ] Include funciona
[ ] Exclude funciona
[ ] Include + Exclude funciona
[ ] Variables no autorizadas son rechazadas
[ ] Conflictos son rechazados
[ ] Training funciona
[ ] Best Model funciona
[ ] MLflow registra trazabilidad
[ ] Entregables siguen generándose
```

---

## 12. Conclusión de Diseño

Cerrar con:

### Solución recomendada

...

### Archivos que se modificarán

...

### Archivos que permanecerán intactos

...

### Riesgos

...

### Decisión de Gate

```text
DESIGN STATUS:
[ ] Incompleto
[ ] Completo — listo para Implementación
```

No implementar si el estado no es:

```text
Completo — listo para Implementación
```

---

# FASE 3 — IMPLEMENTACIÓN

## Objetivo de la salida

Implementar exactamente el diseño aprobado.

No introducir cambios arquitectónicos adicionales durante la implementación salvo que aparezca una incompatibilidad real con el código.

---

## 1. Resumen de cambios realizados

| # | Archivo | Componente | Cambio | Motivo |
| - | ------- | ---------- | ------ | ------ |

---

## 2. Cambios por componente

Para cada cambio:

```text
ARCHIVO:
...

COMPONENTE:
...

CAMBIO:
...

ANTES:
...

DESPUÉS:
...

POR QUÉ:
...
```

---

## 3. Flujo implementado

Mostrar el flujo real después del cambio:

```text
Config
   ↓
Feature Selection
   ↓
Automatic Features
   ↓
Override
   ↓
Final Features
   ↓
Training
   ↓
Algorithms
   ↓
Best Model
   ↓
MLflow
```

Si existen dos ejecuciones/caminos:

```text
Automatic → Training → Best Model
                  +
Adapted   → Training → Best Model
```

Mostrar cómo quedó realmente implementado.

---

## 4. Validaciones implementadas

Documentar:

| Validación             | Entrada | Resultado esperado | Resultado real |
| ---------------------- | ------- | ------------------ | -------------- |
| Sin Override           | ...     | ...                | ...            |
| Include                | ...     | ...                | ...            |
| Exclude                | ...     | ...                | ...            |
| Include + Exclude      | ...     | ...                | ...            |
| Variable no autorizada | ...     | Error              | ...            |
| Conflicto              | ...     | Error              | ...            |

---

## 5. Build e integración

Documentar:

* versión del Wheel
* resultado del build
* integración con Workshop
* configuración utilizada
* Workflow ejecutado
* resultado de ejecución

No asumir éxito: indicar resultados reales.

---

## 6. Resultados de pruebas

Utilizar:

| Caso | Resultado | Evidencia | Observación |
| ---- | --------- | --------- | ----------- |
| A    | PASS/FAIL | ...       | ...         |
| B    | PASS/FAIL | ...       | ...         |
| C    | PASS/FAIL | ...       | ...         |
| D    | PASS/FAIL | ...       | ...         |
| E    | PASS/FAIL | ...       | ...         |
| F    | PASS/FAIL | ...       | ...         |

---

## 7. Validación de MLflow

Demostrar:

```text
Automatic Features
Override
Final Features
Algorithms
Metrics
Best Model
Artifacts
```

Indicar dónde se verificó cada elemento.

---

## 8. Comparación automática vs adaptada

Mostrar, cuando corresponda:

| Elemento           | Automático | Adaptado |
| ------------------ | ---------- | -------- |
| Automatic Features | ...        | ...      |
| Override           | —          | ...      |
| Final Features     | ...        | ...      |
| Algorithms         | ...        | ...      |
| Best Model         | ...        | ...      |
| Métricas           | ...        | ...      |
| Run / Experiment   | ...        | ...      |

---

## 9. Compatibilidad

Confirmar:

```text
[ ] El escenario sin Override sigue funcionando
[ ] El camino automático no fue alterado
[ ] Best Model mantiene la lógica existente
[ ] Los algoritmos configurados se mantienen
[ ] Los entregables existentes se mantienen
[ ] MLflow mantiene trazabilidad
```

---

## 10. Problemas encontrados

Separar:

### Problemas resueltos

...

### Problemas pendientes

...

### Cambios realizados respecto al diseño

...

Si la implementación obligó a modificar el diseño aprobado, explicar:

* qué cambió
* por qué
* impacto
* decisión tomada



---

## 11. Resultado final

Cerrar con:

### Qué se consiguió

...

### Qué quedó demostrado

...

### Qué no forma parte del POC

...

### Próximos pasos sugeridos

...

---

# REGLA DE SALIDA GENERAL

Cada fase debe terminar con un **Gate**.

```text
┌─────────────────────┐
│   DISCOVERY         │
│   entender          │
└──────────┬──────────┘
           │
           ▼
      GATE 1
           │
           ▼
┌─────────────────────┐
│   DESIGN            │
│   decidir            │
└──────────┬──────────┘
           │
           ▼
      GATE 2
           │
           ▼
┌─────────────────────┐
│   IMPLEMENTATION    │
│   construir/probar  │
└──────────┬──────────┘
           │
           ▼
      GATE 3
```

### Gate 1 — Discovery

No debe existir código modificado.

Debe existir suficiente evidencia para explicar cómo funciona actualmente el Toolkit.

### Gate 2 — Design

Debe existir una solución técnica definida y justificada.

No debe existir implementación todavía.

### Gate 3 — Implementation

Debe existir código modificado, Wheel construido, integración con Workshop y pruebas ejecutadas.

---

## Regla fundamental

**No mezclar fases.**

Si estoy en Discovery:

> Entender y documentar.

Si estoy en Design:

> Decidir y justificar.

Si estoy en Implementation:

> Construir y probar.

La secuencia obligatoria es:

**DESCUBRIR → APROBAR → DISEÑAR → APROBAR → IMPLEMENTAR → PROBAR.**
