# Análisis técnico — Resility / Zentris

Análisis de seguridad e IA de **Resility** (empresa chilena de ciberseguridad gestionada,
[resility.io](https://resility.io/)) y su producto propio **Zentris** (SOC agéntico con IA),
realizado el 2026-11-20.

## Metodología y alcance

Este análisis se hizo con **reconocimiento pasivo sobre información pública**:
contenido del sitio web, metadatos HTTP, `robots.txt`, datos estructurados (JSON-LD),
JavaScript servido públicamente a cualquier visitante, y descripciones públicas del
producto (sitio, FAQ, material de ProChile). **No se realizaron pruebas activas,
escaneos de vulnerabilidades, intentos de acceso, ni interacción con la plataforma
Zentris en sí** (que es un producto SaaS para clientes, no una aplicación descargable
ni un binario que se pueda extraer o decompilar).

Se evaluó inicialmente usar [REA](https://github.com/morluto/rea) ("Reverse Engineer
Anything"), una herramienta MIT genérica de reverse engineering de binarios/apps
JS-Electron/sitios web. Como Zentris no expone un artefacto descargable (app, binario,
bundle Electron) sino solo un sitio de marketing estático, el análisis real equivale al
método de "browser observation" que REA documenta para sitios web: inspección de
estructura de página, scripts cargados y requests de red — hecho aquí directamente.

**Importante:** este documento describe lo que Resility *dice públicamente* que hace
Zentris, y lo que es observable desde el exterior de su sitio de marketing. No es una
auditoría de su código fuente, su infraestructura interna ni sus controles reales: esos
datos no son públicos y no se intentó acceder a ellos. Las cifras de rendimiento citadas
(SLA, % de falsos positivos, etc.) son afirmaciones del proveedor, no verificadas de forma
independiente.

## 1. Qué es Resility

- Empresa chilena de ciberseguridad gestionada (MSSP), fundada en 2016, con SOC propio
  en Santiago y presencia en Chile, Perú, Colombia y España.
- Tres líneas de negocio bajo un mismo grupo: **Resility** (servicios gestionados),
  **Zentris** (producto propio de IA) y **Staff21x** (agentes de IA para automatización
  B2B general).
- Certificada ISO/IEC 27001:2022. Postula cumplimiento de normativa chilena: Ley 21.663
  (Marco de Ciberseguridad / ANCI) y Ley 21.719 (protección de datos personales), además
  de requisitos CMF para el sector financiero.
- Servicios: CyberSOC 24×7, MDR gestionado, NOC-as-a-Service, gestión de vulnerabilidades
  (vía Tenable), CAASM, gestión de endpoints/firewalls, ethical hacking, pentesting de
  apps web/móviles/APIs, red/purple team, respuesta a incidentes, forense digital,
  capacitación (Resility Academy) y concientización/phishing simulado.
- Stack de terceros que dicen operar: **Wazuh** (SIEM/XDR open source), **Tenable**
  (Nessus / exposure management), **Bitdefender GravityZone** (EDR/antimalware),
  **Grafana Enterprise** (dashboards), **Akamai** (nube de alta disponibilidad, 99.9%
  uptime reclamado), Fortinet/Check Point/Palo Alto/pfSense (firewalls), Microsoft
  Sentinel/365, y framework **MITRE ATT&CK®** para mapeo de detecciones.

## 2. Zentris — arquitectura de IA (según fuentes públicas)

Zentris se describe como una **plataforma agéntica de defensa** ("ataques con agentes,
defensa con agentes"), reconocida por ProChile como tecnología chilena destacada en
misiones comerciales (Colombia, Perú, Nueva York, Portugal).

Afirmaciones de arquitectura y funcionamiento (fuente: sitio público, no verificadas
de forma independiente):

| Característica | Descripción pública |
|---|---|
| Composición | "~20 agentes especializados autónomos" |
| Modelo de despliegue | **Agentless** — se integra sobre el EDR/SIEM/XDR existente del cliente vía API en la nube, sin agentes pesados ni hardware adicional |
| Tiempo de despliegue | 24–48 horas |
| Pipeline de operación | 1) Ingesta SIEM/XDR → 2) Triage autónomo por agentes Zentris con enriquecimiento de threat intel → 3) Validación humana N1/N2 + **EITL** (Expert-in-the-loop) para falsos positivos/contexto crítico → 4) Respuesta: aislamiento de hosts, escalamiento, reporte con SLA |
| Pentesting continuo | Módulo de "agentes ofensivos" que prueban la superficie expuesta 24/7; al corregir una vulnerabilidad la alerta se cierra sola, sin re-test manual programado |
| Exposure management | Revisión continua de sitios/superficie expuesta a internet, "sin tope de URLs" |
| NOC unificado | Métricas de disponibilidad/desempeño en el mismo panel |
| Cumplimiento | Generación automática de evidencia para Ley 21.663 (ANCI), ISO 27001, CMF y NIST CSF, con reportería en español para directorio/regulador |
| Integración/alertamiento | Slack, Microsoft Teams, Discord, Webhooks |
| Mapeo de amenazas | Basado en MITRE ATT&CK® |

Métricas de marketing (no auditadas por terceros, textuales del sitio):

- Detección: **< 15 min**
- Reducción de falsos positivos: **-93%**
- Despliegue: **24–48 h**
- Costo vs. SOC tradicional: **-85%**

### Lectura crítica de la arquitectura de IA

- El modelo "agentless + multiagente + EITL" es coherente con el patrón de industria
  actual de **SOC aumentado por IA** (triage automatizado, analista humano como última
  validación), no un reemplazo total de analistas — el propio sitio mantiene N1/N2/EITL
  en el loop para decisiones críticas, lo cual es una buena práctica (humano-en-el-loop
  para contención y falsos negativos costosos).
- "~20 agentes" sin detalle de qué modelo(s) LLM, orquestador o framework multiagente
  usan internamente (no publicado). No hay info pública sobre guardrails, sandboxing de
  las acciones de los agentes ofensivos (pentesting autónomo), ni sobre cómo se evita que
  un agente de "pentesting continuo" cause impacto no deseado en producción del cliente.
- Las cifras (-93% falsos positivos, -85% costo, SLA <15min) son comparaciones de
  marketing sin metodología publicada (no se sabe el baseline, el tamaño de muestra ni
  si son promedios de clientes reales o estimaciones).
- "Pentesting continuo autónomo 24/7" sobre la infraestructura del cliente implica que
  Zentris ejecuta técnicas ofensivas de forma recurrente y no supervisada en tiempo real;
  esto es sensible por diseño (riesgo de falsos positivos destructivos, necesidad de
  alcance/reglas de engagement bien definidas, ventanas de mantenimiento, etc.), pero no
  hay documentación pública sobre esos controles.

## 3. Seguridad del sitio público (resility.io)

Evidencia cruda en [`evidence/`](evidence/).

- **Hosting/CDN:** Cloudflare (`Server: cloudflare`, `cf-cache-status`, RUM beacon a
  `/cdn-cgi/rum`). El sitio es un build estático de **Astro** (`/_astro/*.css`) con al
  menos una ruta de API propia (`/api/contact`, detectada por análisis estático de
  `site.js`), probablemente desplegado en Cloudflare Pages/Workers.
- **Cabeceras de seguridad** (ver `evidence/headers-resility-io.txt`): HSTS con
  `preload` y `includeSubDomains`, `Content-Security-Policy` restrictiva (`default-src
  'self'`, sin `unsafe-inline` en scripts — usa un hash `sha256-...` para el único
  inline permitido), `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`,
  `Cross-Origin-Opener-Policy: same-origin`, `Permissions-Policy` restrictiva
  (geolocation, cámara, micrófono, pagos y USB deshabilitados). Esto es una
  configuración de cabeceras **por encima del promedio** para un sitio de marketing.
- **Anti-bot / anti-abuso:** Cloudflare Turnstile (`challenges.cloudflare.com`) más un
  captcha matemático simple en el formulario de contacto (doble capa, aunque la segunda
  es trivial).
- **Certificado SSL:** Sectigo PositiveSSL (sello de confianza visible en el footer).
- **Tracking/marketing:** Meta Pixel, Google Ads, LinkedIn Insight Tag, Google Tag
  Manager — permitidos explícitamente en la CSP como dominios de terceros.
- **`robots.txt`:** usa el mecanismo emergente de *Content Signals* (`Content-Signal:
  search=yes, ai-train=no, use=reference`) y bloquea explícitamente crawlers de
  entrenamiento de IA (GPTBot, ClaudeBot, Google-Extended, Bytespider, CCBot, etc.),
  permitiendo indexación de búsqueda normal. Señal de que la empresa es consciente de
  los riesgos de que su contenido se use para entrenar modelos de terceros.
- **Datos estructurados (JSON-LD):** declara `Organization`, `WebSite`, `FAQPage` y un
  `SoftwareApplication` para Zentris (`applicationCategory: SecurityApplication`,
  `operatingSystem: Cloud / Agentless`) — ver `evidence/structured-data.json`.

### Observaciones

- La postura de seguridad del sitio de marketing es sólida (CSP sin `unsafe-inline`,
  HSTS preload, Cloudflare delante). Es razonable para una empresa de ciberseguridad
  "predicar con el ejemplo" en su propio sitio público, como ellos mismos afirman.
  Pero esto solo certifica la higiene del sitio **de marketing**, no dice nada sobre la
  seguridad real de la plataforma Zentris (backend, multi-tenencia, aislamiento entre
  clientes, gestión de credenciales de integración con el SIEM/EDR de cada cliente),
  que no es observable desde afuera.
- No se detectaron fugas obvias en el sitio público (sin API keys expuestas en el JS
  estático revisado, sin endpoints de administración expuestos, sin información de
  versión de servidor más allá de "cloudflare").

## 4. Conclusión

Resility es un MSSP chileno establecido (10 años, ISO 27001, SOC 24/7) que está
posicionando su diferenciación competitiva en **Zentris**, una capa de agentes de IA
para triage, exposure management y pentesting continuo sobre el stack de seguridad que
el cliente ya tiene (agentless). La propuesta técnica es coherente con la tendencia de
"SOC aumentado por IA con humano en el loop" y está respaldada por integraciones reales
con herramientas conocidas (Wazuh, Tenable, Bitdefender) y por reconocimiento público
(ProChile). Las afirmaciones cuantitativas de rendimiento y el detalle interno de los
agentes (modelos usados, guardrails de las acciones ofensivas autónomas, aislamiento
multi-cliente) no son públicos y no pudieron verificarse con la información disponible
desde el exterior de su sitio.

## Evidencia

- [`evidence/headers-resility-io.txt`](evidence/headers-resility-io.txt) — cabeceras HTTP de `https://resility.io/`
- [`evidence/robots.txt`](evidence/robots.txt) — `robots.txt` público
- [`evidence/structured-data.json`](evidence/structured-data.json) — JSON-LD público de la home
- [`evidence/network-requests-sample.txt`](evidence/network-requests-sample.txt) — muestra de requests observados al cargar la home

---
*Análisis realizado por Joaquín Cuevas a partir de información pública, el 2026-11-20.*
