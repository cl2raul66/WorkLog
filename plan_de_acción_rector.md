# Documento Arquitectónico de Referencia: WorkLog App

## Preámbulo: La Filosofía de WorkLog

WorkLog no es simplemente una aplicación de toma de notas. Nace para resolver un problema fundamental en entornos de alta exigencia (agrícolas, industriales): la **pérdida de valor en la transición de la palabra hablada a los datos estructurados**. Cada vez que un operario anota algo en un papel o en una app genérica, se pierde contexto, se introducen errores y se retrasa la toma de decisiones.

Nuestra filosofía es que la aplicación debe **adaptarse al usuario, no al revés**. El flujo de trabajo del usuario es la fuente de la verdad; la tecnología es la herramienta que debe aprender y servir a ese flujo. Por lo tanto, este documento no solo define una arquitectura de software, sino que establece los principios de un sistema que evoluciona, aprende y, en última instancia, se vuelve invisible para el usuario, permitiéndole centrarse en su trabajo, no en la entrada de datos.

Este plan es el contrato que garantiza que cada decisión técnica sirva a esta filosofía.

## 1. Fundamentos Arquitectónicos No Negociables

Estos son los pilares inmutables de la arquitectura. Cualquier propuesta que viole estos principios será rechazada por definición.

### 1.1. Principio de Portabilidad del Dominio

**Declaración:** *La lógica de negocio y las reglas del dominio de WorkLog deben ser agnósticas a cualquier plataforma, UI o tecnología de persistencia.*

**Justificación Ideológica:**
El valor real de WorkLog no reside en su interfaz MAUI, sino en su capacidad para entender y estructurar el lenguaje del usuario. Esta "inteligencia" es un activo que debe ser portable. Mañana podría ejecutarse en un servicio de Azure para procesar correos electrónicos, en una aplicación web para análisis de gestión, o en un dispositivo IoT en el campo. Acoplar nuestra lógica de dominio a MAUI o a LiteDB sería como soldar el motor de un coche al chasis: limitaría su valor a un único caso de uso y haría cualquier evolución futura prohibitivamente costosa.

**Implementación Arquitectónica: Separación Física de Responsabilidades**
Para hacer cumplir este principio, adoptamos una arquitectura en capas estricta, materializada en proyectos de .NET separados:

*   **`WorkLog.Core`:** El santuario del dominio. Contiene únicamente los modelos (`JobRecord`), las abstracciones (`IRecordRepository`) y las reglas de negocio puras. No tiene dependencias externas, salvo las mínimas de .NET. Es el "cerebro" portable de la aplicación.
*   **`WorkLog.Infrastructure`:** El mundo real. Contiene las implementaciones concretas y "sucias": el código que habla con la base de datos (LiteDB), el que invoca al modelo de IA (ONNX Runtime), el que genera PDFs (SkiaSharp). Depende de `Core`, pero `Core` no sabe que `Infrastructure` existe.
*   **`WorkLog.Maui`:** La cara visible. Contiene las Vistas y ViewModels. Es un mero consumidor de los servicios definidos en `Core` e implementados en `Infrastructure`. Si mañana decidimos crear una versión en Blazor, solo esta capa se reemplazaría.

**Estructura Definitiva:**
(El diagrama de estructura de ficheros presentado anteriormente es la manifestación física de este principio y se mantiene como válido).

### 1.2. Principio de Inmutabilidad del Origen

**Declaración:** *La entrada original del usuario (`OriginalInput`) es un artefacto sagrado y nunca debe ser modificado o perdido.*

**Justificación Ideológica:**
La confianza del usuario en el sistema depende de la certeza de que sus datos no se corrompen. El `OriginalInput` es la evidencia forense de lo que el usuario realmente dijo o escribió. Si nuestro sistema de parseo comete un error, debemos ser capaces de re-procesar la entrada original con una lógica mejorada. Si hay una disputa de auditoría, la entrada original es la única prueba válida. Modificarla sería destruir la evidencia.

**Implementación Arquitectónica: Modelo de Datos Canónico**
El modelo `JobRecord` está diseñado para hacer cumplir este principio:

```csharp
public sealed class JobRecord
{
    public Guid Id { get; init; } // `init` para inmutabilidad post-creación
    public string OriginalInput { get; init; } // `init` para garantizar que no se altere
    public DateTime Timestamp { get; init; } // `init` para registrar el momento exacto
    
    // Propiedades mutables que representan el estado del procesamiento
    public Guid? WorkTemplateId { get; set; }
    public List<ParsedItem>? ParsedItems { get; set; }
    public RecordStatus Status { get; set; }
    // ...
}
```
Las propiedades clave son `init-only`, lo que significa que solo pueden ser establecidas durante la creación del objeto. El resto de las propiedades (`ParsedItems`, `Status`) representan el *resultado del análisis* de la entrada original, no una alteración de la misma.

### 1.3. Principio de Inteligencia Local y Offline-First

**Declaración:** *Toda la funcionalidad crítica de la aplicación, incluido el análisis inteligente, debe funcionar sin conexión a internet.*

**Justificación Ideológica:**
Nuestros usuarios operan en entornos donde la conectividad es un lujo, no una garantía: campos, almacenes, sótanos. Una aplicación que depende de la nube para su funcionalidad principal es una aplicación inútil en estos contextos. La inteligencia no puede ser un servicio remoto; debe residir en el dispositivo, garantizando una respuesta instantánea y una fiabilidad absoluta, independientemente del estado de la red.

**Implementación Arquitectónica: Motor de Inteligencia Híbrido Embebido**

*   **Modelo ONNX Local:** Se ha seleccionado un modelo NER (Named Entity Recognition) en formato ONNX (`dslim/bert-base-NER` cuantizado a INT8). ONNX es un estándar abierto que nos libera de cualquier proveedor específico. El modelo se embebe como un recurso dentro del ensamblado de `Infrastructure` y se extrae al dispositivo en la primera ejecución. Esto garantiza que la IA siempre esté disponible.
*   **Lógica de Refinamiento Contextual:** El modelo de IA proporciona una comprensión genérica ("esto es una persona", "esto es un lugar"). Nuestra lógica de negocio en `Infrastructure` refina esta predicción usando el contexto específico del usuario (sus plantillas, sus keywords). Este enfoque híbrido combina el poder del deep learning con la precisión del conocimiento del dominio, todo ello ejecutándose localmente.

### 1.4. Principio de Escalabilidad y Resiliencia de Datos

**Declaración:** *La persistencia de datos debe ser inmune a la corrupción catastrófica y su rendimiento no debe degradarse con el tiempo.*

**Justificación Ideológica:**
Un único fichero de base de datos monolítico es una bomba de tiempo. A medida que crece, las escrituras se ralentizan, la indexación se vuelve pesada y, lo más peligroso, un único bit corrupto puede destruir años de datos. Esto es inaceptable. La arquitectura de datos debe estar diseñada para la longevidad y la robustez, tratando los datos del usuario como el activo más valioso que son.

**Implementación Arquitectónica: Particionamiento de Datos Basado en el Tiempo con LiteDB**
Rechazamos el enfoque simplista de un solo `worklog.db`. En su lugar, implementamos un patrón de particionamiento de datos:

*   **Aislamiento de Dominios por Fichero:**
    *   **`config.db`:** Un fichero pequeño y estable para datos de configuración (`WorkTemplate`, `AppSettings`). Su riesgo de corrupción es mínimo debido a su baja frecuencia de escritura.
    *   **`records_YYYY_MM.db`:** Ficheros de datos transaccionales, particionados por mes. Las escrituras siempre ocurren en el fichero del mes actual, que es de tamaño limitado. Los ficheros de meses anteriores se convierten en archivos de solo lectura (conceptualmente inmutables).

*   **Beneficios de esta Arquitectura:**
    1.  **Aislamiento de Fallos (Blast Radius Reduction):** La corrupción de un fichero de archivo (ej: `records_2025_08.db`) no afecta en absoluto a los datos actuales ni a la configuración. El "radio de la explosión" está contenido.
    2.  **Rendimiento Sostenido:** Las operaciones de escritura e indexación siempre operan sobre un fichero de tamaño manejable, garantizando una latencia baja y predecible a lo largo del tiempo.
    3.  **Gestión de Ciclo de Vida:** Este patrón simplifica drásticamente futuras funcionalidades como el archivado, la purga de datos antiguos o los backups incrementales.

### 1.5. Principio de Consistencia Atómica

**Declaración:** *Las operaciones de negocio complejas que afectan a múltiples partes del sistema deben ser atómicas: o tienen éxito por completo, o fallan por completo sin dejar estados intermedios corruptos.*

**Justificación Ideológica:**
Consideremos la transición del modo "Learning" al modo "Estructurado". Esta operación implica: 1) guardar una nueva plantilla, 2) re-procesar docenas de registros antiguos, y 3) cambiar un ajuste global de la aplicación. Si el paso 2 falla a la mitad, nos quedamos con un sistema en un estado inconsistente y corrupto. El usuario no puede confiar en un sistema que puede romperse a sí mismo.

**Implementación Arquitectónica: Patrón Unit of Work (Unidad de Trabajo)**
Para garantizar la atomicidad, la capa de abstracción de datos (`WorkLog.Core`) define una interfaz `IUnitOfWork`.

*   **Contrato de `IUnitOfWork`:** Expone las interfaces de los repositorios y los métodos para controlar una transacción (`BeginTransactionAsync`, `CommitAsync`, `RollbackAsync`).
*   **Orquestación:** Los servicios de alto nivel (como el `OnboardingOrchestrator`) no interactúan directamente con los repositorios. En su lugar, reciben una `IUnitOfWork` y envuelven toda la operación de negocio dentro de una única transacción. Si cualquier paso falla, se invoca `RollbackAsync`, y la base de datos vuelve a su estado original, como si la operación nunca hubiera ocurrido. Esto garantiza la integridad de los datos en todo momento.

---

## 2. Fases Evolutivas del Producto

El desarrollo de WorkLog se abordará en fases, cada una construida sobre los principios anteriores y con un objetivo estratégico claro.

### 2.1. Fase 1: El Sistema Determinista Confiable

**Objetivo Estratégico:** Entregar una herramienta de producción **predecible y 100% fiable** para el usuario experto. En esta fase, la confianza es más importante que la inteligencia. El sistema debe hacer exactamente lo que se le dice, sin ambigüedad.

**Alcance Ideológico:**
*   Gestión de Plantillas: El usuario define explícitamente su vocabulario.
*   Captura de Registros: El parseo se basa en reglas estrictas (keywords, regex). El resultado es binario: o coincide o no.
*   Revisión y Corrección: El sistema identifica claramente lo que no entiende y pide ayuda al usuario.
*   Reportes: Se generan reportes basados únicamente en datos verificados (`RecordStatus.Verified`).

### 2.2. Fase 2: El Asistente Proactivo

**Objetivo Estratégico:** Reducir la carga cognitiva del usuario. El sistema pasa de ser una herramienta pasiva a un **asistente activo** que aprende de los errores y sugiere mejoras.

**Alcance Ideológico:**
*   Generación de Sugerencias: Cuando el parseo determinista falla, el sistema no se rinde. Activa su motor de IA (`OnnxIntelligenceService`) en segundo plano para analizar el término no reconocido y proponer una clasificación.
*   Aprendizaje No Intrusivo: Las sugerencias se presentan de forma pasiva (un badge numérico), respetando el flujo de trabajo del usuario. El sistema sugiere, el usuario decide.
*   Ciclo de Retroalimentación: Al aceptar o rechazar una sugerencia, el usuario está entrenando y mejorando el vocabulario del sistema, haciendo que el motor determinista sea más inteligente para la siguiente vez.

### 2.3. Fase 3: El Onboarding Inteligente

**Objetivo Estratégico:** Eliminar la barrera de entrada para el usuario novato. La aplicación debe ser **útil desde el primer segundo**, sin necesidad de configuración manual.

**Alcance Ideológico:**
*   Modo de Aprendizaje (Learning Mode): La aplicación arranca en un modo sin plantillas. El usuario simplemente habla o escribe. El sistema escucha y almacena las notas en su forma cruda (`RecordStatus.Unstructured`).
*   Descubrimiento de Patrones: Tras acumular un número suficiente de registros, el `OnboardingOrchestrator` analiza el corpus de notas del usuario, identifica entidades recurrentes usando el motor de IA y la frecuencia estadística, y **descubre la estructura latente** en el trabajo del usuario.
*   Transición Asistida: El sistema presenta una plantilla pre-generada al usuario. Al aceptarla, la aplicación transiciona al modo estructurado y, utilizando una `IUnitOfWork`, re-procesa atómicamente todos los registros históricos con la nueva plantilla. La aplicación ha aprendido y configurado por sí misma.

---

## Conclusión

Este documento define el ADN de WorkLog. No es una lista de tecnologías, sino un conjunto de principios que guiarán nuestro desarrollo. Cada línea de código, cada decisión de diseño, será medida contra estos fundamentos. Al adherirnos a esta ideología, construiremos un producto que no solo es técnicamente sólido, sino que también cumple nuestra promesa fundamental: hacer que la tecnología sirva al trabajo humano de la manera más fluida y eficaz posible.