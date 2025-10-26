# Fase 2

## 2. Análisis de requisitos técnicos:

### Frontend (React): ¿Qué bibliotecas adicionales necesitaréis? (routing, state management, UI components, etc.)

#### Routing:

● Necesitamos enrutamiento cliente para navegación entre páginas (Login, Dashboard, Búsqueda de canales, Detalle de vídeo)

● Protección de rutas privadas (requieren autenticación)

#### State Management:

● Estado del servidor: Caché de datos del backend, sincronización automática, invalidación inteligente.

● Estado global del cliente: Información de autenticación (usuario, token), preferencias de UI.

● Estado local: Formularios, modales, estados de componentes individuales.

#### UI Components:

● Sistema de componentes reutilizables y accesibles

● Diseño responsive (tanto para desktop como para móvil)

● Componentes necesarios: Cards, Modals, Forms, Loading states, Empty states, Error boundaries

#### Gestión de formularios:

● Validación en tiempo real

● Manejo de errores del servidor

● Estados de loading durante submit

● Hacer pruebas de seguridad para evitar ataques XSS

#### HTTP Client:

● Interceptores para inyectar tokens JWT automáticamente

● Manejo centralizado de errores (401, 403, 500)

● Retry automático en fallos de red

### Backend (Node.js + Express): ¿Qué APIs o servicios externos integraréis? ¿Necesitáis autenticación? ¿Qué tipo?

| Servicio | Propósito | ¿Por qué es necesario? |
|----------|-----------|------------------------|
| YouTube RSS Feeds | Detectar nuevos vídeos publicados de un canal en específico. | No requiere API key, ilimitado, tiempo real. |
| youtube-transcript-plus | Extraer transcripciones de vídeos | Gratis, sin límites, no requiere autenticación. (en caso que no tenga trascripción se obtiene con un modelo de IA, detallado posteriormente) |
| OpenRouter API | Generar resúmenes con modelos de IA | Acceso a múltiples modelos LLM con un solo endpoint |
| YouTube Data API v3 | Búsqueda de canales, obtener metadatos | Obtener Channel ID desde username/handle |

#### Sistema de autenticación requerido:

**Tipo:** Autenticación basada en JWT (JSON Web Tokens)

**Justificación:**

● Stateless: escalable horizontalmente sin sesiones en servidor

● Compatible con arquitectura REST y un estándar muy utilizado.

● Tokens de corta duración para seguridad

● Refresh tokens para UX fluida

**Componentes necesarios:**

● Hash seguro de contraseñas (librerías como bcrypt)

● Generación y verificación de tokens

● Middleware de autenticación para rutas protegidas

● Manejo de refresh tokens

#### Sistema de procesamiento asíncrono:

**Requisito:** Procesamiento en background de vídeos (transcripción + resumen)

**Justificación:**

● Operaciones pueden tardar 30-120 segundos

● No bloquear respuestas HTTP

● Permitir reintentos automáticos en fallos

● Procesamiento concurrente de múltiples vídeos

**Solución:** Sistema de colas con workers y redis.

### Base de datos (MongoDB): Diseñad un esquema preliminar de las colecciones principales

#### USERS - Almacenar información de usuarios registrados

● Identificador único

● Email (login)

● Contraseña hasheada

● Nombre

● Fechas de creación y actualización

#### CHANNELS - Catálogo centralizado de canales de YouTube monitoreados

● Identificador único

● Username

● YouTube Channel ID (clave externa única)

● Información del canal (nombre, thumbnail, descripción)

● Última vez verificado (para polling)

● Contador de seguidores (cuántos usuarios siguen este canal)

● Estado activo/inactivo (si debe monitorearse)

● Fechas de creación y actualización

**Optimización clave:** Existe un solo documento por canal de YouTube, independientemente de cuántos usuarios lo sigan. Si 10 usuarios siguen "Joe Rogan", hay un único documento de ese canal con followerCount: 10.

#### USER_CHANNELS - Relación muchos a muchos (usuarios ⇔ canales)

● ID de usuario

● ID de canal

● Fecha de seguimiento

**Justificación:** Normalización para evitar duplicación. Un canal puede ser seguido por múltiples usuarios, y un usuario puede seguir múltiples canales. Esta tabla intermedia permite la relación sin duplicar información del canal.

**Optimización clave:**

● Si 10 usuarios siguen el mismo canal => hay 10 documentos en USER_CHANNELS apuntando al mismo canal

● NO se duplica el documento del canal

● NO se verifica el RSS del canal 10 veces (solo 1 vez cada x minutos)

#### VIDEOS - Almacenar vídeos procesados con resúmenes

● Identificador único

● YouTube Video ID (clave externa única)

● Referencia al canal (ID del canal al que pertenece)

● Metadata (título, thumbnail, duración, fecha publicación)

● URL del vídeo

● Estado de procesamiento (pending, processing, completed, failed)

● Transcripción completa (texto extraído del vídeo)

● Resumen generado por IA (contenido resumido)

● Puntos clave extraídos (3-5 bullets principales)

● Modelo de IA utilizado

● Tokens consumidos (para tracking de costes)

● Contador de visualizaciones (cuántos usuarios lo han visto)

● Información de errores (si falla el procesamiento)

● Fechas de creación y actualización

**Optimización clave:**

● Un vídeo se procesa una sola vez independientemente de cuántos usuarios sigan el canal

● Si 10 usuarios siguen "Joe Rogan" y sube un nuevo vídeo => se hace 1 transcripción + 1 resumen (no 10)

● Los 10 usuarios ven el mismo documento con el resumen

● Ahorro exponencial en llamadas a APIs de IA (el coste reducido mediante esta esta estrategia es exponencial, cuanto más usuarios hayas, más probabilidad hay que los usuarios pongan los mismo canales, reduciendo así la media de videos a analizar por usuario).

#### Índices necesarios:

● users.email (unique) - Búsqueda rápida por email de usuario para login

● channels.channelId (unique) - Búsqueda por YouTube Channel ID, evita duplicados

● channels.lastChecked - Para determinar qué canales verificar en cron jobs

● user_channels.userId + channelId (compound, unique) - Evita que un usuario siga el mismo canal dos veces

● user_channels.channelId - Para contar seguidores de un canal

● videos.videoId (unique) - Búsqueda por YouTube Video ID, evita duplicados

● videos.channelId + publishedAt (compound) - Para obtener feed de vídeos de un canal ordenados por fecha

● videos.status - Búsqueda de vídeos pendientes de procesar para workers

#### Flujo de optimización en la práctica:

**Escenario:** 10 usuarios siguen el canal "Joe Rogan Experience"

1. Base de datos contiene:
   - 10 documentos en USER_CHANNELS (uno por usuario)
   - 1 solo documento en CHANNELS para "Joe Rogan" con followerCount: 10
   - N documentos en VIDEOS (uno por cada vídeo publicado)

2. Cuando Joe Rogan publica un nuevo vídeo:
   - Cron job (no "cron" del sistema, sino bull con repeated jobs) verifica el RSS del canal 1 vez (no 10 veces)
   - Detecta nuevo vídeo => crea 1 documento en VIDEOS
   - Worker procesa 1 vez: extrae transcripción y genera resumen
   - Todos los usuarios ven el mismo resumen sin duplicación

3. Cuando un usuario consulta su feed:
   - Se obtienen los canales que sigue desde USER_CHANNELS
   - Se buscan los vídeos de esos canales en VIDEOS (ya procesados)
   - Sin regenerar resúmenes: se reutilizan los existentes

4. Cuando un usuario deja de seguir el canal:
   - Se elimina su documento en USER_CHANNELS
   - Se decrementa followerCount en CHANNELS
   - Si followerCount llega a 0 => el canal se marca como inactivo y deja de monitorearse

### Infraestructura: ¿Dónde desplegaréis? (VPS, Vercel, Render, Railway, etc.) ¿Necesitáis servicios cloud?

#### Estrategia de deployment: VPS único con contenedores Docker

Desplegaremos toda la aplicación en un VPS (Virtual Private Server) de DigitalOcean, aprovechando el crédito de $200 que proporciona GitHub Student Developer Pack. Esta aproximación nos permite tener control total sobre la infraestructura en una única máquina virtual.

#### Arquitectura de despliegue:

Todos los componentes del stack MERN + Redis se ejecutarán en el mismo VPS mediante contenedores Docker, gestionados con Docker Compose:

● Frontend (React): Contenedor con Nginx sirviendo build de producción

● Backend (Node.js + Express): Contenedor con la API REST

● MongoDB: Contenedor con base de datos

● Redis: Contenedor para colas Bull y cron jobs con bull.

● Nginx (reverse proxy): Contenedor para routing y SSL/TLS

#### Servicios cloud necesarios:

● ✅ VPS de DigitalOcean: Aloja todos los contenedores

● ✅ GitHub: Control de versiones y CI/CD con GitHub Actions

● ✅ Cloudflare (opcional): CDN y protección DDoS en capa gratuita

● ❌ NO necesitamos: Servicios de hosting especializados (Vercel, Railway, Render)

● ❌ NO necesitamos: MongoDB Atlas externo (usamos contenedor local)

● ❌ NO necesitamos: Redis Cloud externo (usamos contenedor local)

#### Red de contenedores Docker:

```
Internet => Cloudflare (CDN) => Nginx (reverse proxy, puerto 80/443)
- Frontend
- Backend (express:5000)
- MongoDB:27017
- Redis:6379
```

#### Pipeline de deployment:

1. Git push a main => GitHub Actions se activa
2. CI: Tests automatizados
3. CD: Build de imágenes Docker
4. SSH al VPS de DigitalOcean
5. docker-compose pull (descargar nuevas imágenes)
6. docker-compose up -d (reiniciar servicios)
7. Zero-downtime con health checks
