# ADR-002: Arquitectura híbrida a nivel macro (SOA + monolito modular + procesamiento por eventos)

## Estado
Propuesto

## Contexto

El ADR-001 resolvió el problema del **cliente** (EBAIS): app de escritorio self-contained, con lectura 100% local y sincronización oportunista con el CORE. Ese ADR dejó abierta una pregunta de nivel superior: cómo se organiza el **CORE central** y cómo se comunica con el resto del ecosistema (EDUS, CCSS, los clientes de escritorio distribuidos en cada EBAIS).

El RFP impone tensiones que operan a este nivel macro, no en el cliente:

1. **RNF-01**: toda entrada al EDUS debe pasar por validación centralizada antes de persistirse como oficial.
2. **RNF-07**: 99.999% de disponibilidad, con penalización financiera por incumplimiento, y referencia explícita a la infraestructura de mensajería orientada a servicios de la CCSS — es decir, el sistema debe integrarse a un ecosistema que ya opera bajo un estilo SOA.
3. El ADR-001 ya estableció que la lectura local (RNF-02/04) no depende del CORE, pero la **escritura/validación** y el **registro extemporáneo** de eventos generados sin conexión sí deben llegar al CORE cuando la red se recupera.

Esto plantea tres preguntas de diseño a nivel de sistema:

- ¿Cómo se organiza internamente el CORE (EDUS + validación)?
- ¿Cómo se ingiere y valida el volumen de eventos que llegan de forma diferida desde EBAIS con conectividad intermitente (como Hojancha)?
- ¿Cómo se integra todo esto con la infraestructura de mensajería de la CCSS sin duplicar responsabilidades?

Se evaluaron tres estilos macro:

- **Monolito modular puro para todo el sistema**: descartado porque no calza con el requisito explícito de integrarse a la infraestructura de mensajería orientada a servicios de la CCSS (RNF-07), y porque acoplaría la ingesta de eventos diferidos al mismo ciclo de despliegue que el núcleo de validación EDUS.
- **Microservicios completos**: descartado por sobre-ingeniería frente al alcance del RFP; no hay evidencia en el RFP de múltiples equipos independientes ni de necesidad de escalado granular por servicio, que es lo que justificaría microservicios sobre SOA.
- **Híbrido — SOA a nivel de sistema, con monolito modular como uno de los servicios y un componente de ingesta orientado a eventos**: se ajusta a lo que el RFP ya exige (integración SOA con CCSS) sin introducir complejidad no solicitada.

## Decisión

Se adopta una **arquitectura híbrida a nivel macro**, compuesta por:

- **SOA como estilo de integración del sistema**: el CORE se expone como un servicio (o conjunto acotado de servicios) dentro de la infraestructura de mensajería orientada a servicios de la CCSS, consistente con RNF-07. Esta es la capa de comunicación entre el CORE, EDUS y otros sistemas de la CCSS.
- **Monolito modular para el CORE EDUS/validación**: la lógica de validación centralizada (RNF-01) vive en un solo despliegue con módulos internos bien delimitados (validación, persistencia oficial EDUS, reglas de negocio), en lugar de fragmentarse en microservicios. Esto reduce la complejidad operativa de un equipo pequeño manteniendo cohesión sobre la regla de negocio más crítica del sistema.
- **Componente de Ingesta y Validación orientado a eventos**: recibe los registros generados por las apps de escritorio (incluyendo Registro Extemporáneo desde EBAIS que estuvieron desconectados) y los encola para validación por el CORE. Este componente absorbe los picos de sincronización cuando un EBAIS recupera conectividad, sin bloquear al CORE ni a otros EBAIS que sincronizan al mismo tiempo.
- **La app de escritorio (ADR-001) permanece sin cambios**: sigue operando su ruta de lectura 100% local; lo único que cambia es a qué se conecta cuando hay red — al componente de Ingesta y Validación, no directamente al monolito del CORE.

## Consecuencias

**Lo que se facilita:**
- Cumple RNF-07 usando el mecanismo de integración que la CCSS ya exige (SOA), sin inventar una infraestructura de comunicación paralela.
- RNF-01 queda protegido: toda escritura oficial al EDUS pasa por el monolito de validación, sin importar si el evento llegó en tiempo real o de forma diferida.
- El pico de sincronización de EBAIS con conectividad intermitente (como Hojancha) no degrada la disponibilidad del CORE, porque el componente de Ingesta y Validación absorbe la carga de forma asíncrona.
- El monolito modular es más simple de operar y mantener que microservicios, coherente con el alcance del RFP.

**Lo que se vuelve más difícil o requiere decisión adicional:**
1. **Política de resolución de conflictos en la Ingesta**: si dos EBAIS sincronizan eventos sobre el mismo paciente en ventanas cercanas, falta definir el orden de validación (¿por timestamp de captura, no de llegada?). Este punto ya había quedado abierto en el ADR-001 (punto 4) y ahora se resuelve específicamente en el componente de Ingesta.
2. **Garantía de entrega**: si el componente de Ingesta falla o se reinicia mientras procesa la cola, hay que definir si los eventos encolados se pierden o se reprocesan (idempotencia del lado del monolito).
3. **Definición de contrato con la infraestructura SOA de la CCSS**: el RFP referencia la infraestructura de mensajería de la CCSS pero no especifica el protocolo/formato exacto; habría que confirmarlo antes de detallar la integración a nivel de C4 Nivel 2.
4. **Cumplimiento de RNF-07 con penalización financiera**: la disponibilidad del 99.999% aplica ahora a un sistema con más partes móviles (SOA + monolito + componente de eventos); conviene dejar explícito cuál de estos componentes es el que efectivamente mide el SLA, o si aplica al conjunto.

## Alternativas descartadas (resumen)

| Opción | Motivo de descarte |
|---|---|
| Monolito modular puro para todo el sistema | No calza con la integración SOA exigida por RNF-07 vía infraestructura de mensajería de la CCSS |
| Microservicios completos | Sobre-ingeniería frente al alcance y evidencia del RFP |
