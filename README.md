# Diseño de arquitecturas de software robustas

Como analista SOA senior, tu tarea es definir arquitecturas de software que reflejen una aproximación efectiva a los atributos de calidad identificados. Debes considerar tanto aplicaciones web como mobile, y asegurar que las arquitecturas sean robustas y escalables. Los actores involucrados incluyen el equipo de desarrollo, el equipo de operaciones, y los stakeholders del negocio. Las propiedades operativas clave son la latencia (máximo 200ms en aplicaciones mobile), la disponibilidad (SLA 99.9%), y la escalabilidad (manejo de 10,000 usuarios concurrentes). Debes identificar y justificar las decisiones de diseño que tomes.

## Informacion General

| Campo | Valor |
|-------|-------|
| **Tema** | Definición de arquitecturas de software basadas en atributos de calidad |
| **Nivel** | senior-l2 |
| **Tipo** | theoretical |
| **Tiempo estimado** | 8 horas |

## Fases del Reto

### Fase 0: Configuración del Proyecto

**Objetivo:** Obtener el proyecto base funcional enviando el Código Base a un asistente de IA, que lo analizará, corregirá errores y generará un ZIP listo para usar.

**Tiempo estimado:** 15-30 minutos

**Instrucciones:**

- Asegúrate de tener instalado para ejecutar el proyecto: Un IDE o editor de código.
- Copia todo el contenido del campo **Código Base** de este reto — incluyendo el texto de instrucciones que aparece al inicio.
- Abre un asistente de IA (Claude en claude.ai, ChatGPT o Gemini — se recomienda Claude), pega el contenido copiado en el chat y envíalo.
- El asistente analizará los archivos, corregirá errores y generará un archivo ZIP descargable. Descárgalo y extráelo en la carpeta donde quieras trabajar.
- Verifica que el proyecto arranca sin errores.

**Entregable:** El proyecto compila/arranca sin errores.

<details>
<summary>Pistas de conocimiento</summary>

- Copia el Código Base completo incluyendo el texto de instrucciones al inicio — esas instrucciones le indican al asistente exactamente qué hacer con los archivos.
- Si el asistente no genera el ZIP automáticamente al terminar el análisis, escríbele: "genera el ZIP ahora".
- Si el proyecto tiene errores al arrancar, comparte el mensaje de error con el mismo asistente para que lo corrija.

</details>

### Fase 1: Exploración del dominio y atributos de calidad

**Objetivo:** Identificar y comprender los atributos de calidad relevantes para la arquitectura de software.

**Tiempo estimado:** 2 horas

**Instrucciones:**

- Investiga y documenta los atributos de calidad que son críticos para las aplicaciones web y mobile.
- Identifica las restricciones y ambigüedades en las especificaciones actuales del sistema.

**Entregable:** Documento que lista los atributos de calidad y las restricciones identificadas.

<details>
<summary>Pistas de conocimiento</summary>

- Considera la latencia, disponibilidad, y escalabilidad como atributos clave.
- Busca ejemplos de sistemas similares y sus arquitecturas.

</details>

### Fase 2: Definición de la arquitectura de software

**Objetivo:** Diseñar una arquitectura de software que refleje los atributos de calidad identificados.

**Tiempo estimado:** 3 horas

**Instrucciones:**

- Propón una arquitectura de software que considere tanto aplicaciones web como mobile.
- Justifica tus decisiones de diseño en términos de los atributos de calidad identificados.

**Entregable:** Diagrama y documentación de la arquitectura de software propuesta.

<details>
<summary>Pistas de conocimiento</summary>

- Considera la separación de preocupaciones, la modularidad, y la escalabilidad.
- Busca ejemplos de arquitecturas similares y sus ventajas/desventajas.

</details>

### Fase 3: Evaluación y refinamiento de la arquitectura

**Objetivo:** Evaluar y refinar la arquitectura de software propuesta.

**Tiempo estimado:** 3 horas

**Instrucciones:**

- Evalúa la arquitectura propuesta en términos de los atributos de calidad identificados.
- Identifica áreas de mejora y refina la arquitectura en consecuencia.

**Entregable:** Documento que evalúa y refina la arquitectura de software propuesta.

<details>
<summary>Pistas de conocimiento</summary>

- Considera el feedback de stakeholders y equipos técnicos.
- Busca ejemplos de arquitecturas refinadas y sus mejoras.

</details>

## Dimensiones Evaluadas

- **queEs**: ¿Qué son los atributos de calidad y por qué son importantes en la definición de arquitecturas de software?
- **paraQueSirve**: ¿Para qué sirve considerar los atributos de calidad al diseñar una arquitectura de software?
- **comoSeUsa**: ¿Cómo se usan los atributos de calidad para guiar las decisiones de diseño en una arquitectura de software?
- **erroresComunes**: ¿Cuáles son los errores comunes al definir arquitecturas de software sin considerar adecuadamente los atributos de calidad?
- **queDecisionesImplica**: ¿Qué decisiones implica el diseño de una arquitectura de software robusta que refleje los atributos de calidad identificados?

## Criterios de Evaluacion

- Identificación y comprensión de los atributos de calidad relevantes.
- Propuesta de una arquitectura de software que refleje los atributos de calidad identificados.
- Justificación de las decisiones de diseño en términos de los atributos de calidad.
- Evaluación y refinamiento de la arquitectura propuesta en base a feedback y mejores prácticas.

## Como trabajar con un asistente de IA

Hay dos caminos, elegi uno:

- **AGENTS.md** (recomendado) — instrucciones nativas del repo. Abri esta carpeta con tu agente local (Claude Code, Cursor, Codex, Copilot, Gemini) y las carga solo. Sabe que archivos faltan y con que comando se verifica, y completa el scaffold escribiendo en disco.
- **PROMPT_MEJORA.md** — para copiar y pegar en un chat (claude.ai, ChatGPT). Devuelve un ZIP con el proyecto. Sirve si no tenes un agente en el IDE.

Ninguno de los dos resuelve las fases del reto: eso es tu trabajo.

## Verificacion

El proyecto esta listo para trabajar cuando este comando corre sin errores:

```bash
npx --yes @redocly/cli lint contracts/openapi.yaml
```

---

*Reto generado automaticamente por Challenge Generator - Pragma*
