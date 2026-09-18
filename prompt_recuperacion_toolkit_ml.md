# Prompt de recuperación — Toolkit ML / Evolución MLOps

## Objetivo

Usa este prompt para reconstruir el contexto de la propuesta de evolución del Toolkit ML si se pierde información de la conversación.

## Prompt

Estoy trabajando como Machine Learning Engineer / MLOps en un COE de Advanced Analytics de banca y necesito continuar una propuesta de evolución de un Toolkit ML interno.

Contexto actual:
- Los proyectos parten de templates de GitHub.
- El Data Scientist (DS) no desarrolla código productivo; configura parámetros en un archivo.
- Un notebook generador convierte esa configuración en YAML y despliega un Databricks Workflow secuencial.
- El backend del Workflow está controlado por librerías internas y entry points fijos.
- El flujo actual realiza preparación/preprocesamiento, selección automática de variables (variables ganadoras) y entrenamiento con MLflow.
- El preprocesamiento ya genera un reporte detallado de influencia y selección de variables.
- Por gobierno/riesgo bancario, los DS no deben ejecutar código arbitrario ni algoritmos fuera del catálogo/librerías internas. Nuevos algoritmos deben pasar por un Feature Request hacia Toolkit/MLE.

Problema principal:
- Los DS usan el Toolkit para la primera etapa, pero algunos abandonan el Toolkit para modelar en notebooks externas.
- Una causa importante es la rigidez posterior a la selección automática: al revisar resultados, el DS puede necesitar forzar la inclusión de una variable, excluir una variable seleccionada o experimentar con distintos entrenamientos.
- Si cualquier cambio obliga a repetir todo el proceso, se genera fricción y aumenta el abandono del Toolkit.

Principio de solución:
"Automatizado por defecto. Configurable por excepción. Gobernado de extremo a extremo."

Recomendación acordada:
Evolucionar progresivamente hacia un Toolkit desacoplado, separando preparación de features/datos del entrenamiento.

Arquitectura objetivo:
Config → Feature Workflow → Dataset versionado + Feature Manifest → Training Workflow → MLflow

Estrategia por etapas:
1. Quick Win: incorporar overrides controlados de selección, por ejemplo force_include y force_exclude.
2. Feature Manifest: registrar selección automática, excepciones, lista final, versión del Toolkit, configuración y lineage.
3. Desacoplar workflows: generar un dataset versionado que pueda reutilizarse para múltiples entrenamientos sin repetir la preparación.
4. Evolución futura: Algorithm Registry / Feature Registry y un proceso formal de Feature Requests.

Regla conceptual para overrides:
FINAL = (AUTOMATIC_SELECTION - FORCE_EXCLUDE) ∪ FORCE_INCLUDE

Los overrides deben validar que:
- una variable no esté simultáneamente en include y exclude;
- no existan duplicados;
- las variables pertenezcan al universo aprobado;
- no se puedan incorporar fuentes o código arbitrario;
- cada excepción pueda registrar reason_code y reason.

Ejemplo de configuración:
feature_selection:
  overrides:
    force_include:
      - name: VAR_A
        reason_code: BUSINESS_REQUIREMENT
        reason: "Variable solicitada por negocio"
    force_exclude:
      - name: VAR_B
        reason_code: ANALYTICAL_JUDGEMENT
        reason: "Variable descartada por criterio analítico"

Gobierno:
- DS configura universo, tablas candidatas, force include/exclude y parámetros permitidos.
- Toolkit/MLE controla implementación, librerías, algoritmos aprobados, infraestructura, validaciones, MLflow y auditoría.
- Algoritmos nuevos no se agregan mediante código libre del DS: se solicitan mediante Feature Request y se incorporan centralmente en una nueva versión del Toolkit.

MLflow / auditoría:
Registrar como mínimo proyecto/modelo/run, Git commit, config hash, versión Toolkit, dataset/version, fuente de datos, método y versión de selección, variables automáticas, forzadas, excluidas y finales, además de razones de las excepciones.

Mensaje central para una presentación:
No buscamos darle código libre al DS; buscamos darle el espacio de experimentación que necesita dentro de límites controlados, evitando que tenga que salir del Toolkit.

Si te pido continuar esta propuesta, prioriza claridad ejecutiva y lenguaje entendible para personas técnicas y no técnicas. Evita convertirla en una discusión puramente técnica. La recomendación debe quedar clara: empezar con overrides y evolucionar hacia Feature Workflow + dataset/manifest versionado + Training Workflow independiente.

## Skill / criterios a considerar

- Presentación ejecutiva: problema → alternativas → recomendación → roadmap.
- MLOps y gobierno: reproducibilidad, versionado, lineage, auditoría y separación de responsabilidades.
- Data/ML Platform: separación de preparación de datos/features y entrenamiento.
- Developer Experience: reducir fricción y evitar que el usuario abandone la plataforma.
- No proponer código libre para DS ni eliminar controles de gobierno.
- Mantener la propuesta incremental: Quick Win primero, arquitectura objetivo después.
- Si se solicita un slide, priorizar poco texto, una idea principal y un flujo visual simple.
