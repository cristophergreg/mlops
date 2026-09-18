# Contenido de presentación — Toolkit MLData

## Slide 1 — Problemática de adopción del Toolkit

### Título
**Toolkit MLData — Evolución para mejorar la adopción**

### Mensaje central
> El Toolkit automatiza y estandariza el desarrollo de modelos, pero actualmente su rigidez dificulta la experimentación del Data Scientist.

### Visual
```text
                 TOOLKIT ACTUAL

 Configuración
      │
      ▼
 Preprocesamiento
      │
      ▼
 Selección automática
      │
      ▼
 Entrenamiento
      │
      ▼
    MLflow

       ⚠️ PUNTO DE FRICCIÓN

 El DS descubre después de la selección
 que necesita ajustar sus variables.

      ↓

 Notebook externo
      ↓
 Pérdida de estandarización
 Pérdida de trazabilidad
 Pérdida de adopción
```

### Mensaje a recordar
No necesariamente existe un problema de capacidad técnica, sino de **flexibilidad del flujo**.

---

## Slide 2 — ¿Qué necesitamos resolver?

### Título
**El desafío: flexibilidad sin perder gobierno**

| Necesidad del DS | Necesidad del COE |
|---|---|
| Experimentar con variables | Mantener trazabilidad |
| Excluir variables | Controlar el proceso |
| Incorporar variables aprobadas | Reproducibilidad |
| Iterar rápidamente | Algoritmos gobernados |
| Probar diferentes modelos | MLflow estandarizado |
| No reprocesar todo | Auditoría |

### Visual central
```text
             ┌──────────────────────┐
             │       TOOLKIT        │
             │                      │
             │ Flexibilidad +       │
             │ Gobernanza           │
             └──────────────────────┘
```

### Frase
> **El objetivo no es liberar el Toolkit, sino hacerlo flexible dentro de límites controlados.**

---

## Slide 3 — Opciones de arquitectura

### Título
**Alternativas evaluadas**

### Opción A — Workflow único + Overrides

```text
Config
  ↓
Preprocessing
  ↓
Selection
  ↓
Overrides
  ↓
Training
```

**A = Quick Win**

### Opción B — Desacoplar Features y Training

```text
             Config
               │
        ┌──────┴──────┐
        ▼             ▼
 Feature Workflow   Training Workflow
        │             ▲
        ▼             │
 Dataset + Manifest ──┘
```

**B = Evolución recomendada**

### Opción C — Evolución hacia plataforma de Features

```text
Feature Registry
       ↓
Feature Store
       ↓
Feature Pipeline
       ↓
Training
       ↓
MLflow
```

**C = Visión futura**

### Nota
No presentar C como “mejor”; representa una evolución de mayor alcance y esfuerzo.

---

## Slide 4 — Pros y contras

### Título
**Comparación de alternativas**

| Alternativa | Pros | Contras |
|---|---|---|
| **A. Workflow + Overrides** | Implementación rápida; bajo impacto; mantiene arquitectura actual | El entrenamiento sigue acoplado; menor capacidad de experimentación |
| **B. Features + Training** | Permite iterar entrenamiento; evita reprocesamiento; dataset versionado; mejor separación de responsabilidades | Mayor complejidad inicial; requiere gestionar versiones de datasets/manifests |
| **C. Plataforma de Features** | Mayor reutilización; gobierno centralizado; potencial para múltiples proyectos | Mayor inversión; más componentes; mayor esfuerzo de adopción y gobierno |

### Visual sugerido
```text
                  ESFUERZO
                     ▲
                     │             C
                     │
                     │       B
                     │
                     │  A
                     └──────────────────►
                         FLEXIBILIDAD
```

---

## Slide 5 — Propuesta recomendada

### Título
**Propuesta: Automated by Default, Configurable by Exception**

### Arquitectura

```text
                 CONFIGURACIÓN
                       │
                       ▼
              ┌────────────────┐
              │ PREPROCESSING  │
              └───────┬────────┘
                      ▼
             SELECCIÓN AUTOMÁTICA
                      │
                      ▼
              ┌───────────────┐
              │ DS REVISA     │
              │               │
              │ + VAR_A       │
              │ - VAR_B       │
              └───────┬───────┘
                      ▼
              FEATURE MANIFEST
                      │
                      ▼
               DATASET VERSIONADO
                      │
                      ▼
                TRAINING
                      │
                      ▼
                   MLflow
```

### El Toolkit controla
- Fuentes aprobadas
- Transformaciones
- Selección
- Infraestructura
- Algoritmos
- MLflow
- Auditoría

### El DS puede configurar
- Universo
- Tablas candidatas
- Variables a incluir
- Variables a excluir
- Parámetros permitidos
- Experimentos de entrenamiento

### Mensaje
La automatización sigue siendo el camino principal; las excepciones son explícitas, controladas y auditables.

---

## Slide 6 — Gobernanza y trazabilidad

### Título
**Cada excepción queda registrada**

### Visual

```text
                 MODEL RUN
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   Git Commit    Dataset V003   Toolkit 5.8
       │             │             │
       └─────────────┼─────────────┘
                     ▼
              Feature Manifest
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    Automáticas   Incluidas    Excluidas
       47             2            3
                     │
                     ▼
                  MLflow
```

### Ejemplo
```text
VAR_A → Seleccionada automáticamente
VAR_B → Seleccionada → EXCLUIDA
VAR_X → No seleccionada → INCLUIDA
```

### Registrar además
- Motivo de la excepción
- Versión del Toolkit
- Versión del dataset
- Hash de configuración
- Commit Git

### Mensaje
> **La flexibilidad del DS no elimina la trazabilidad; la convierte en una decisión explícita y auditable.**

---

## Slide 7 — Gobierno de algoritmos

### Título
**Flexibilidad sí, código libre no**

### Visual

```text
             DATA SCIENTIST
                    │
                    ▼
            Selecciona algoritmo
                    │
                    ▼
          ┌────────────────────┐
          │ Algorithm Registry │
          └─────────┬──────────┘
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
       APROBADO            NO DISPONIBLE
          │                    │
          ▼                    ▼
       Training          Feature Request
                               │
                               ▼
                         Toolkit / MLE
                               │
                               ▼
                         Nueva versión
```

### Bajo gobierno del COE
- Implementación de algoritmos
- Versiones
- Librerías
- Parámetros permitidos
- Validaciones
- Integración con MLflow

### Lo que obtiene el DS
- Catálogo transparente
- Parámetros configurables
- Documentación
- Proceso claro para solicitar nuevos algoritmos

---

## Slide 8 — Recomendación y roadmap

### Título
**Evolución propuesta**

```text
HOY
 │
 ▼
1. OVERRIDES
   force_include / force_exclude
 │
 ▼
2. FEATURE MANIFEST
   Dataset + selección + decisiones
 │
 ▼
3. WORKFLOWS DESACOPLADOS
   Features → Training
 │
 ▼
4. EVOLUCIÓN DEL TOOLKIT
   Registry + Requests + mayor reutilización
```

### Objetivo
> **Reducir el abandono del Toolkit sin reducir los controles de gobierno.**

### Recomendación ejecutiva
La alternativa objetivo es **desacoplar Feature/Data Preparation de Training**, comenzando por overrides controlados y evolucionando hacia dataset + manifest versionado.

### Frase de cierre
> **“No buscamos darle más libertad al DS; buscamos darle el espacio de experimentación que necesita, manteniendo dentro del Toolkit todo aquello que debe permanecer gobernado.”**

---

# Slide opcional — El Toolkit como producto

### Título
**El Toolkit como producto**

```text
Toolkit
   │
   ├── Tecnología
   │    ├── Databricks
   │    ├── MLflow
   │    └── GitHub
   │
   ├── Gobierno
   │    ├── Algoritmos
   │    ├── Datos
   │    └── Auditoría
   │
   └── Developer Experience
        ├── Configuración simple
        ├── Feedback rápido
        ├── Experimentación
        ├── Documentación
        └── Feature Requests
```

### Mensaje
El éxito no debería medirse únicamente por:

> “¿El workflow funciona?”

sino también por:

> **“¿El DS puede completar su ciclo analítico sin abandonar el Toolkit?”**

---

# KPIs sugeridos

## 1. Adoption Rate

Modelos que completan Training con Toolkit / Modelos iniciados con Toolkit

## 2. Notebook Escape Rate

Modelos que abandonan Toolkit / Modelos iniciados con Toolkit

## 3. Training Reuse Rate

Entrenamientos realizados reutilizando un Feature Dataset existente / Entrenamientos totales
