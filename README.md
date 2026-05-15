
# Nebula — Sistema de gestión y venta de boletas para teatro

> Plataforma distribuida desarrollada a medida para la gestión integral de un teatro. Reemplaza plataformas externas de venta de boletas, dando al teatro control total sobre sus ventas, sus clientes y sus datos. Construida con arquitectura de microservicios usando Laravel, ASP.NET Core, React PWA, MySQL, MongoDB y Docker.

---

## Tabla de contenidos

- [Contexto del proyecto](#contexto-del-proyecto)
- [Arquitectura general](#arquitectura-general)
- [CI/CD y disponibilidad](#cicd-y-disponibilidad)
- [Aplicaciones](#aplicaciones)
  - [Frontends](#frontends)
  - [Backends](#backends)
- [Bases de datos](#bases-de-datos)
- [Servicios de soporte](#servicios-de-soporte)
- [Comunicación entre servicios](#comunicación-entre-servicios)
- [Modelo de asientos](#modelo-de-asientos)
- [Flujo de compra online](#flujo-de-compra-online)
- [Flujo de validación en puerta](#flujo-de-validación-en-puerta)
- [Flujo de venta presencial](#flujo-de-venta-presencial)
- [Tecnologías utilizadas](#tecnologías-utilizadas)
- [Repositorios](#repositorios)
- [Equipo](#equipo)

---

## Contexto del proyecto

El teatro administraba la venta de boletas y el control de acceso a través de plataformas externas como TuBoleta, pagando comisiones elevadas por cada transacción. Con el crecimiento del volumen de eventos durante el año, este modelo dejó de ser rentable.

Nebula es el sistema propio del teatro: una plataforma web completa que permite vender boletas en línea y en taquilla física, gestionar todos los eventos, controlar el acceso de los asistentes el día del espectáculo, y mantener una comunicación directa con los clientes. El teatro retiene el 100% de sus ingresos y tiene control total sobre sus datos.

---

## Arquitectura general

El sistema sigue una arquitectura de **microservicios con múltiples frontends**. Cada frontend es una aplicación independiente que consume únicamente las APIs correspondientes a su función. La autenticación es transversal: un único Auth Service emite tokens JWT firmados que los demás servicios validan de forma local usando una clave secreta compartida, sin necesidad de consultar al Auth Service en cada petición.

![Infraestructura del proyecto Eventos](images/arquitectura_nebula_v2.svg)

## CI/CD y disponibilidad

El sistema implementa un pipeline de integración y despliegue continuo con **GitHub Actions**, y garantiza alta disponibilidad mediante **Docker con política de reinicio automático**.

### Pipeline CI/CD

```
Developer hace push / abre PR en GitHub
        ↓
GitHub Actions — CI:
    compila el proyecto
    ejecuta pruebas automatizadas
    verifica lint y calidad de código
        ↓
Si el CI pasa — GitHub Actions — CD:
    construye la imagen Docker actualizada
    despliega automáticamente al servidor
    reemplaza el contenedor anterior sin downtime
```

### Reinicio automático con Docker

Cada servicio corre en su propio contenedor Docker con la política `restart: always`. Si un servicio se cae por cualquier razón (error de memoria, excepción no controlada, reinicio del servidor), Docker lo detecta y lo levanta automáticamente sin intervención manual.

```yaml
services:
  auth-service:
    image: nebula-auth
    restart: always

  eventos-service:
    image: nebula-eventos
    restart: always

  validacion-service:
    image: nebula-validacion
    restart: always
```

---

## Aplicaciones

### Frontends

Los cuatro frontends son aplicaciones **React 18 PWA** (Progressive Web App). Al ser PWA, pueden instalarse en el dispositivo del usuario como una aplicación nativa, funcionan con conectividad intermitente gracias al Service Worker, y se acceden desde cualquier navegador sin necesidad de una tienda de aplicaciones. Esto es especialmente valioso para `tickets.` y `access.`, que se usan en condiciones de campo donde la señal puede ser inestable.

---

#### `nebula.andrescortes.dev` — Taquilla virtual del público

Interfaz pública para los asistentes del teatro. Es la cara principal del sistema en internet.

**Acceso y registro:**
- Cualquier persona puede ver la cartelera de eventos sin necesidad de cuenta.
- Para comprar una boleta, el sistema requiere registro.
- El registro puede hacerse con correo y contraseña, o mediante **login con Google** en un solo clic.

**Panel privado del cliente — cuatro secciones:**

*Mi Perfil*
El cliente puede actualizar su información personal (nombre, teléfono) y subir una foto de perfil. Las fotos se almacenan en AWS S3.

*Mis Compras y Mis Entradas*
Historial completo de boletas: eventos pasados y eventos futuros. Para los eventos futuros, el cliente puede ver y descargar su **código QR**, que es su boleta digital para presentar en la puerta del teatro.

*Mis Favoritos*
El cliente puede marcar eventos de su interés para no perderlos de vista. Si el administrador modifica la fecha, hora u otro dato relevante de un evento marcado como favorito, el sistema envía automáticamente un correo de notificación a todos los clientes que lo tienen guardado.

*Servicio al Cliente — PQRS*
Canal directo de comunicación con el teatro. El cliente puede enviar Preguntas, Quejas, Reclamos o Sugerencias, y desde esta misma sección puede consultar el estado de su solicitud y leer las respuestas del administrador.

| Funcionalidad | Backend que consume |
|---|---|
| Registro e inicio de sesión | Auth Service |
| Login con Google | Auth Service + Google OAuth |
| Listado y detalle de eventos | Eventos Service |
| Selección de asiento y compra | Eventos Service + Stripe |
| Ver boletas y QR | Eventos Service |
| Favoritos | Eventos Service + n8n |
| PQRS | Eventos Service |

---

#### `admin.nebula.andrescortes.dev` — Panel de administración

Interfaz privada exclusiva para el administrador del teatro. Requiere inicio de sesión con usuario y contraseña.

**Creación y gestión de eventos:**
El administrador crea todos los eventos manualmente desde este portal. Puede configurar el nombre, descripción, fecha del evento, rango de fechas de venta de boletas, precio de la boleta, póster o imagen del espectáculo, y cualquier información relevante del evento. También puede editar o cancelar eventos ya publicados.

**Atención al cliente — PQRS:**
Todas las solicitudes enviadas por los clientes desde `nebula.` llegan a este panel. El administrador puede leerlas, gestionarlas y enviarles respuestas directamente. El cliente ve la respuesta en tiempo real desde su perfil.

**Control de empleados y permisos:**
El administrador registra a los empleados del teatro y define a qué portales tiene acceso cada uno. Los permisos son independientes: un empleado puede tener acceso a `tickets.`, a `access.`, a ambos, o a ninguno. Tener acceso a uno no implica acceso al otro.

**Métricas y estadísticas:**
Panel de indicadores clave del negocio:
- Boletas vendidas por semana o por rango de fechas personalizado
- Número de usuarios registrados en la plataforma
- Índice de llenado u ocupación del teatro por evento
- Métricas adicionales a definir en conjunto con el cliente durante el desarrollo

| Funcionalidad | Backend que consume |
|---|---|
| Inicio de sesión | Auth Service |
| CRUD de eventos | Eventos Service |
| Subida de imágenes | Eventos Service + AWS S3 |
| Gestión de empleados y permisos | Auth Service |
| Gestión de PQRS | Eventos Service |
| Métricas | Eventos Service |

---

#### `tickets.nebula.andrescortes.dev` — Taquilla física

Interfaz para los empleados que atienden la venta presencial en la entrada del teatro. Solo accesible para empleados con permiso de taquilla asignado por el administrador.

**Proceso de venta presencial:**
El empleado ingresa el correo electrónico del cliente, selecciona el evento y el asiento disponible, y registra la venta. El cobro se realiza directamente en la ventanilla.

**Boleta física y digital simultáneas:**
Al confirmar la venta, el sistema imprime automáticamente la boleta física en la impresora de la taquilla. Al mismo tiempo, la boleta queda registrada en la cuenta del cliente en `nebula.`, disponible en formato digital con su código QR. Si el cliente no tiene cuenta, la boleta se envía al correo proporcionado.

| Funcionalidad | Backend que consume |
|---|---|
| Inicio de sesión | Auth Service |
| Búsqueda de cliente por correo | Auth Service + Eventos Service |
| Venta presencial y selección de asiento | Eventos Service |
| Impresión de boleta física | Eventos Service (integración impresora) |

---

#### `access.nebula.andrescortes.dev` — Control de acceso

Interfaz para los empleados que controlan la entrada al teatro el día del evento. Solo accesible para empleados con permiso de acceso asignado por el administrador.

**Escaneo de entradas:**
El empleado escanea el código QR del cliente, ya sea la boleta digital en el celular o la boleta física impresa. El sistema verifica la validez en tiempo real.

**Asignación de ubicación:**
Al aprobar la boleta, el sistema muestra inmediatamente el asiento exacto del cliente dentro del teatro (sección, fila y número de asiento), para que el empleado pueda guiarlo a su lugar.

**Sistema antifraude:**
El sistema está diseñado para detectar y rechazar:
- Boletas falsas o con QR inválido.
- Códigos QR duplicados: si alguien intenta ingresar con un QR que ya fue escaneado, el sistema emite una alerta inmediata y deniega el acceso.

| Funcionalidad | Backend que consume |
|---|---|
| Inicio de sesión | Auth Service |
| Escaneo y validación de QR | Validación Service |
| Visualización de asiento exacto | Validación Service |
| Detección de fraude y duplicados | Validación Service |

---

### Backends

#### Auth Service — `Laravel 11 + Passport + Socialite`

Servicio transversal de autenticación e identidad. Es el único servicio que emite tokens JWT y el único con acceso directo a la base de datos de usuarios. Los demás servicios validan los tokens de forma local con la clave secreta compartida, sin hacer peticiones adicionales al Auth Service en cada operación.

**Responsabilidades:**
- Registro de usuarios con correo y contraseña
- Login social con Google mediante Laravel Socialite
- Emisión de JWT con payload: `{ sub, email, rol, permisos_portal, exp }`
- Invalidación de tokens en logout
- Recuperación de contraseña por correo
- Creación de empleados por parte del administrador
- Gestión de permisos por portal independientes para cada empleado
- Activación y desactivación de cuentas

**Roles del sistema:** `cliente`, `empleado`, `admin`

---

#### Eventos Service — `ASP.NET Core 8`

Núcleo del negocio. Gestiona toda la lógica relacionada con eventos, asientos, boletas, compras, favoritos y PQRS. Los permisos dentro del servicio se determinan por el rol y los permisos incluidos en el JWT.

**Responsabilidades:**
- CRUD completo de eventos: nombre, descripción, fecha, rango de venta, precio de boleta, imagen
- Subida de imágenes de eventos y fotos de perfil a AWS S3
- Gestión del mapa de asientos del teatro por evento
- Bloqueo temporal de asientos durante el proceso de compra (reserva de 10 minutos)
- Compra de boletas con verificación de disponibilidad de asiento
- Integración con Stripe para procesamiento de pagos
- Generación de código QR único por boleta usando QRCoder
- Registro de favoritos y disparo de notificaciones cuando un evento favorito cambia
- Gestión de PQRS: creación por parte del cliente y respuesta por parte del admin
- Venta presencial desde `tickets.` con registro en la cuenta del cliente
- Métricas: boletas vendidas por período, usuarios registrados, índice de ocupación

---

#### Validación Service — `ASP.NET Core 8`

Servicio liviano y crítico encargado del control de acceso el día del evento. Opera sobre la BD Negocio compartida con Eventos Service. Está optimizado para respuesta inmediata bajo alta demanda de escaneos simultáneos.

**Responsabilidades:**
- Recepción del UUID extraído del código QR escaneado
- Verificación de existencia del ticket, estado y correspondencia con el evento en curso
- Detección de QR duplicados con alerta inmediata
- Detección de boletas falsas o inválidas
- Marcado del ticket como usado al primer escaneo exitoso
- Retorno del asiento exacto (sección, fila, número) para guía del empleado
- Respuesta de resultado: `VÁLIDO + asiento`, `YA USADO`, o `INVÁLIDO`

---

## Bases de datos

### BD Auth — `MySQL 8`

Propiedad exclusiva del Auth Service. Ningún otro servicio tiene acceso directo.

| Tabla | Descripción |
|---|---|
| `users` | Usuarios con email, contraseña hasheada, proveedor OAuth, foto de perfil, estado |
| `roles` | Roles del sistema: cliente, empleado, admin |
| `user_roles` | Relación entre usuarios y roles |
| `portal_permissions` | Permisos por portal para cada empleado (tickets, access) de forma independiente |
| `personal_access_tokens` | Tokens JWT emitidos por Passport |
| `password_reset_tokens` | Tokens temporales para recuperación de contraseña |

### BD Negocio — `MySQL 8`

Compartida entre Eventos Service y Validación Service.

| Tabla | Descripción |
|---|---|
| `eventos` | Nombre, descripción, fecha, precio de boleta, imagen URL, estado, rango de venta |
| `secciones` | Secciones físicas del teatro (platea, palco, balcón, etc.) |
| `filas` | Filas dentro de cada sección |
| `asientos` | Asientos físicos del teatro por fila. Estructura fija, no cambia por evento |
| `asiento_evento` | Estado de cada asiento para cada evento: libre, reservado_temp, ocupado |
| `boletas` | UUID único, QR generado, asiento asignado, comprador, precio al momento de compra, estado |
| `compras` | Registro de transacciones con referencia a Stripe, fecha, monto total |
| `favoritos` | Relación usuario → evento para la funcionalidad de favoritos |
| `pqrs` | Solicitudes de clientes con tipo, asunto, descripción y estado |
| `pqrs_respuestas` | Hilo de respuestas entre cliente y administrador por cada PQRS |

### MongoDB Atlas — `Logs del sistema`

Una sola base de datos `nebula_logs` con tres colecciones independientes, una por servicio.

| Colección | Descripción |
|---|---|
| `logs_auth` | Intentos de login, registros, cambios de contraseña, accesos denegados |
| `logs_eventos` | Compras, cancelaciones, modificaciones de eventos, cambios de asiento |
| `logs_validacion` | Escaneos realizados, accesos concedidos, rechazos, alertas de fraude |

---

## Servicios de soporte

### n8n — Orquestación de notificaciones

Herramienta de automatización de flujos que desacopla completamente la lógica de notificaciones del código de negocio. Los backends publican eventos de forma asíncrona y n8n los recoge, arma las plantillas y los envía a través del proveedor de correo configurado.

| Flujo | Disparador |
|---|---|
| Correo de bienvenida al nuevo usuario | `usuario_registrado` |
| Correo con QR tras compra online | `ticket_comprado` |
| Correo con QR en venta presencial sin cuenta | `ticket_vendido_presencial` |
| Notificación a favoritos cuando un evento cambia | `evento_actualizado` |
| Notificación masiva al cancelar un evento | `evento_cancelado` |

### Stripe — Pasarela de pagos

Integrado en Eventos Service para el procesamiento de pagos en línea. Opera en **modo test** durante el desarrollo, permitiendo simular transacciones completas con tarjetas de prueba predefinidas sin dinero real.

### AWS S3 — Almacenamiento de archivos

Bucket S3 para imágenes de portada de eventos y fotos de perfil de usuarios. Los frontends consumen los archivos directamente desde S3 sin pasar por la API, reduciendo la carga sobre los backends.

### Google OAuth 2.0 — Login social

Integrado en el Auth Service mediante **Laravel Socialite**. Permite a los usuarios de `nebula.` registrarse e iniciar sesión con su cuenta de Google en un solo clic.

---

## Comunicación entre servicios

| Tipo | Descripción |
|---|---|
| HTTP REST | Toda comunicación entre frontends y backends |
| JWT compartido | Auth Service emite el token; los demás servicios lo validan localmente con la misma clave secreta |
| Evento asíncrono | Backends publican eventos hacia n8n para correos y notificaciones |
| BD compartida | Eventos Service y Validación Service comparten la BD Negocio |
| CORS | Cada backend configura explícitamente qué dominios tienen permiso de consumirlo |

La clave secreta del JWT se configura como variable de entorno en cada servicio y nunca se incluye en el código fuente ni en el repositorio.

```
# Auth Service — .env de Laravel
JWT_SECRET=clave_secreta_compartida

# Eventos Service — appsettings.json de ASP.NET
"Jwt": {
  "Key": "clave_secreta_compartida",
  "Issuer": "auth.nebula.andrescortes.dev"
}

# Validación Service — appsettings.json de ASP.NET
"Jwt": {
  "Key": "clave_secreta_compartida",
  "Issuer": "auth.nebula.andrescortes.dev"
}
```

---

## Modelo de asientos

El teatro tiene una distribución física fija que no cambia entre eventos. Al crear un nuevo evento, el sistema genera automáticamente la disponibilidad de cada asiento para ese evento específico.

```
Teatro (estructura fija)
  └── Secciones (platea, palco, balcón...)
        └── Filas (A, B, C, D...)
              └── Asientos (1, 2, 3...)

Por cada evento creado, el sistema genera:
  asiento_evento
    ├── asiento_id      → referencia al asiento físico
    ├── evento_id       → referencia al evento
    ├── estado          → libre | reservado_temp | ocupado
    ├── boleta_id       → se asigna al confirmar la compra
    └── reservado_hasta → timestamp de expiración de reserva temporal
```

**Reserva temporal:** cuando un usuario selecciona un asiento, este pasa a `reservado_temp` por 10 minutos. Si completa el pago, queda `ocupado`. Si el tiempo expira sin pago, vuelve a `libre` automáticamente.

El precio de la boleta se guarda en la boleta en el momento de la compra, preservando la integridad histórica de los datos independientemente de cambios futuros en el precio del evento.

---

## Flujo de compra online

```
1. Usuario entra a nebula. y ve la cartelera
2. Selecciona un evento → debe iniciar sesión o registrarse
   (con correo/contraseña o con Google)
3. Auth Service valida credenciales y emite JWT
4. Usuario ve el mapa de asientos disponibles del evento
5. Selecciona su asiento → Eventos Service lo bloquea 10 minutos
6. Usuario completa el pago a través de Stripe
7. Eventos Service confirma el pago, genera UUID único y código QR
   Guarda el precio actual en la transacción
8. Eventos Service publica evento: ticket_comprado
9. n8n recoge el evento → envía correo con QR al usuario
10. El QR queda disponible también en "Mis Entradas" del perfil
```

---

## Flujo de validación en puerta

```
1. Empleado inicia sesión en access. (JWT con permiso: access = true)
2. Empleado escanea el QR del cliente
   (boleta digital en celular o boleta física impresa)
3. access. extrae el UUID del QR y lo envía al Validación Service
4. Validación Service consulta la BD Negocio:
   ¿Existe el ticket? ¿Está activo? ¿Corresponde al evento en curso?
   ¿Ya fue escaneado antes?

5a. VÁLIDO   → marca como usado, devuelve sección + fila + asiento
               pantalla verde, empleado guía al cliente a su lugar
5b. YA USADO → alerta inmediata, acceso denegado, pantalla roja
5c. INVÁLIDO → alerta inmediata, acceso denegado, pantalla roja
```

---

## Flujo de venta presencial

```
1. Empleado inicia sesión en tickets. (JWT con permiso: tickets = true)
2. Empleado ingresa el correo del cliente
3a. Cliente tiene cuenta → sistema la identifica automáticamente
3b. Cliente no tiene cuenta → se procede con solo el correo
4. Empleado selecciona el evento y el asiento disponible
5. Empleado registra la venta y cobra en ventanilla
6. Eventos Service genera la boleta con UUID y QR
7. Sistema imprime la boleta física automáticamente
8a. Cliente con cuenta → boleta aparece en su perfil en nebula.
8b. Cliente sin cuenta → boleta se envía al correo proporcionado vía n8n
```

---

## Tecnologías utilizadas

| Capa | Tecnología | Versión | Propósito |
|---|---|---|---|
| Frontend | React | 18 | Interfaces de usuario |
| Frontend | PWA | — | Instalable, offline-ready, Service Worker |
| Auth Service | Laravel | 11 | Framework backend PHP |
| Auth Service | Laravel Passport | — | Autenticación OAuth2 y JWT |
| Auth Service | Laravel Socialite | — | Login social con Google |
| Eventos Service | ASP.NET Core | 8 | Framework backend C# |
| Validación Service | ASP.NET Core | 8 | Framework backend C# |
| BD Auth | MySQL | 8 | Base de datos relacional de identidad |
| BD Negocio | MySQL | 8 | Base de datos relacional de negocio |
| Logs | MongoDB Atlas | — | Base de datos no relacional para logs |
| Imágenes y archivos | AWS S3 | — | Almacenamiento de archivos estáticos |
| Pagos | Stripe | — | Pasarela de pagos (modo test en desarrollo) |
| Notificaciones | n8n | — | Orquestación de flujos y correos |
| Login social | Google OAuth 2.0 | — | Autenticación con cuenta Google |
| QR | QRCoder (NuGet) | — | Generación de códigos QR por boleta |
| Contenedores | Docker + Compose | — | Contenerización y reinicio automático |
| CI/CD | GitHub Actions | — | Pipeline de integración y despliegue continuo |

---

## Repositorios

| Repositorio | Tecnología | Descripción |
|---|---|---|
| `events_infrastructure` | — | Documentación de arquitectura (este repositorio) |
| `events_auth` | Laravel 11 | Auth Service |
| `events_api` | ASP.NET Core 8 | Eventos Service |
| `events_validation` | ASP.NET Core 8 | Validación Service |
| `events_web` | React 18 PWA | Frontend nebula. |
| `events_admin` | React 18 PWA | Frontend admin. |
| `events_tickets` | React 18 PWA | Frontend tickets. |
| `events_access` | React 18 PWA | Frontend access. |

---

## Equipo

| Persona | Responsabilidad |
|---|---|
| P1 | Auth Service (Laravel) · Frontend login/registro (nebula.) |
| P2 | Eventos Service (ASP.NET) · Frontend nebula. |
| P3 | Eventos Service (ASP.NET) · Frontend admin. |
| P4 | Eventos Service (ASP.NET) · Frontend tickets. |
| P5 | Validación Service (ASP.NET) · Frontend access. |

---

*Proyecto académico — RIWI Cohorte 6 · 2026*