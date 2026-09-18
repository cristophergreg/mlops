# Slide final — Recomendación de evolución del Toolkit ML

## Título
**Recomendación: evolucionar hacia un Toolkit desacoplado y configurable**

## Mensaje principal
**Automatizado por defecto. Configurable por excepción. Gobernado de extremo a extremo.**

## Visual central sugerido

```text
                    TOOLKIT ML
                       │
                 Configuración
                       │
                       ▼
              ┌──────────────────┐
              │ Feature Workflow │
              │                  │
              │ Preprocesamiento │
              │ Selección auto.  │
              │ Overrides         │
              └────────┬─────────┘
                       │
                       ▼
             Dataset + Feature Manifest
                  V001 / V002 / V003
                       │
              ┌────────┴─────────┐
              ▼                  ▼
         Training 1          Training 2
          XGBoost             LightGBM
              │                  │
              └────────┬─────────┘
                       ▼
                     MLflow
```

## Evolución por etapas

**1. Ahora — Quick Win**  
Incorporar `force_include` / `force_exclude` con validaciones y motivo de la excepción.

**2. Siguiente paso — Feature Manifest**  
Versionar qué variables fueron seleccionadas automáticamente, cuáles fueron modificadas y cuál es el conjunto final.

**3. Evolución — Workflows desacoplados**  
Separar Feature Workflow y Training Workflow para reutilizar un dataset versionado y experimentar sin repetir toda la preparación.

**4. Visión futura — Toolkit como plataforma**  
Algorithm Registry, Feature Registry y Feature Requests para ampliar capacidades manteniendo el gobierno centralizado.

## Qué gana el DS
- Mayor capacidad de experimentación.
- Inclusión/exclusión controlada de variables.
- Reutilización de datasets preparados.
- Entrenamientos múltiples sin reprocesar todo.

## Qué mantiene el COE
- Algoritmos aprobados.
- Librerías y código productivo centralizados.
- Fuentes y transformaciones gobernadas.
- Versionado, trazabilidad y auditoría en MLflow.

## Cierre del slide
> **No buscamos darle más libertad al DS; buscamos darle el espacio de experimentación que necesita, manteniendo dentro del Toolkit todo aquello que debe permanecer gobernado.**

## Nota para exposición oral
La recomendación no es implementar todo de una vez. Primero resolvemos la fricción más visible con overrides controlados. Después desacoplamos preparación y entrenamiento mediante un dataset/manifest versionado. Así reducimos el incentivo de salir del Toolkit sin sacrificar gobierno, trazabilidad ni reproducibilidad.
