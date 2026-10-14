---
domain: agent-knowledge
tags: [agent-knowledge]
sources: []
updated: 2026-09-30
---

# RDF, OWL y Turtle

## What it is

**RDF** (Resource Description Framework) es el formato estándar del W3C para expresar conocimiento como triples: `sujeto → predicado → objeto`. Cada triple es una afirmación sobre el mundo.

**OWL** (Web Ontology Language) es un vocabulario construido encima de RDF que permite definir esquemas: qué tipos de cosas existen, cómo se relacionan, y qué restricciones aplican.

**Turtle** (`.ttl`) es una sintaxis concreta para escribir RDF — compacta y legible por humanos. Otros formatos (RDF/XML, JSON-LD) expresan lo mismo con distinta sintaxis.

## What problem it solves

Sin un estándar compartido, cada sistema de conocimiento inventa su propio esquema. RDF/OWL permite que ontologías creadas en un sistema (o por el W3C) sean legibles, reutilizables e importables por cualquier otro sistema compatible.

## Notes

### La regla central de RDF

Todo es un triple con tres partes:

```
sujeto  →  predicado  →  objeto  .
```

Siempre termina en punto. Eso es todo.

### La distinción fundamental: esquema vs. instancias

| | RDF/OWL | Analogía con base de datos |
|---|---|---|
| **Esquema (OWL)** | Clases y propiedades | `CREATE TABLE` |
| **Instancias (RDF)** | Recursos concretos extraídos | Filas de datos |

**OWL = la plantilla vacía.** Define qué tipos existen y cómo se relacionan.
**RDF (completo) = plantilla + todos los datos.** Incluye las instancias reales.

### Cómo identificar clases vs. instancias en Turtle

**Clase (definición de tipo)** — siempre tiene `owl:Class`:
```turtle
<urn:utopia:kb:...:class:budget>  a  owl:Class ;
    rdfs:label    "Presupuesto" ;
    rdfs:comment  "Una asignación de dinero o recursos..." .
```
Lectura: *"budget" es un tipo. Su nombre es "Presupuesto".*

**Propiedad / Relación** — siempre tiene `owl:ObjectProperty` o `owl:DatatypeProperty`:
```turtle
<urn:utopia:kb:...:relation:tiene_cobertura>  a  owl:ObjectProperty ;
    rdfs:domain  <urn:utopia:kb:...:class:poliza> ;
    rdfs:range   <urn:utopia:kb:...:class:cobertura> .
```
Lectura: *"tiene_cobertura" es una relación. El lado izquierdo debe ser Poliza, el derecho Cobertura.*

**Instancia con tipo asignado** — tiene `a <...:class:algo>`:
```turtle
<urn:utopia:kb:...:entity:01a0d8e6-...>  rdfs:label "memory" ;
    a  <urn:utopia:kb:...:class:system_component> .
```
Lectura: *Se extrajo "memory" y se clasificó como tipo "system_component".*

**Instancia sin tipo (sin emparejar)** — tiene solo `rdfs:label`, sin `a`:
```turtle
<urn:utopia:kb:...:entity:01a0d8e6-...>  rdfs:label "Agent" .
```
Lectura: *Se extrajo "Agent" pero aún no tiene tipo asignado. Aparece en "Sin emparejar".*

### Las 4 secciones de un archivo .ttl exportado de Utopia

1. **Prefijos** (líneas 1–7) — atajos para no repetir URLs largas
2. **Clases OWL** — los tipos de entidad (el esquema de la ontología)
3. **Propiedades OWL** — los tipos de relación
4. **Instancias** — documentos subidos + entidades extraídas + hechos

### Sintaxis Turtle: puntuación

| Símbolo | Significado | Ejemplo |
|---|---|---|
| `.` | Fin de la afirmación | `rdfs:label "Agent" .` |
| `;` | Mismo sujeto, siguiente propiedad | `rdfs:label "X" ; a owl:Class .` |
| `a` | "es de tipo" (abreviatura de `rdf:type`) | `"memory" a system_component` |
| `<...>` | URI — identificador único | `<urn:utopia:kb:...:class:budget>` |
| `"..."` | Valor de texto literal | `"Presupuesto"` |
| `@prefix` | Define un atajo | `@prefix owl: <http://...owl#> .` |

### El sistema de prefijos

Al inicio del archivo:
```turtle
@prefix owl:  <http://www.w3.org/2002/07/owl#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
```

`owl:Class` es un atajo para `<http://www.w3.org/2002/07/owl#Class>`. Misma cosa, más corta. Todos los prefijos funcionan igual.

### Qué son las URLs estándar (`http://www.w3.org/...`)

El W3C (la organización que define HTML, CSS, etc.) publica estándares y les asigna una URL como identificador permanente. No es una página que visitas; es un nombre globalmente único que identifica la especificación. Cuando escribes `owl:Class` estás diciendo "la clase tal como la define ese estándar del W3C".

### Qué es un URN

Un URN es un nombre permanente que no apunta a una dirección web:

| Tipo | Ejemplo | Significa |
|---|---|---|
| URL | `https://example.com/page` | Una ubicación (puede romperse) |
| URN | `urn:isbn:978-3-16-148410-0` | Un ID permanente (ISBN de un libro) |

En Utopia: `urn:utopia:kb:01a0d8c1-...:entity:...` es el ID permanente de esa entidad dentro de esa KB. No es una web, es un nombre que nunca colisionará con nada.

### Señal de la extracción rota visible en el .ttl

En una KB sana habría docenas de propiedades OWL:
```turtle
<...:relation:tiene_cobertura>  a  owl:ObjectProperty ;
    rdfs:domain  <...:class:poliza> ;
    rdfs:range   <...:class:cobertura> .
```

En una KB creada con la arquitectura actual de Utopia (open extraction activada), la sección de propiedades tiene solo `known_as`. Esto confirma la rotura del pipeline de auto-creación de relaciones descrita en [[utopia]].

## Relations

- [[utopia]] — el sistema que genera estos archivos .ttl; la rotura de auto-creación de tipos de relación es visible directamente en el export
- [[graphrag/index|graphrag]] — GraphRAG también construye grafos de conocimiento, pero no usa OWL como esquema

## Tags

#agent-knowledge
