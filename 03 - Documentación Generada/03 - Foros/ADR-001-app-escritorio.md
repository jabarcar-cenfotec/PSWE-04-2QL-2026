# ADR-001: Aplicación de escritorio self-contained para consulta de historial médico en EBAIS

## Estado

Propuesto

## Contexto

El RFP exige que el médico en el punto de atención (EBAIS) pueda leer el historial clínico almacenado en un dispositivo portado por el paciente en **menos de 2 segundos**, incluso **sin conectividad ni electricidad estable**. Un caso representativo es un EBAIS en Hojancha, Guanacaste, con conectividad intermitente.

Esto impone tres restricciones duras sobre la arquitectura del cliente:

1. **Latencia de lectura**: la ruta crítica de lectura no puede depender de una llamada de red.
2. **Disponibilidad sin red**: la app debe funcionar completa (o en un modo degradado suficiente) sin internet.
3. **Acceso a hardware local**: el dispositivo médico del paciente se conecta por **USB o Bluetooth**, algo que un navegador no expone de forma confiable ni completa.

Se evaluaron tres opciones de arquitectura de cliente:

- **PWA**: descartada por soporte limitado/inconsistente de Web Bluetooth y WebUSB entre navegadores y SO, y por depender del navegador como runtime.
- **Cliente web tradicional**: descartado porque requiere red para operar, lo que viola directamente el requisito de los 2 segundos sin conectividad.
- **Aplicación de escritorio instalable, self-contained**: cumple los tres requisitos porque corre como proceso nativo con acceso directo a puertos USB/Bluetooth y puede leer datos locales sin red.

## Decisión

Se construirá una **aplicación de escritorio instalable** para Windows 10+, con las siguientes características:

- **Empaquetado self-contained**: el instalador incluye el runtime y todas las dependencias necesarias, sin requerir componentes preinstalados en el equipo del consultorio. Esto elimina el escenario de "no instaló porque falta tal librería" y da compatibilidad predecible entre equipos heterogéneos de los EBAIS.
- **Persistencia local con SQLite embebido**, usado para:
  - Configuración de la instalación (identificador de EBAIS, parámetros de conexión al CORE, preferencias locales).
  - Caché/registro del historial leído del dispositivo, para consulta rápida sin depender de la red.
- **Conexión directa al dispositivo médico del paciente** vía USB o Bluetooth, sin intermediarios de red para la lectura.
- **Portal web de distribución de instaladores**, separado de la app clínica, donde TI de cada centro descarga la versión más reciente del instalador bajo demanda. No se asume push automático hacia los equipos de consultorio.
- **Integración con un CORE central**, que concentra los servicios EDUS y expone *hooks*/eventos adicionales. La app de escritorio se conecta al CORE cuando hay red disponible (para sincronizar, notificar eventos, etc.), pero la lectura del historial del dispositivo del paciente **no depende de esa conexión**.

## Consecuencias

**Lo que se facilita:**

- Se cumple el requisito de lectura sub-2s sin red, porque la ruta de lectura es 100% local (dispositivo → app → SQLite/memoria).
- Instalación predecible en Windows 10+ sin depender del estado previo del equipo (drivers de .NET, VC++ redistributables, etc.).
- Acceso completo y estable a USB/Bluetooth, cosa que un navegador no puede garantizar hoy.
- TI de cada centro controla cuándo actualizar, sin depender de visitas físicas para cada instalación inicial.

**Lo que se vuelve más difícil o requiere decisión adicional (puntos que yo dejaría explícitos en este mismo ADR o en uno derivado, antes de aprobarlo):**

1. **Tamaño y distribución del instalador**: self-contained en .NET típicamente ronda 60–150+ MB según el runtime y las dependencias. En zonas con conectividad limitada (como Hojancha), descargar el instalador desde el portal puede ser lento. Vale la pena definir si el portal soporta descarga reanudable o distribución alterna (USB/imagen).
2. **Estrategia de actualización**: si TI debe entrar manualmente al portal a buscar la versión más reciente, hay riesgo de que EBAIS queden en versiones desactualizadas por meses. Conviene decidir si habrá algún mecanismo de notificación de versión nueva (aunque sea pasivo, ej. la app avisa "hay actualización disponible" al conectar con el CORE).
3. **Seguridad de los datos en reposo**: SQLite por sí solo no cifra los datos. Dado que se está guardando historial clínico (dato sensible bajo la Ley 8968 de Protección de la Persona frente al tratamiento de sus datos personales), recomendaría evaluar SQLCipher o cifrado a nivel de disco/columna para el archivo local, y dejarlo como decisión explícita, no implícita.
4. **Consistencia entre lo local y el CORE**: si la app cachea historial localmente y el CORE es la fuente de verdad, falta definir la política de sincronización/resolución de conflictos cuando la conectividad se recupera (¿last-write-wins?, ¿el CORE siempre gana?, ¿colas de eventos pendientes?).
5. **Certificación de hardware**: "USB o Bluetooth" es amplio. Convendría un ADR o sección aparte que liste los dispositivos/protocolos médicos soportados explícitamente, porque el driver y el SDK del fabricante pueden condicionar qué tan "self-contained" puede ser realmente la app (algunos dispositivos médicos requieren instalar su propio driver de fabricante, lo cual podría reintroducir una dependencia externa al instalador).
6. **Alcance futuro multiplataforma**: si en algún momento algún centro usa equipos no-Windows, la decisión actual (Windows 10+ nativo) implicaría reescritura o una capa adicional. No es necesariamente un problema ahora, pero vale la pena anotarlo como supuesto explícito de esta decisión (alcance = Windows únicamente).

## Alternativas descartadas (resumen)

| Opción | Motivo de descarte |
|---|---|
| PWA | Soporte inconsistente/insuficiente de Web Bluetooth/WebUSB entre navegadores y SO |
| Cliente web tradicional | Requiere red para funcionar; incumple el requisito de 2s sin conectividad |
