# Informe de Alteraciones de Django

## Proyecto: Sistema de Gestión — Gobierno de Jujuy
### Fecha: 31/05/2026 | Versión: 1.0
### Alcance: apps `sistema` + `administracion` + `servicios_web`
### (Excluye: `afiliaciones`, `geografico_genos`)

---

## PÁGINA 1 — RESUMEN GERENCIAL

| Categoría | Alteraciones | Objetivo común | Impacto |
|-----------|-------------|----------------|---------|
| Modelos (monkey-patching) | 5 | Extender User/Group | 🔴 Crítico |
| Settings no default | 12+ | Simplificar + hardening | 🟡 Alto |
| Auth y permisos | 13+ | Controlar menú lateral | 🔴 Crítico |
| Email system | 4 | Control SMTP, async | 🔴 Crítico |
| Admin site | 6 | Exponer campos custom | 🟡 Alto |
| Vistas (0 genéricas) | 72+ custom | Control total por vista | 🔴 Crítico |
| Templates + contexts | 3 | Exponer roles en HTML | 🟡 Alto |
| Error handling | 2 | Capturar errores global | 🟡 Medio |
| DevOps | 5 | Mantener simple | 🟡 Alto |

**Tres hallazgos críticos:**

1. **Monkey-patching (modificación de clases Django en tiempo de ejecución) + auth.Permission ignorado + contraseñas en base64 (codificación, NO cifrado)** atacan simultáneamente escalabilidad, seguridad y flexibilidad del sistema.
2. **Sin paginación ni genéricas (vistas pre-diseñadas de Django tipo ListView/CreateView)** → el sistema no escala. Una lista de afiliados devuelve 190K filas sin filtro, orden ni paginación.
3. **Sin LogEntry (registro de auditoría de Django) ni sistema de auditoría alternativo** → 500 usuarios operando sin trazabilidad. No se puede determinar quién accedió o modificó qué dato ni cuándo.

**Lo que dice la documentación base:** README.md y SETUP.md presentan estas alteraciones como funcionalidades del proyecto sin documentar costos ni alternativas Django nativas.

**Escenario hipotético evaluado:** Instituto de seguro con 190K afiliados, miles de transacciones/día, 500+ usuarios concurrentes. Las alteraciones generan riesgo de caída del sistema en hora pico, imposibilidad de auditar accesos a datos personales y sobrecarga del equipo de desarrollo.

→ Ver secciones 2-6 para análisis detallado.

---

## Índice

1. [Resumen Gerencial](#1-resumen-gerencial)
2. [Catálogo de Alteraciones](#2-catálogo-de-alteraciones)
   - 2.1 [Modelos](#21-modelos)
   - 2.2 [Settings](#22-settings)
   - 2.3 [Auth y Permisos](#23-auth-y-permisos)
   - 2.4 [Email System](#24-email-system)
   - 2.5 [Admin Site](#25-admin-site)
   - 2.6 [Vistas](#26-vistas)
   - 2.7 [Templates](#27-templates)
   - 2.8 [Error Handling](#28-error-handling)
   - 2.9 [DevOps](#29-devops)
3. [Impacto por Dimensión](#3-impacto-por-dimensión)
   - 3.1 [Seguridad](#31-seguridad)
   - 3.2 [Mantenibilidad](#32-mantenibilidad)
   - 3.3 [Escalabilidad](#33-escalabilidad)
   - 3.4 [Observabilidad](#34-observabilidad)
   - 3.5 [Portabilidad / DevOps](#35-portabilidad--devops)
   - 3.6 [Flexibilidad](#36-flexibilidad)
4. [Escenario Hipotético: Instituto de Seguro](#4-escenario-hipotético-instituto-de-seguro)
   - 4.1 [Perfil de carga](#41-perfil-de-carga)
   - 4.2 [Lo que falla primero](#42-lo-que-falla-primero)
   - 4.3 [Riesgos de seguridad sobre datos personales](#43-riesgos-de-seguridad-sobre-datos-personales)
   - 4.4 [Riesgos operativos diarios](#44-riesgos-operativos-diarios)
   - 4.5 [Costo de operación con equipo de desarrollo](#45-costo-de-operación-con-equipo-de-desarrollo)
   - 4.6 [Prioridad de corrección para producción](#46-prioridad-de-corrección-para-producción)
5. [Tabla Comparativa Django Nativo vs. Proyecto](#5-tabla-comparativa-django-nativo-vs-proyecto)
6. [Recomendaciones Priorizadas](#6-recomendaciones-priorizadas)
   - 6.1 [Urgente (día 1 en producción)](#61-urgente-día-1-en-producción)
   - 6.2 [Corto plazo (1-2 sprints)](#62-corto-plazo-1-2-sprints)
   - 6.3 [Mediano plazo (2-4 sprints)](#63-mediano-plazo-2-4-sprints)
   - 6.4 [Largo plazo (6+ meses)](#64-largo-plazo-6-meses)

---

## 2. Catálogo de Alteraciones

Cada alteración sigue este formato:

**ALTERACIÓN**
Objetivo original → Qué se buscaba resolver
  Documentado en   → Si README.md o SETUP.md lo mencionan
  Impacto resumido → Cómo afecta al proyecto en producción
  Evidencia        → Archivo:línea


---

### 2.1 Modelos

#### 2.1.1 Monkey-patching de auth.User (3 campos)

**ALTERACIÓN**
  Se agregan 3 campos a django.contrib.auth.User vía add_to_class() (método que agrega atributos a una clase en runtime):
  User.origen (FK→Origen), User.userPermission (FK→UserPermission),
  User.allowed_menus (M2M→MenuGrupo). Archivo: sistema/models.py:98-100

**OBJETIVO**
Necesitaban campos extra en User (origen del usuario, permisos
  heredados, menús permitidos). No definieron AUTH_USER_MODEL al
  iniciar el proyecto (difícil de agregar después). add_to_class()
  fue la solución rápida para no reescribir migraciones existentes.

**DOCUMENTADO EN BASE**
❌ README.md no menciona monkey-patching. Solo dice
  "Sistema de autenticacion con roles" como si fuera Django estándar.

**IMPACTO**
🔴 Crítico. Los campos no son descubribles (inspectdb no los ve,
  las migraciones no los reflejan). Cualquier upgrade de Django
  puede romperlos. El ORM genera migraciones inconsistentes.
  Las apps de terceros que interactúan con auth.User pueden fallar
  porque esperan un modelo estándar.
  Afecta a: seguridad (no hay trazabilidad de qué versión de User (modelo de usuario de Django)
  está en cada entorno), mantenibilidad (cada deploy requiere
  verificar que los campos sigan funcionando), escalabilidad
  (add_to_class se ejecuta en tiempo de importación, es código
  ejecutado fuera del control de migraciones).

**EVIDENCIA**
sistema/models.py:98-100 → add_to_class('origen', ...)
  sistema/models.py:99     → add_to_class('userPermission', ...)
  sistema/models.py:100    → add_to_class('allowed_menus', ...)


#### 2.1.2 Monkey-patching de auth.Group (2 campos)

**ALTERACIÓN**
Se agregan 2 campos a django.contrib.auth.models.Group vía
  add_to_class(): Group.home (CharField), Group.icon (CharField).
  Archivo: sistema/models.py:11-12

**OBJETIVO**
Agregar metadatos visuales al modelo Group para el menú lateral
  (sidebar). Cada Group representa un módulo del sistema, y se
  necesitaba un ícono y una URL de inicio por módulo.

**DOCUMENTADO EN BASE**
❌ No documentado. README.md menciona "permisos por rol" pero
  no explica que Group tiene campos inyectados.

**IMPACTO**
🟡 Alto. Altera el modelo Group de Django, que otras apps de
  contrib (admin, auth) esperan encontrar en su forma original.
  Las migraciones de auth no reflejan estos campos. Si se
  desregistra y registra un GroupAdmin propio, los campos
  aparecen sin configuración.
  GroupForm en administracion/forms.py:238 referencia estos campos
  monkey-patched como si fueran nativos — el formulario no funciona
  sin el add_to_class() previo.

**EVIDENCIA**
sistema/models.py:11 → add_to_class('home', ...)
  sistema/models.py:12 → add_to_class('icon', ...)
  administracion/forms.py:238 → GroupForm con fields=['name','home','icon']


#### 2.1.3 AUTH_USER_MODEL (modelo de usuario personalizado de Django) no definido

**ALTERACIÓN**
El archivo config/settings.py NO define AUTH_USER_MODEL.
  Usa el modelo auth.User de Django por defecto.

**OBJETIVO**
El proyecto comenzó sin la intención de reemplazar el User model.
  Cuando se necesitaron campos extra, ya había migraciones aplicadas
  sobre auth.User, y cambiar a AUTH_USER_MODEL después del primer
  migrate requiere migración compleja (proxy model + base de datos
  existente).

**DOCUMENTADO EN BASE**
❌ No documentado. README.md y SETUP.md tratan UserProfile como
  "perfil de usuario" sin mencionar la imposibilidad de cambiar
  el modelo de usuario raíz.

**IMPACTO**
🔴 Crítico. No tener AUTH_USER_MODEL propio impide:
  - Agregar campos directamente en User (forzando monkey-patching)
  - Usar GenericForeignKey desde User
  - Unificar la lógica de perfil en un solo modelo
  - Heredar funcionalidades de AbstractUser (permisos, fechas, etc.)
  La migración a un User model propio requiere downtime y debe
  planearse con mucho cuidado.
  Buena práctica Django: siempre definir AUTH_USER_MODEL al
  inicio del proyecto, incluso si se usa el mismo modelo por defecto.

**EVIDENCIA**
config/settings.py → búsqueda de AUTH_USER_MODEL = sin resultados
  sistema/models.py:98-100 → monkey-patching como workaround
  administracion/models.py:68 → UserProfile(OneToOneField(User))


#### 2.1.4 Migraciones gitignored

**ALTERACIÓN**
.gitignore excluye **/migrations/* (excepto __init__.py).
  Las migraciones se generan localmente y no se versionan.

**OBJETIVO**
Evitar conflictos de merge entre desarrolladores cuando varios
  trabajan en modelos simultáneamente. También evita migraciones
  huérfanas en el repositorio cuando se squash.

**DOCUMENTADO EN BASE**
✅ SETUP.md línea 60: python manage.py makemigrations ... es
  parte de los comandos de instalación. El hecho de que makemigrations
  sea necesario cada vez implica migraciones gitignored.

**IMPACTO**
🔴 Crítico. Garantiza diferencias de esquema entre entornos:
  - Dos clones frescos pueden generar migraciones diferentes
  - Desarrollo y producción pueden tener schemas distintos
  - Rollbacks requieren regenerar migraciones históricas
  - No se puede auditar cambios de esquema en el tiempo
  Alternativa Django nativa: versionar migraciones y usar
  --merge para resolver conflictos (práctica estándar).

**EVIDENCIA**
.gitignore → **/migrations/*
  SETUP.md → pasos 5-6 requieren makemigrations manual


#### 2.1.5 UserProfile como OneToOne (relación uno a uno) en vez de AbstractUser (clase base de Django para usuario personalizado)

**ALTERACIÓN**
Los campos de perfil (rol, ministerio, repartición, CUIL, etc.)
  se almacenan en un modelo separado UserProfile con OneToOneField
  a User, en vez de integrarlos en User vía AbstractUser.

**OBJETIVO**
Extender User sin modificar su modelo, ya que AUTH_USER_MODEL
  no se definió al inicio. OneToOne fue la opción disponible sin
  romper migraciones existentes.

**DOCUMENTADO EN BASE**
✅ README.md línea 29: "Perfil de usuario con ministerio,
  reparticion, CUIL y verificacion SUNICSU" — documentado como
  feature, no como workaround.

**IMPACTO**
🟡 Alto. OneToOne + monkey-patching crean dos fuentes de verdad
  para el usuario: campos en User (monkey-patched) y campos en
  UserProfile. Cada consulta de perfil requiere un JOIN adicional.
  No hay consistencia transaccional entre User y UserProfile.
  Con 500 usuarios y miles de requests/día, cada JOIN se multiplica.
  django.contrib.auth.admin.UserAdmin no muestra UserProfile
  automáticamente (hubo que crear CustomUserAdmin).

**EVIDENCIA**
administracion/models.py:68 → class UserProfile(models.Model)
  administracion/models.py:71 → OneToOneField(settings.AUTH_USER_MODEL)


---

### 2.2 Settings

#### 2.2.1 USE_TZ = False

**ALTERACIÓN**
USE_TZ = False en config/settings.py:148. Las fechas se almacenan
  sin zona horaria.

**OBJETIVO**
Simplificar el manejo de fechas: todos los usuarios están en la
  misma provincia (Jujuy, mismo huso horario). Evita la complejidad
  de conversiones UTC/local.

**DOCUMENTADO EN BASE**
❌ No documentado explícitamente en README.md o SETUP.md. Es un
  detalle de implementación.

**IMPACTO**
🟡 Alto. Sin timezone awareness:
  - Los timestamps son ambiguos durante cambios de horario
  - Cualquier integración futura con sistemas externos requiriá
    normalización manual de zonas horarias
  - Los logs y auditorías no tienen referencia horaria consistente
  - Si en el futuro se opera desde otras provincias, todos los datos
    históricos tienen zona horaria incorrecta
  Django nativo: USE_TZ = True y registrar en UTC. La conversión
  a hora local se hace en el template o serialización.

**EVIDENCIA**
config/settings.py:148 → USE_TZ = False


#### 2.2.2 USE_I18N = False

**ALTERACIÓN**
USE_I18N = False en config/settings.py:147. El sistema no soporta
  internacionalización.

**OBJETIVO**
Solo opera en Argentina, idioma español. No necesita traducciones
  ni soporte multilenguaje. Evita la complejidad de archivos .po/.mo.

**DOCUMENTADO EN BASE**
❌ No documentado explícitamente.

**IMPACTO**
🟢 Bajo. Para un sistema del Gobierno de Jujuy que opera solo en
  la provincia, i18n no es prioritario. Es una decisión aceptable
  siempre que se documente para futuros mantenedores.

**EVIDENCIA**
config/settings.py:147 → USE_I18N = False


#### 2.2.3 From credenciales import *

**ALTERACIÓN**
Al final de config/settings.py, se ejecuta from config.credenciales import *
  para sobreescribir settings con valores sensibles (SECRET_KEY, DATABASES,
  tokens API, etc.).

**OBJETIVO**
Separar la configuración sensible (tokens, contraseñas) del código
  versionado, sin usar variables de entorno. El equipo no tenía
  experiencia con env vars o secret managers.

**DOCUMENTADO EN BASE**
✅ SETUP.md línea 37-56: instrucciones detalladas para crear y
  editar credenciales.py. README.md línea 69: "Las URLs y tokens
  de las APIs se configuran en config/credenciales.py".

**IMPACTO**
🔴 Crítico. El patrón from X import * sobreescribe settings en
  tiempo de importación:
  - Cualquier error de sintaxis en credenciales.py rompe todo el
    sistema (no hay validación al arrancar)
  - Si credenciales.py falta, el sistema arranca con defaults
    inseguros (DEBUG=True, SECRET_KEY='', etc.)
  - No hay separación por entorno (dev/staging/prod comparten
    la misma sobreescritura)
  - Las herramientas de Django (check, diffsettings) no reflejan
    los valores sobreescritos
  Alternativa nativa: python-decouple o django-environ con
  archivos .env separados por entorno + validación de
  variables requeridas al startup.

**EVIDENCIA**
config/settings.py:261-264 → from config.credenciales import *


#### 2.2.4 CSRF (Cross-Site Request Forgery, ataque que ejecuta acciones no autorizadas en nombre de un usuario autenticado) — CSRF_HTTPONLY = True

**ALTERACIÓN**
CSRF_COOKIE_HTTPONLY = True en config/settings.py:121.

**OBJETIVO**
Seguridad: evitar que JavaScript acceda al token CSRF vía
  document.cookie.

**DOCUMENTADO EN BASE**
❌ No documentado. Es un detalle de hardening.

**IMPACTO**
🔴 Crítico. Django requiere que el token CSRF esté disponible
  mediante JavaScript para funcionar correctamente en formularios
  que usan fetch() o XMLHttpRequest. Con HttpOnly activo:
  - Los formularios enviados vía AJAX fallan por CSRF
  - El template tag {% csrf_token %} en HTML sigue funcionando
    (el token está en el DOM), pero cualquier request AJAX que
    necesite leer la cookie CSRF no puede hacerlo
  - django.middleware.csrf.CsrfViewMiddleware usa la cookie y el
    token del cuerpo/form; si no se pueden igualar, rechaza el
    request
  Fue implementado con intención de seguridad pero rompe el
  mecanismo de protección CSRF de Django.
  NOTA: En Django 5.1, CSRF_HTTPONLY por defecto ya es True
  para protección contra robo de token vía XSS. Sin embargo,
  CSRF funciona porque el token se pasa en el DOM (no solo en
  cookie). Si el proyecto usa fetch() leyendo la cookie, falla.
  Si usa {% csrf_token %} en el form HTML, funciona.
  Correción necesaria: verificar que todas las requests POST
  incluyan el token CSRF desde el DOM ({% csrf_token %}) y no
  desde la cookie.

**EVIDENCIA**
config/settings.py:121 → CSRF_COOKIE_HTTPONLY = True


#### 2.2.5 Session: SESSION_COOKIE_AGE = 28800 (8h)

**ALTERACIÓN**
SESSION_COOKIE_AGE = 28800 segundos en vez de 1209600 (2 semanas).
  SESSION_EXPIRE_AT_BROWSER_CLOSE = True.

**OBJETIVO**
Hardening de sesiones: sesiones más cortas que expiran al cerrar
  el navegador. Reduce la ventana de ataque si alguien deja la
  sesión abierta.

**DOCUMENTADO EN BASE**
❌ No documentado en README/SETUP.

**IMPACTO**
🟢 Beneficioso. Buena práctica de seguridad para sistemas
  gubernamentales. Mantener.

**EVIDENCIA**
config/settings.py:115-118 → SESSION_COOKIE_AGE, _EXPIRE_AT_BROWSER_CLOSE, etc.


#### 2.2.6 PASSWORD_RESET_TIMEOUT = 3600 (1 hora)

**ALTERACIÓN**
Tiempo de expiración del token de reset de password en 1 hora
  en vez de 72 horas (default Django).

**OBJETIVO**
Seguridad: minimizar la ventana de ataque de un token de reset.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟢 Beneficioso. Práctica de seguridad recomendada. Mantener.

**EVIDENCIA**
config/settings.py:112 → PASSWORD_RESET_TIMEOUT = 3600


#### 2.2.7 RECAPTCHA settings custom

**ALTERACIÓN**
Settings personalizados: RECAPTCHA_ENABLED, RECAPTCHA_SITE_KEY,
  RECAPTCHA_SECRET_KEY. No usan una app Django de captcha existente
  ni las settings estándar del ecosistema.

**OBJETIVO**
Implementar reCAPTCHA en login sin depender de apps de terceros.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. La implementación custom en sistema/views.py tiene un
  fail-open (modo que permite el acceso si el componente de seguridad falla): si Google reCAPTCHA no responde (timeout), el login
  procede sin verificación. Esto invalida el propósito del captcha.
  Con 500+ usuarios y un sistema expuesto a internet, un ataque de
  brute force puede eludir el captcha simplemente saturando Google.

**EVIDENCIA**
config/settings.py:169-171 → RECAPTCHA_* settings
  sistema/views.py:527 → LoginConCaptchaView (fail-open)


#### 2.2.8 SECURE_SSL_REDIRECT = False

**ALTERACIÓN**
SECURE_SSL_REDIRECT = False y SECURE_HSTS_SECONDS = 0.

**OBJETIVO**
Desarrollo local sin HTTPS. En producción se sobreescribe vía
  credenciales.py (según comentarios en settings.py).

**DOCUMENTADO EN BASE**
✅ SETUP.md línea 117: "Descomentar bloque SEGURIDAD HTTPS" en
  producción. Documentado como paso de deploy.

**IMPACTO**
🟡 Alto riesgo si se olvida activar en producción. El comentario
  en settings.py y la instrucción en SETUP.md mitigan parcialmente,
  pero no hay validación automática. Un deploy sin activar HTTPS
  expone todas las contraseñas en texto plano.
  Alternativa: forzar HTTPS mediante middleware condicional o
  chequear al startup.

**EVIDENCIA**
config/settings.py:132-137 → SECURE_SSL_REDIRECT, HSTS settings


#### 2.2.9 DEFAULT_AUTO_FIELD = BigAutoField

**ALTERACIÓN**
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

**OBJETIVO**
Usar enteros de 64 bits para claves primarias, evitando el
  límite de 2.1B registros por tabla.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟢 Beneficioso. Práctica recomendada para sistemas con +190K registros.
  Mantener.

**EVIDENCIA**
config/settings.py:166 → DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"


#### 2.2.10 LOGIN_URL / LOGIN_REDIRECT_URL / LOGOUT_REDIRECT_URL custom

**ALTERACIÓN**
LOGIN_URL = "/login/", LOGIN_REDIRECT_URL = "/",
  LOGOUT_REDIRECT_URL = "/".

**OBJETIVO**
Rutas personalizadas que no usan el patrón /accounts/ de Django.

**DOCUMENTADO EN BASE**
❌ No documentado explícitamente. README.md no menciona URLs de login.

**IMPACTO**
🟢 Bajo. Las URLs de auth son custom pero funcionalmente
  equivalentes. No generan riesgo. El único costo es que difieren
  de la convención Django, lo que puede confundir a nuevos
  desarrolladores.

**EVIDENCIA**
config/settings.py:63-65 → LOGIN_URL, LOGIN_REDIRECT_URL, LOGOUT_REDIRECT_URL
  config/urls.py:20 → path('login/', ...)


#### 2.2.11 Logging custom (RotatingFileHandler)

**ALTERACIÓN**
Sistema de logging con RotatingFileHandler a logs/security.log y
  logs/app.log, con loggers específicos (django.security, django.request,
  administracion, servicios_web).

**OBJETIVO**
Tener logs persistentes y rotados para producción, no solo la
  consola de desarrollo. Loggear específicamente seguridad y requests.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟢 Beneficioso. Buena implementación de logging. Mantener.

**EVIDENCIA**
config/settings.py:180-234 → LOGGING


---

### 2.3 Auth y Permisos

#### 2.3.1 auth.Permission completamente ignorado

**ALTERACIÓN**
Django genera 4 permisos por modelo (add_xxx, change_xxx, delete_xxx,
  view_xxx). El proyecto no los usa: 0 ocurrencias de @permission_required,
  has_perm(), {% perms %}, PermissionRequiredMixin en todo el código.

**OBJETIVO**
El sistema de permisos de Django fue reemplazado por un mecanismo
  propio (roles + MenuGrupo.roles_permitidos) cuyo objetivo original
  era controlar el menú lateral. Django no permite filtrar items de
  menú por permiso sin código adicional, y el equipo optó por un
  sistema custom que creció hasta abarcar toda la autorización.

**DOCUMENTADO EN BASE**
❌ No documentado. README.md dice "permisos por rol" sin aclarar
  que los permisos de Django están inertes.

**IMPACTO**
🔴 Crítico. El sistema de autorización paralelo (cadena Group →
  MenuGrupo.roles_permitidos → UserProfile.rol) tiene varias
  limitaciones graves:
  - Granularidad binaria: solo "ve el menú o no lo ve", no hay
    permisos de crear/editar/eliminar
  - 5 roles fijos en código, no configurables desde admin
  - CSV split como mecanismo de matching (frágil y lento)
  - 75+ vistas con chequeo manual de rol (alta superficie de error)
  - Los permisos reales de Django (auth.Permission) están en la BD
    ocupando espacio sin ser usados
  El Django admin (sistema/admin.py) registra CustomUserAdmin que
  hereda de UserAdmin con filter_horizontal = ('groups', 'user_permissions'),
  permitiendo asignar permisos que NUNCA se verifican — falsa sensación
  de control.

**EVIDENCIA**
grep @permission_required → 0 resultados
  grep has_perm → 0 resultados
  grep {% perms %} → 0 resultados
  grep PermissionRequiredMixin → 0 resultados
  sistema/admin.py:22-34 → CustomUserAdmin(UserAdmin) hereda filter_horizontal


#### 2.3.2 UserProfile.Rol: 5 roles fijos en TextChoices

**ALTERACIÓN**
administracion/models.py:71-76 define Rol como TextChoices con
  5 valores fijos: superadmin, administrador, supervisor, operador,
  consulta. No es extensible sin modificar código.

**OBJETIVO**
Tener un sistema de roles simple y predecible para el personal
  del gobierno de Jujuy, donde los perfiles están bien definidos.

**DOCUMENTADO EN BASE**
✅ README.md línea 72-80: tabla de roles documentada como
  feature del proyecto.

**IMPACTO**
🟡 Alto. 5 roles fijos significan:
  - No se puede crear un rol nuevo sin deploy (cambiar TextChoices
    requiere makemigrations + migrate + deploy)
  - No se pueden asignar permisos combinados (ej: "ve afiliaciones
    pero no puede editar")
  - TextChoices es razonable para MVP pero no escala a un sistema
    con 500+ usuarios de distintos perfiles organizacionales
  Django nativo: auth.Group + auth.Permission ofrecen roles
  dinámicos sin tocar código.

**EVIDENCIA**
administracion/models.py:71-76 → class Rol(models.TextChoices)


#### 2.3.3 MenuGrupo.roles_permitidos como CSV (str.split)

**ALTERACIÓN**
sistema/models.py:53-63: MenuGrupo.usuario_tiene_acceso() parsea
  roles_permitidos con str.split(',') y compara contra el string
  del rol del usuario.

**OBJETIVO**
Asociar ítems de menú con roles sin crear una tabla M2M. Rápido
  de implementar, simple de entender.

**DOCUMENTADO EN BASE**
❌ No documentado. Es un detalle de implementación interno.

**IMPACTO**
🔴 Crítico. CSV como mecanismo de integridad referencial:
  - No hay validación de que los strings coincidan con roles
    existentes (un typo en roles_permitidos = "supervisr" falla
    silenciosamente)
  - No hay FK, no hay M2M, no hay integridad referencial
  - Si se renombra un rol, hay que buscar y reemplazar en N
    registros de MenuGrupo
  - No se puede consultar "¿qué menús ve este rol?" sin split()
    de todos los registros
  - str.split() en cada request es overhead innecesario
  Con 500 usuarios concurrentes, cada uno haciendo múltiples
  requests, el split se ejecuta cientos de miles de veces por día.
  Django nativo: M2M a Group o a Permission con integridad
  referencial y consultas eficientes.

**EVIDENCIA**
sistema/models.py:53-63 → usuario_tiene_acceso() con split(',')
  sistema/models.py → roles_permitidos = CharField(max_length=200)


#### 2.3.4 allowed_menus M2M (relación muchos a muchos) monkey-patched

**ALTERACIÓN**
User.allowed_menus es un M2M a MenuGrupo inyectado via add_to_class().
  Se consulta en cada request en sistema/context_processors.py:9-35.

**OBJETIVO**
Permitir que un administrador asigne menús individuales a un
  usuario específico, complementando el filtro por rol.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. Cada request ejecuta:
  - user.allowed_menus.all() → query M2M
  - user.groups.all() → query M2M a auth.Group
  - user.userprofile → query OneToOne
  - user_profile_context → otro query a UserProfile
  4+ queries adicionales por request solo para saber qué menú
  mostrar. Con 500 usuarios activos y ~10 requests/minuto cada uno,
  esto agrega miles de queries innecesarias por hora.
  Django nativo: {% perms %} ya está en el context processors
  por defecto, no agrega queries extra, y usa las tablas de
  Permission que Django ya mantiene.

**EVIDENCIA**
sistema/context_processors.py:9-35 → user_allowed_menus
  sistema/models.py:98-100 → add_to_class('allowed_menus', ...)


#### 2.3.5 LoginConCaptchaView con reCAPTCHA fail-open

**ALTERACIÓN**
sistema/views.py:527 define LoginConCaptchaView(LoginView) con
  validación de reCAPTCHA en form_valid(). Si la verificación con
  Google falla (timeout, error de red), el login procede sin captcha.

**OBJETIVO**
Agregar protección contra bots al login sin usar apps de terceros.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🔴 Crítico. Fail-open significa que un atacante puede eludir el
  captcha saturando la API de Google o esperando un timeout de red.
  Para un sistema con 500+ usuarios expuesto a internet, esto
  permite ataques de fuerza bruta sin restricción.
  La implementación correcta es fail-closed: si reCAPTCHA no responde,
  denegar el login y loguear el error para monitoreo.

**EVIDENCIA**
sistema/views.py:527 → LoginConCaptchaView
  sistema/views.py:540-550 → validación captcha sin raise en catch


#### 2.3.6 Logout por GET (método HTTP que envía datos en la URL, no debería modificar estado)

**ALTERACIÓN**
sistema/views.py:45 define logout_view(View) que acepta requests GET
  para cerrar sesión. Django requiere POST para logout y así evitar
  CSRF.

**OBJETIVO**
Simplificar la experiencia de usuario: un link (GET) cierra sesión.
  No requiere formulario ni confirmación.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🔴 Crítico. Cerrar sesión por GET es vulnerable a CSRF:
  - Un atacante puede incrustar <img src="/logout_view/"> en una
    página maliciosa y el usuario cierra sesión sin saberlo
  - Si la sesión no expira y el usuario no nota que cerró sesión,
    puede haber pérdida de datos no guardados
  - Para un sistema gubernamental, esto es una violación de
    OWASP Top 10 (A1: Broken Access Control)
  Django nativo: LogoutView requiere POST por defecto. Se puede
  customizar pero siempre con método POST.

**EVIDENCIA**
sistema/views.py:45 → class logout_view(View) con GET
  config/urls.py:23 → path('logout_view/', ...)


#### 2.3.7 Señales (signals: mecanismo de Django para reaccionar a eventos como login/logout) de auth sin suscriptores

**ALTERACIÓN**
Las señales user_logged_in, user_logged_out, user_login_failed
  de django.contrib.auth no tienen ningún suscriptor en el proyecto.

**OBJETIVO**
No se identificó una necesidad de reaccionar a estos eventos de
  autenticación durante el desarrollo inicial.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. Sin suscriptores en estas señales:
  - No se loguean intentos de login fallidos (detección de brute force)
  - No se registra la última conexión del usuario
  - No se puede enviar notificación de login desde ubicación no
    reconocida
  - No se puede invalidar sesiones anteriores al login
  Para un sistema con 500+ usuarios y datos sensibles, la falta
  de monitoreo de autenticación es un riesgo de seguridad.
  Django nativo: conectar estas señales es trivial (2 líneas cada una).

**EVIDENCIA**
Señales verificadas: 0 suscriptores para las 3 señales de auth.


#### 2.3.8 Sin AuthenticationBackends definidos

**ALTERACIÓN**
No hay AUTHENTICATION_BACKENDS en settings.py. Se usa el backend
  por defecto (ModelBackend).

**OBJETIVO**
Simplicidad: solo autenticación local contra la BD de usuarios.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. Usar solo ModelBackend impide:
  - Integrar LDAP/AD para autenticación corporativa (gobierno)
  - SSO (Single Sign-On) entre sistemas del gobierno
  - Autenticación multifactor
  - Login con token API para integraciones
  Para 500+ usuarios de un gobierno provincial, la integración
  con LDAP del ministerio sería lo esperable.

**EVIDENCIA**
grep AUTHENTICATION_BACKENDS → 0 resultados


#### 2.3.9 AdminRequiredMixin + decorador custom

**ALTERACIÓN**
administracion/views.py:24 define AdminRequiredMixin que verifica
  profile.es_admin en vez de usar @permission_required o
  PermissionRequiredMixin.

**OBJETIVO**
Control de acceso por rol en vistas, siguiendo el mismo patrón
  del sistema custom de roles.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. Cada vista con AdminRequiredMixin tiene un chequeo de
  roles que depende de user_profile_context o de la propiedad
  es_admin en UserProfile. Esto duplica la lógica de autorización
  y aumenta la superficie de error.
  Con ~72 vistas, cada una con su chequeo de acceso, la probabilidad
  de que al menos una tenga un bypass es alta.
  Django nativo: @permission_required('app.view_model') centraliza
  y unifica el control de acceso.

**EVIDENCIA**
administracion/views.py:24 → class AdminRequiredMixin


#### 2.3.10 GroupForm con campos monkey-patched

**ALTERACIÓN**
administracion/forms.py:238 define GroupForm con los campos
  'name', 'home', 'icon' — donde 'home' e 'icon' son campos
  inyectados con add_to_class(), no originales de auth.Group.

**OBJETIVO**
Poder editar los metadatos visuales de Group (ícono, página
  de inicio) desde el formulario de grupos.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. El formulario depende de que los campos existan en
  runtime. Si se mueve el add_to_class() o se ejecuta en orden
  incorrecto, el formulario falla con AttributeError.

**EVIDENCIA**
administracion/forms.py:238 → class GroupForm(forms.ModelForm)


#### 2.3.11 user_permissions expuesto en admin pero no verificado

**ALTERACIÓN**
CustomUserAdmin hereda de UserAdmin, que incluye
  filter_horizontal = ('groups', 'user_permissions'). Esto permite
  asignar permisos a usuarios desde el admin, pero ninguna vista
  los verifica.

**OBJETIVO**
No fue intencional. Es herencia directa de UserAdmin sin
  eliminar el campo user_permissions.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🔴 Crítico. Falsa sensación de control:
  - Un administrador puede asignar permisos (add_xxx, change_xxx)
    pensando que funcionan
  - Los permisos nunca se verifican en vistas ni en templates
  - La seguridad real depende del sistema paralelo de roles
  - Auditoría descubre permisos asignados pero inefectivos
  Corregir: eliminar user_permissions de los fieldsets de
  CustomUserAdmin o implementar la verificación real.

**EVIDENCIA**
sistema/admin.py:22-34 → CustomUserAdmin(UserAdmin)


#### 2.3.12 user_profile_context: booleanos en templates en vez de {% perms %}

**ALTERACIÓN**
sistema/context_processors.py expone variables como es_superadmin,
  es_admin, es_supervisor, es_operador, es_consulta, solo_lectura
  en todas las templates. Reemplazan a {% perms %}.

**OBJETIVO**
Tener disponibles los roles del usuario en templates sin usar
  el sistema de permisos de Django.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. Cada template recibe variables inyectadas que duplican
  la información que Django ya provee vía {{ perms }}. No hay
  granularidad: un template sabe si el usuario es "administrador"
  pero no si tiene permiso para "crear solicitudes".
  Con ~72 vistas, todas usan este context processor, que ejecuta
  queries adicionales en cada request.
  Django nativo: {{ perms.app_label.view_model }} sin queries extra.

**EVIDENCIA**
sistema/context_processors.py → user_profile_context


#### 2.3.13 Auth forms de Django no utilizados

**ALTERACIÓN**
No se usan AuthenticationForm, PasswordChangeForm, PasswordResetForm,
  SetPasswordForm, UserCreationForm, UserChangeForm. Todos reemplazados
  por implementaciones custom.

**OBJETIVO**
Personalizar la apariencia y comportamiento de los formularios
  de auth.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. Cada formulario custom requiere mantenimiento separado.
  Las validaciones de seguridad de Django (password strength, token
  expiry, rate limiting) pueden no estar replicadas.

**EVIDENCIA**
administracion/forms.py → userForm custom, no UserCreationForm
  grep AuthenticationForm → solo en import de LoginConCaptchaView


---

### 2.4 Email System

#### 2.4.1 smtplib directo en vez de EMAIL_BACKEND

**ALTERACIÓN**
El envío de email se hace con smtplib directamente
  (administracion/services/email_service.py:6) y también en
  administracion/views.py:1387. No usa Django send_mail() ni
  EMAIL_BACKEND.

**OBJETIVO**
Tener control total sobre la conexión SMTP (TLS/SSL, autenticación,
  adjuntos). Django send_mail() no exponía suficiente control en
  versiones anteriores.

**DOCUMENTADO EN BASE**
✅ README.md línea 81-113: documenta EmailService con ejemplos
  de uso como si fuera un feature estándar, sin mencionar que es
  smtplib directo.

**IMPACTO**
🔴 Crítico. smtplib directo significa:
  - Sin cola de tareas: el envío bloquea el request HTTP
    (cada email tarda 0.5-2 segundos)
  - Sin reintentos automáticos: si SMTP falla, el usuario ve error
  - Sin integración con el sistema de logging de Django
  - Sin soporte para backends alternativos (console, file, memory)
  - Sin gestión de conexiones (pooling, timeout configurables)
  Con miles de transacciones/día y notificaciones por email, esto
  es un cuello de botella severo.

**EVIDENCIA**
administracion/services/email_service.py:6 → import smtplib
  administracion/views.py:1387 → import smtplib (test de conexión)
  administracion/services/email_service.py:18 → class EmailService


#### 2.4.2 Contraseñas de email en base64 (codificación que convierte datos binarios a texto, NO es cifrado)

**ALTERACIÓN**
administracion/models.py:293-324: ConfiguracionCorreo.email_password
  se almacena con codificación base64 (set_password/get_password).

**OBJETIVO**
"Cifrar" la contraseña SMTP almacenada en la BD, asumiendo que
  base64 es suficiente protección.

**DOCUMENTADO EN BASE**
❌ No documentado. README.md menciona configuración de email
  pero no cómo se almacena la contraseña.

**IMPACTO**
🔴 Crítico. Base64 NO es cifrado, es codificación:
  - Cualquier atacante con acceso a la BD (SQLi, backup robado,
    dump) puede decodificar la contraseña en 1 milisegundo
  - base64.b64decode('...') → contraseña en texto plano
  - La contraseña SMTP da acceso al servidor de correo, desde
    donde se pueden enviar emails fraudulentos en nombre del
    gobierno
  - get_password() línea 324 tiene fallback a texto plano por
    compatibilidad con datos legacy
  Solución: usar django.core.signing o un cifrado simétrico real
  (Fernet de cryptography) con clave gestionada externamente.

**EVIDENCIA**
administracion/models.py:311-324 → set_password/get_password con base64
  administracion/models.py:324 → fallback a texto plano


#### 2.4.3 Daemon thread para envío async de email

**ALTERACIÓN**
administracion/signals.py:80-83: cuando se crea un MensajeSistema,
  se dispara un daemon thread (threading.Thread(daemon=True)) para
  enviar el email en segundo plano.

**OBJETIVO**
Enviar notificaciones por email sin bloquear el request HTTP y
  sin instalar infraestructura de colas (Celery, RabbitMQ, Redis).

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🔴 Crítico. Daemon thread sin monitoreo:
  - Si el thread falla (excepción), el error se pierde silenciosamente
  - Si el servidor se reinicia, los threads en ejecución se pierden
  - No hay reintentos ni persistencia de la cola
  - Con miles de transacciones/día, las notificaciones perdidas
    son un problema operativo grave
  - No hay límite de threads: bajo carga, se pueden crear cientos
    de threads simultáneos, agotando recursos del sistema
  Solución a corto plazo: django-qsqueue o django-rq. La solución
  definitiva es Celery + Redis/RabbitMQ.

**EVIDENCIA**
administracion/signals.py:80-83 → threading.Thread(daemon=True)


#### 2.4.4 Sin EMAIL_* settings definidos

**ALTERACIÓN**
No existen EMAIL_BACKEND, EMAIL_HOST, EMAIL_PORT, EMAIL_USE_TLS,
  EMAIL_HOST_USER, EMAIL_HOST_PASSWORD en settings.py. Toda la
  configuración SMTP está en ConfiguracionCorreo (tabla BD).

**OBJETIVO**
Configurar el email desde la interfaz de administración (Parametros
  > Email) sin tocar código ni archivos de configuración.

**DOCUMENTADO EN BASE**
✅ README.md línea 83: "Se configura el proveedor SMTP desde
  Parametros > Email (sin tocar codigo)".

**IMPACTO**
🟡 Medio. Almacenar la configuración SMTP en BD:
  - La contraseña está en la BD (con el problema de base64)
  - Cambiar de servidor SMTP es una query SQL, no un cambio de
    configuración versionado
  - No se puede tener diferentes configuraciones por entorno
    (dev usa Mailtrap, prod usa SMTP real)
  Ventaja: los administradores no técnicos pueden cambiar el SMTP
  sin intervención de desarrollo.

**EVIDENCIA**
grep EMAIL_BACKEND → 0 resultados
  grep EMAIL_HOST → 0 resultados


---

### 2.5 Admin Site

#### 2.5.1 URL del admin ofuscada (/admin/panel_administrativo/)

**ALTERACIÓN**
La URL del admin es /admin/panel_administrativo/ en vez de /admin/.

**OBJETIVO**
Seguridad por oscuridad: ocultar la URL del admin para evitar
  ataques automatizados.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. La seguridad por oscuridad no es una medida de
  seguridad real. Cualquier atacante puede descubrir la URL
  mediante:
  - Archivo robots.txt
  - Error 404 vs 403 (detección de rutas válidas)
  - Referencias en HTML, JS, CSS del sidebar
  No aporta seguridad real pero complica el mantenimiento
  y la documentación.

**EVIDENCIA**
config/urls.py:21 → path('admin/panel_administrativo/', admin.site.urls)


#### 2.5.2 CustomUserAdmin(UserAdmin) con campos monkey-patched

**ALTERACIÓN**
sistema/admin.py:22-34 define CustomUserAdmin que hereda de
  UserAdmin pero agrega campos monkey-patched (origen, userPermission)
  a los fieldsets.

**OBJETIVO**
Mostrar y editar los campos inyectados en User desde el admin
  de Django.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. La herencia de UserAdmin también trae filter_horizontal
  con 'user_permissions' y 'groups' que no se usan (ver 2.3.11).
  Los campos monkey-patched aparecen como si fueran nativos, lo
  que puede confundir a administradores.

**EVIDENCIA**
sistema/admin.py:22-34 → class CustomUserAdmin(UserAdmin)
  sistema/admin.py:374 → admin.site.unregister(User)


#### 2.5.3 Cuatro views incrustadas en el admin

**ALTERACIÓN**
sistema/admin.py y administracion/admin.py contienen 4 vistas
  custom incrustadas: crear-usuario-admin, actualizar-menu,
  marcar-notificaciones-leidas, inicializar-notificaciones.

**OBJETIVO**
Funcionalidades que el admin de Django no provee nativamente.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. Views incrustadas en admin:
  - Mantenimiento adicional (no siguen el patrón CRUD estándar)
  - Cada una tiene su propia template y lógica
  - Si se migra del admin a vistas custom, hay que reescribirlas

**EVIDENCIA**
sistema/admin.py:37-46 → get_urls() + crear-usuario-admin
  sistema/admin.py:383-392 → actualizar-menu
  sistema/admin.py:510-519 → marcar-todas-leidas
  administracion/admin.py:271-279 → inicializar-notificaciones


#### 2.5.4 admindocs no instalado

**ALTERACIÓN**
django.contrib.admindocs no está en INSTALLED_APPS.

**OBJETIVO**
Mantener las apps instaladas al mínimo. No se identificó
  la necesidad de documentación automática del admin.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟢 Bajo. admindocs es opcional y los desarrolladores pueden
  consultar la documentación de otra forma.

**EVIDENCIA**
grep admindocs en settings.py/INSTALLED_APPS → 0 resultados


#### 2.5.5 Cuatro change_list_template personalizados

**ALTERACIÓN**
Cuatro admin classes tienen change_list_template propio, lo que
  requiere templates separadas para funcionalidades no estándar.

**OBJETIVO**
Agregar botones/acciones extra en las listas del admin.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟢 Bajo. Es una funcionalidad válida del admin de Django. No
  genera riesgo, solo costo de mantenimiento de 4 templates extra.

**EVIDENCIA**
sistema/admin.py → 3 change_list_template
  administracion/admin.py → 1 change_list_template


#### 2.5.6 GroupAdmin no desregistrado (grupos muestran permisos no funcionales)

**ALTERACIÓN**
GroupAdmin de Django no fue desregistrado ni personalizado.
  El admin muestra el campo group.permissions (M2M a Permission)
  que nunca se verifica en vistas.

**OBJETIVO**
No fue intencional. Simplemente no se desregistró GroupAdmin.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. Los administradores pueden asignar permisos a grupos
  en el admin (con la interfaz estándar de Django) pero esos
  permisos no tienen ningún efecto. Falsa sensación de control
  y auditoría engañosa.

**EVIDENCIA**
sistema/admin.py → unregister(User) pero no unregister(Group)


---

### 2.6 Vistas

#### 2.6.1 Cero vistas genéricas de Django

**ALTERACIÓN**
El proyecto NO usa ListView, DetailView, CreateView, UpdateView,
  DeleteView, FormView, TemplateView ni RedirectView en ninguna
  de las apps analizadas.
  Las ~72 vistas son class-based que heredan directamente de View
  con implementación manual de get() y post().

**OBJETIVO**
Control total sobre el flujo de cada vista. Las genéricas de
  Django tienen comportamientos predefinidos que no siempre se
  alinean con las necesidades del proyecto.

**DOCUMENTADO EN BASE**
❌ No documentado. README.md no describe la arquitectura de vistas.

**IMPACTO**
🔴 Crítico. Sin genéricas:
  - Cada vista reimplementa CRUD manualmente → ~72 × 20 líneas
    de boilerplate = 1,440 líneas de código repetido
  - Sin patrones predefinidos: cada vista puede tener bugs
    diferentes (validación, permisos, manejo de errores)
  - Sin paginación (ListView la incluye por defecto)
  - Sin manejo automático de formularios (CreateView/UpdateView
    conectan Form + modelo + template automáticamente)
  - Sin protección CSRF automática en POST
  - Sin mensajes de éxito/error (messages framework no usado)
  Django nativo: 4 líneas de ListView reemplazan ~30 líneas de
  View manual con get_context_data() + get() + paginate().

**EVIDENCIA**
grep class.*ListView → 0 resultados
  grep class.*CreateView → 0 resultados
  grep class.*UpdateView → 0 resultados
  grep class.*DeleteView → 0 resultados
  grep class.*DetailView → 0 resultados
  grep class.*FormView → 0 resultados
  grep class.*TemplateView → 0 resultados
  administracion/views.py → ~55 clases que heredan de View
  sistema/views.py → 11 clases que heredan de View
  servicios_web/views.py → 6 clases que heredan de View


#### 2.6.2 ~72 vistas custom con get()/post() manual

**ALTERACIÓN**
Todas las vistas del proyecto (sistema: 11, administracion: ~55,
  servicios_web: 6) heredan de View e implementan get() y post()
  manualmente, más get_context_data() duplicado en cada una.

**OBJETIVO**
Mantener control granular sobre cada endpoint. Django View es
  la clase base más simple y da control total.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. Cada vista custom:
  - Duplica código de get_context_data() (obtener queryset,
    filtrar, ordenar, serializar)
  - Duplica lógica de permisos (cada una verifica rol manualmente)
  - Duplica manejo de errores (try/except en cada método)
  - Duplica lógica de formularios (validación, save, redirect)
  Costo: cada nueva funcionalidad requiere escribir ~50-80 líneas
  vs. ~10-15 con genéricas.

**EVIDENCIA**
Patrón en cada vista de administracion/views.py:
  class XView(View):
      template_name = '...'
      def get_context_data(self, **kwargs): ...
      def get(self, request, ...): ...
      def post(self, request, ...): ...


#### 2.6.3 Cero paginación

**ALTERACIÓN**
No existe Paginator, page_obj, paginate ni ningún mecanismo de
  paginación en el proyecto.

**OBJETIVO**
Durante el desarrollo con datos de prueba (64 municipios, 1000+
  escuelas), la paginación no fue necesaria.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🔴 Crítico para el escenario ISJ. Sin paginación:
  - Cualquier listado de afiliados (190K registros) devuelve todas
    las filas en una sola query
  - La memoria del servidor se agota con cada listado
  - El navegador del usuario se cuelga al recibir 190K filas de HTML
  - El tiempo de respuesta excede cualquier timeout razonable
  Django nativo: ListView.paginate_by = 25 resuelve en 1 línea.

**EVIDENCIA**
grep Paginator → 0 resultados
  grep page_obj → 0 resultados
  grep paginate → 0 resultados


#### 2.6.4 messages framework no usado en vistas de app

**ALTERACIÓN**
Django.contrib.messages (messages framework) solo se usa en
  admin.py (sistema/admin.py y administracion/admin.py). Las ~72
  vistas de las apps no usan messages para notificar éxito/error
  al usuario.

**OBJETIVO**
No se identificó como prioridad. Las notificaciones se manejan
  vía el sistema de MensajeSistema o redirección directa.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. Sin messages framework:
  - Las operaciones CRUD no confirman éxito al usuario
  - Los errores de formulario muestran mensajes en el form pero
    no notificaciones de acción completada
  - La experiencia de usuario es confusa (submit sin
    retroalimentación)

**EVIDENCIA**
Grep messages en administracion/views.py → 0 (solo en admin.py)


#### 2.6.5 login_required() decorator en cada URL

**ALTERACIÓN**
Cada URL en los archivos urls.py de cada app está envuelta
  individualmente con login_required().

**OBJETIVO**
Asegurar que todas las vistas requieran autenticación.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟢 Bajo. Funcionalmente correcto, pero es más verbose que
  definir LOGIN_REQUIRED como setting global o usar middleware
  LoginRequiredMiddleware.

**EVIDENCIA**
Patrón en urls.py de cada app:
  path('x/', login_required(XView.as_view()), name='x')


---

### 2.7 Templates

#### 2.7.1 user_profile_context como reemplazo de {% perms %}

**ALTERACIÓN**
sistema/context_processors.py expone variables es_superadmin,
  es_admin, es_administrador, es_supervisor, es_operador, es_consulta,
  solo_lectura, user_profile, puede_ver_campanita a todas las templates.

**OBJETIVO**
Tener los roles del usuario disponibles en templates sin usar
  el sistema de permisos de Django. Es más simple que aprender
  {% perms %} y más directo.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. Cada template recibe 9+ variables adicionales en cada
  request:
  - user_profile requiere un query a UserProfile (1 query extra)
  - Las variables es_* no cubren permisos granulares (solo rol)
  - Si se agrega un nuevo rol, hay que actualizar el context
    processor, las templates y todas las vistas que lo usan
  - Duplica la funcionalidad de {{ perms }} que Django ya provee
    sin queries extra
  Django nativo: {{ perms.app_label.view_model }} en template.
  {% if perms.afiliaciones.view_solicitud %} ... {% endif %}

**EVIDENCIA**
sistema/context_processors.py → user_profile_context (líneas 60-95)
  config/settings.py:80 → 'sistema.context_processors.user_profile_context'


#### 2.7.2 Dos context processors (funciones que inyectan variables en todas las plantillas) extra por request

**ALTERACIÓN**
Además de los 5 context processors por defecto de Django, el
  proyecto ejecuta 2 adicionales en cada request:
  - user_allowed_menus
  - user_profile_context

**OBJETIVO**
Inyectar datos de menú y perfil en todas las templates sin
  tener que pasarlos desde cada vista.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Alto. Cada context processor ejecuta queries adicionales:
  - user_allowed_menus: consulta MenuGrupo + M2M
  - user_profile_context: consulta UserProfile
  Con 500 usuarios × N requests/día, esto son miles de queries
  extra que podrían evitarse.
  Django nativo: {{ perms }} y {{ user }} ya están disponibles
  sin context processors adicionales.

**EVIDENCIA**
config/settings.py:79-80 → TEMPLATES.OPTIONS.context_processors


#### 2.7.3 error.html único para 403/404/500

**ALTERACIÓN**
El proyecto usa un único template error.html para 3 tipos de
  error (403, 404, 500). Django recomienda templates separados
  (404.html, 500.html, 403.html).

**OBJETIVO**
Template unificado para errores, más fácil de mantener.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟢 Bajo. Funcional pero la experiencia de usuario es la misma
  para un "no encontrado" que para un "error interno". La
  información de debug no se muestra al usuario (correcto),
  pero el desarrollador no puede distinguir el tipo de error
  desde el template.

**EVIDENCIA**
sistema/middleware.py → NotFoundMiddleware renderiza 'error.html'


---

### 2.8 Error Handling

#### 2.8.1 NotFoundMiddleware (middleware: capa de procesamiento entre el request y la response) reemplaza handler404/500/403

**ALTERACIÓN**
sistema/middleware.py define NotFoundMiddleware que captura
  responses con status 404, 500, 403 y renderiza error.html.
  No existen handler404, handler500 ni handler403 en ninguna
  urls.py.

**OBJETIVO**
Capturar todos los errores en un solo punto, incluyendo 403
  que Django no maneja con handler por defecto. El middleware
  permite mayor control sobre el flujo de errores.

**DOCUMENTADO EN BASE**
❌ No documentado. README.md y SETUP.md no mencionan el mecanismo de error handling.

**IMPACTO**
🟡 Medio. Usar middleware en vez de handlers:
  - El middleware se ejecuta en cada request (overhead mínimo
    pero presente)
  - No se puede customizar por tipo de error desde urls.py
  - No se puede usar debug detallado en desarrollo (el middleware
    oculta el traceback de Django)
  - Si el middleware mismo falla, no hay fallback de error handling
  Django nativo: handler404, handler500, handler403 en urls.py
  raíz. Más simple y predecible.

**EVIDENCIA**
sistema/middleware.py → NotFoundMiddleware
  grep handler404 → 0 resultados
  grep handler500 → 0 resultados
  grep handler403 → 0 resultados


#### 2.8.2 Sin handler404/handler500/handler403 (funciones nativas de Django para manejar errores HTTP) definidos

**ALTERACIÓN**
No se definen los handlers de error de Django en ninguna urls.py.

**OBJETIVO**
No fue necesario porque NotFoundMiddleware captura todos los
  errores antes de que Django busque handlers.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🟡 Medio. Si se elimina el middleware (por refactor), no hay
  fallback de manejo de errores. Los errores mostrarían el
  debug de Django en producción si DEBUG está mal configurado.

**EVIDENCIA**
grep handler en urls.py → 0 resultados


---

### 2.9 DevOps

#### 2.9.1 Sin Docker

**ALTERACIÓN**
No existe Dockerfile ni docker-compose.yml. El deploy se hace
  manual sobre el servidor.

**OBJETIVO**
Mantener la complejidad baja. El equipo no tenía expertise en
  Docker.

**DOCUMENTADO EN BASE**
❌ No documentado. SETUP.md describe deploy manual como la
  única opción.

**IMPACTO**
🔴 Crítico para portabilidad. Sin Docker:
  - Entorno de desarrollo y producción no son idénticos
  - Cada nuevo desarrollador gana horas configurando su máquina
  - Rollbacks son complejos (re-clonar, re-instalar, re-migrar)
  - No hay replicabilidad del entorno de producción para debug

**EVIDENCIA**
ls Dockerfile → no existe
  ls docker-compose.yml → no existe


#### 2.9.2 Sin CI/CD (Integración Continua/Despliegue Continuo: automatización de builds, tests y deploys)

**ALTERACIÓN**
No hay GitHub Actions, GitLab CI, Jenkins ni ningún pipeline
  de integración/despliegue continuo.

**OBJETIVO**
Evitar la complejidad de configurar pipelines CI/CD.

**DOCUMENTADO EN BASE**
❌ No documentado. SETUP.md describe deploy manual completo.

**IMPACTO**
🔴 Crítico. Sin CI/CD:
  - Cada deploy es manual y propenso a errores
  - No se ejecutan tests automáticos antes de deployar
  - No hay linting ni verificación de calidad de código
  - No hay rollback automático si algo falla
  - El equipo pierde horas en deploys manuales
  Para 500+ usuarios, cada deploy manual es un riesgo operativo
  que requiere comunicación a todos los usuarios.

**EVIDENCIA**
ls .github/workflows → no existe
  Config de CI/CD → no encontrada


#### 2.9.3 requirements.txt sin versiones pinneadas

**ALTERACIÓN**
requirements.txt especifica dependencias sin versiones fijas,
  solo nombres de paquetes.

**OBJETIVO**
Evitar conflictos de versiones y dejar que pip resuelva las
  dependencias compatibles.

**DOCUMENTADO EN BASE**
❌ No documentado.

**IMPACTO**
🔴 Crítico. Sin versiones pinneadas:
  - Dos instalaciones en distintos momentos pueden obtener
    versiones diferentes de las mismas dependencias
  - Una actualización de dependencia (Django 5.1 → 5.1.1 de
    seguridad) puede introducir cambios silenciosos
  - No se puede reproducir el mismo entorno exacto
  - No hay trazabilidad de qué versiones están en producción
  Buena práctica: pip freeze > requirements.txt con versiones
  exactas, o usar Pipfile/Pipfile.lock con pipenv.

**EVIDENCIA**
requirements.txt → líneas sin versiones


#### 2.9.4 Gunicorn timeout = 600 segundos (10 minutos)

**ALTERACIÓN**
gunicorn.conf.py define timeout = 600 en vez del default de 30.

**OBJETIVO**
Operaciones WFS (Web Feature Service) de sincronización geográfica
  que pueden tardar varios minutos.

**DOCUMENTADO EN BASE**
❌ No documentado. SETUP.md menciona Gunicorn con timeout default de 30s pero no el valor custom de 600s.

**IMPACTO**
🟡 Medio. Timeout alto es necesario para sincronizaciones largas,
  pero:
  - Si una request normal se cuelga (bug, deadlock), el worker
    queda bloqueado 10 minutos
  - Con 3 workers (sugerido en SETUP.md), 3 requests colgadas
    dejan el sistema inoperativo por 10 minutos
  - No hay timeout diferenciado: las requests normales usan el
    mismo timeout que las sincronizaciones largas
  Mejora: timeout global de 30s + endpoints específicos de WFS
  sin timeout (async) o con timeout extendido.

**EVIDENCIA**
gunicorn.conf.py → timeout = 600
  SETUP.md → Gunicorn con 3 workers


#### 2.9.5 Sin tests configurados

**ALTERACIÓN**
Los archivos tests.py existen en cada app pero están vacíos.
  No hay pytest, no hay test runner configurado, no hay fixtures
  de prueba.

**OBJETIVO**
El proyecto no priorizó testing en las etapas iniciales de
  desarrollo.

**DOCUMENTADO EN BASE**
❌ No documentado explícitamente. Es un hecho conocido.

**IMPACTO**
🔴 Crítico. Sin tests:
  - Cada cambio en código puede romper funcionalidad existente
    sin que nadie lo sepa
  - Las ~72 vistas custom no tienen tests unitarios ni de
    integración
  - Los modelos no tienen tests de validación
  - No se puede hacer refactoring seguro
  - La calidad del software depende enteramente de la revisión
    manual
  Para 500+ usuarios y miles de transacciones/día, la ausencia
  de tests garantiza que bugs lleguen a producción.

**EVIDENCIA**
sistema/tests.py → vacío
  administracion/tests.py → vacío
  servicios_web/tests.py → vacío
  pytest.ini → no existe


---

## 3. Impacto por Dimensión

### 3.1 Seguridad 🔴

| Alteración | Riesgo | Severidad |
|-----------|--------|-----------|
| Base64 en passwords email | Exposición de credenciales SMTP por SQLi | 🔴 Crítico |
| Logout por GET | CSRF permite cerrar sesión forzadamente | 🔴 Crítico |
| Fail-open reCAPTCHA | Brute force bypass | 🔴 Crítico |
| Sin LogEntry ni auditoría | 500 usuarios sin trazabilidad | 🔴 Crítico |
| Monkey-patching User | Inconsistencias entre auth.User real y esperado | 🔴 Crítico |
| auth.Permission ignorado | Falsa sensación de control en admin | 🔴 Crítico |
| from credenciales import * | Error sintáctico rompe todo el sistema | 🟡 Alto |
| CSRF_HTTPONLY=True | Requests AJAX pueden fallar | 🟡 Alto |
| SECURE_SSL_REDIRECT=False | Sin HTTPS si olvidan activar | 🟡 Alto |
| Señales auth sin usar | Sin detección de brute force | 🟡 Alto |
| Sin retry en APIs | Degradación silenciosa si API externa falla | 🟡 Alto |
| user_profile_context | Exposición de roles a templates no autorizadas | 🟡 Medio |

**Riesgo acumulativo**: la combinación de base64 + fail-open + GET logout + sin auditoría significa que un ataque exitoso puede escalar a robo de contraseñas SMTP, eludir captcha, cerrar sesiones arbitrarias y operar sin dejar rastro.

---

### 3.2 Mantenibilidad 🔴

| Alteración | Costo | Severidad |
|-----------|-------|-----------|
| ~72 vistas custom (0 genéricas) | Cada bug requiere entender implementación específica | 🔴 Crítico |
| 0 tests | Cada cambio puede romper algo sin saberlo | 🔴 Crítico |
| Monkey-patching (5) | No descubrible, frágil en upgrades | 🔴 Crítico |
| CSV split en roles_permitidos | Sin integridad referencial | 🔴 Crítico |
| user_profile_context | Nuevo rol requiere actualizar processor + templates | 🟡 Alto |
| daemon thread | Notificaciones perdidas sin alerta | 🟡 Alto |
| 2 context processors extra | Queries adicionales en cada request | 🟡 Alto |
| Login/logout custom | Mantenimiento separado de auth de Django | 🟡 Alto |
| from credenciales import * | Configuración invisible en diffsettings | 🟡 Alto |
| 4 views incrustadas en admin | No siguen patrón estándar | 🟡 Medio |
| Migraciones gitignored | Schemas inconsistentes entre entornos | 🟡 Alto |

**Riesgo acumulativo**: ~72 vistas sin tests + monkey-patching + CSV + migraciones gitignored = cualquier refactor o upgrade de Django es una incógnita. El equipo debe verificar manualmente docenas de archivos después de cada cambio.

---

### 3.3 Escalabilidad 🔴

| Alteración | Cuello de botella | Impacto con 190K afiliados |
|-----------|------------------|---------------------------|
| Sin paginación | Cualquier listado carga TODOS los registros | Caída del servidor |
| 0 genéricas (ListView) | Sin paginate_by, sin filtros automáticos | Cada listado manual es inseguro |
| SMTP síncrono | Cada email bloquea el request 0.5-2s | Timeouts en cascada |
| Daemon thread sin límite | Cientos de threads bajo carga | OOM Killer (Out of Memory: el servidor agota la RAM) |
| 2 context processors extra | N+1 queries (una consulta extra por cada registro devuelto) por página | +100K queries/día extras |
| CSV split en roles_permitidos | str.split en cada request | Overhead CPU innecesario |
| 4+ queries por request solo para menú | Queries innecesarias | Contención de BD |
| Gunicorn timeout 600s | Workers bloqueados si hay bugs | Sin capacidad de servicio |

**Riesgo acumulativo**: sin paginación, el servidor cae ante el primer listado real. SMTP síncrono + daemon threads sin límite generan picos de latency y OOM. 500 usuarios concurrentes llevan el sistema al límite en hora pico.

---

### 3.4 Observabilidad 🟡

| Alteración | Problema | Severidad |
|-----------|---------|-----------|
| Sin LogEntry ni auditoría | No hay registro de accesos a datos | 🔴 Crítico |
| Señales auth sin usar | No se loguean logins fallidos | 🟡 Alto |
| Daemon thread sin monitoreo | Notificaciones perdidas sin alerta | 🟡 Alto |
| Sin health check endpoints | No se puede monitorear disponibilidad | 🟡 Alto |
| Logging custom (bien implementado) | ✅ Correcto | 🟢 Bueno |
| Sin métricas (Prometheus/StatsD) | No hay visibilidad de performance | 🟡 Alto |
| Sin alertas configuradas | Nadie sabe si el sistema está caído | 🔴 Crítico |

**Riesgo acumulativo**: un ataque de seguridad o un bug de performance puede pasar desapercibido por horas o días. Sin métricas ni alertas, los incidentes los reportan los usuarios, no el sistema.

---

### 3.5 Portabilidad / DevOps 🟡

| Alteración | Problema | Severidad |
|-----------|---------|-----------|
| Sin Docker | Entornos no reproducibles | 🔴 Crítico |
| Sin CI/CD | Deploys manuales, sin verificación automática | 🔴 Crítico |
| requirements.txt sin versiones | Builds no reproducibles | 🔴 Crítico |
| Migraciones gitignored | Schemas inconsistentes | 🔴 Crítico |
| from credenciales import * | Errores de configuración en producción | 🟡 Alto |
| Gunicorn timeout 600s | Workers bloqueados | 🟡 Alto |

**Riesgo acumulativo**: no hay forma de garantizar que dos servidores (dev, staging, prod) tengan el mismo entorno. Los deploys son manuales, riesgosos y no replicables. Un error de sintaxis en credenciales.py en producción puede dejar el sistema fuera de servicio.

---

### 3.6 Flexibilidad 🟡

| Alteración | Limitación | Severidad |
|-----------|-----------|-----------|
| 5 roles fijos en TextChoices | No se pueden crear roles sin deploy | 🔴 Crítico |
| auth.Permission ignorado | Sin permisos granulares (crear/editar/eliminar) | 🔴 Crítico |
| CSV split en roles_permitidos | Sin integridad referencial | 🟡 Alto |
| AUTH_USER_MODEL no definido | No se puede migrar a custom User fácilmente | 🔴 Crítico |
| Monkey-patching | No se pueden usar apps que requieran User estándar | 🟡 Alto |
| Sin genéricas | Cada nueva vista requiere implementación manual | 🟡 Alto |
| Sin AuthenticationBackends | No se puede integrar LDAP/SSO | 🟡 Alto |
| TextChoices frozen | Nuevos roles requieren deploy + migrate | 🟡 Alto |

**Riesgo acumulativo**: el sistema está diseñado para 5 roles fijos y una estructura auth rígida. Cualquier cambio organizacional (nuevo rol, permiso granular, integración con LDAP del gobierno) requiere modificar código fuente y hacer deploy.

---

## 4. Escenario Hipotético: Instituto de Seguro

Este escenario es hipotético y se incluye para evaluar el impacto de las alteraciones en un sistema de producción con carga real.

### 4.1 Perfil de carga

| Dimensión | Valor |
|-----------|-------|
| Afiliados en BD | 190.000 registros |
| Transacciones/día | Miles (altas, bajas, modificaciones, solicitudes) |
| Usuarios concurrentes | 500+ durante horario laboral |
| Picos de carga | 08:00-10:00 y 14:00-16:00 |
| Sesiones simultáneas | ~100-200 en hora pico |
| Requests por usuario/día | ~50-100 (navegación, CRUD, consultas) |
| Total requests/día | ~25.000 - 50.000 |

### 4.2 Lo que falla primero

**1. Sin paginación — caída inmediata**

Cualquier vista que liste afiliados ejecuta `SELECT * FROM afiliados_personafisica` devolviendo 190.000 filas. Resultado:
- El servidor agota RAM intentando construir el queryset
- Django ORM carga todos los objetos en memoria
- El navegador recibe un HTML de varios MB y se cuelga
- Tiempo de respuesta: >30 segundos (timeout del navegador)

**2. SMTP síncrono + daemon thread — cuello de botella**

Cada transacción (solicitud, aprobación, creación de usuario) puede disparar una notificación por email. Con miles de transacciones/día:
- SMTP tarda 0.5-2s por email → 500 notificaciones = 250-1000 segundos bloqueantes
- Los workers de Gunicorn se saturan con requests de email
- Las requests normales (navegación, consultas) se acumulan en cola
- El usuario percibe lentitud general, no solo en emails

Daemon thread: con cientos de threads simultáneos sin límite, el servidor puede quedar sin memoria (OOM).

**3. 4+ queries por request para el menú**

500 usuarios × 50 requests/día × 4 queries extra = 100.000 queries/día solo para determinar qué menú mostrar. La BD se satura con queries redundantes.

**4. Sin genéricas ni paginación en ~72 vistas**

Cada vista custom tiene su propia implementación de listados. Ninguna tiene paginación implementada. Las 72 vistas fueron escritas con datos de prueba (decenas de registros), no 190K.

**5. from credenciales import * — un error y todo para**

Si alguien comete un error de sintaxis en credenciales.py durante un deploy, `from credenciales import *` lanza SyntaxError y Django ni siquiera arranca. Downtime inmediato.

### 4.3 Riesgos de seguridad sobre datos personales

**1. Sin trazabilidad de accesos (Ley 25.326 de Protección de Datos Personales)**

500 usuarios acceden a datos personales de 190K afiliados (DNI, CUIL, domicilio, datos de salud) sin que quede registro de quién accedió a qué y cuándo. Esto es una violación de la Ley 25.326. En una auditoría o denuncia, no hay forma de demostrar qué empleado accedió a qué afiliado.

**2. Base64 en passwords de email: vector de escalamiento**

Cualquier vulnerabilidad SQLi (SQL Injection: ataque que inyecta comandos SQL maliciosos) en cualquiera de las ~72 vistas permite extraer la contraseña SMTP en texto plano (base64 decode). Con el control del servidor SMTP, el atacante puede suplantar al instituto para enviar phishing a los 190K afiliados.

**3. Fail-open reCAPTCHA: brute force sobre cuentas con acceso a datos sensibles**

Si reCAPTCHA falla (timeout), el login procede sin verificación. Un atacante puede realizar fuerza bruta contra cuentas de operadores que tienen acceso a los 190K afiliados. Con 500+ usuarios, la probabilidad de contraseñas débiles es alta.

**4. Logout por GET: CSRF sobre sesiones de 8h**

Las sesiones duran 8h. Un atacante puede hacer que un operador cierre sesión sin saberlo (<img src="/logout_view/">), forzándolo a reingresar. Si el operador nota algo raro, puede que no. Si no nota, la sesión queda abierta o cerrada sin control.

**5. ~72 vistas con auth custom: alta probabilidad de al menos un bypass**

Cada vista verifica roles manualmente (AdminRequiredMixin o chequeo directo). Con 72+ implementaciones diferentes, la probabilidad de que al menos una tenga un bug de autorización es muy alta. Una vista sin verificación correcta permite acceso no autorizado a datos de afiliados.

### 4.4 Riesgos operativos diarios

**1. Notificaciones perdidas sin alerta**

El daemon thread de envío de email no tiene monitoreo. Si falla (red, SMTP timeout, excepción), la notificación se pierde silenciosamente. Una solicitud que requiere aprobación puede quedar sin notificar. El usuario cree que está en proceso pero nadie lo aprobó.

**2. Sin reintentos en APIs externas**

SIAF, SUNICSU y Registro Civil son servicios externos del gobierno. Cualquier fallo de red o timeout hace fallar la request del usuario. Sin retry logic ni circuit breaker, una caída temporaria de una API externa deja el sistema inoperativo para todas las funcionalidades que dependen de ella.

**3. Monkey-patching en User: migraciones frágiles**

En cada deploy: `makemigrations` + `migrate`. Como los campos monkey-patched no generan migraciones propias, cualquier cambio en esos campos requiere verificación manual de que las migraciones reflejen correctamente el estado de la BD.

**4. Deploy manual: ventana de riesgo**

Sin CI/CD, cada deploy es una secuencia manual de comandos. Un error tipográfico, olvidar collectstatic o no ejecutar migrate en el orden correcto puede dejar el sistema caído. Rollback requiere repetir todo el proceso manualmente.

### 4.5 Costo de operación con equipo de desarrollo

| Factor | Impacto estimado |
|--------|-----------------|
| Onboarding de nuevo desarrollador | 2-4 semanas para entender auth custom + ~72 vistas |
| Tiempo en mantenimiento de vistas | ~40% del tiempo de desarrollo en deuda técnica |
| Corrección de bugs sin tests | Cada bug requiere reproducción manual + prueba manual |
| Deploys manuales | 1-2 horas por deploy vs. 10 min con CI/CD |
| Incidentes en producción detectados por usuarios | Mayor costo reputacional y operativo |

**Comparación**: un equipo que mantiene un sistema Django estándar (con genéricas, auth.Permission, admin, tests) dedica ~60% del tiempo a nuevas funcionalidades y ~40% a mantenimiento. En el estado actual del proyecto, la relación se invierte: ~60% mantenimiento, ~40% nuevas funcionalidades.

### 4.6 Prioridad de corrección para producción

| Prioridad | Qué cambiar | Por qué |
|-----------|------------|---------|
| **Día 1** | Paginación en listados | Sin esto, el sistema no opera con 190K registros |
| **Día 1** | Fail-open reCAPTCHA → fail-closed | El login es la puerta de entrada |
| **Día 1** | Logout GET → POST | CSRF crítico |
| **Semana 1** | Base64 → cifrado real (Fernet) | Riesgo inmediato de exposición de credenciales |
| **Semana 1** | from credenciales import * → validación | Error sintáctico = caída total |
| **Sprint 1** | tests.py → tests mínimos de auth y modelos | Sin tests no se puede refactorizar nada |
| **Sprint 1** | Context processors → reducir queries extra | 100K queries/día extras |
| **Sprint 1-2** | Paginación permanente (ListView genéricas) | Sin paginación, el sistema no escala |
| **Sprint 2-3** | requirements.txt con versiones | Reproducibilidad de builds |
| **Sprint 2-3** | GitHub Actions básico (lint + test + deploy) | Deploy automatizado |
| **Sprint 3-4** | Dockerizar (Dockerfile + compose) | Entornos reproducibles |
| **Sprint 3-4** | handler404/500/403 nativos | Reemplazar middleware |
| **Sprint 4-6** | auth.Permission como único sistema de autorización | Eliminar CSV + context processors |
| **Sprint 4-6** | Auditoría (señales auth + LogEntry) | Trazabilidad requerida por ley |
| **6+ meses** | AUTH_USER_MODEL propio + migración | Unificar user + profile |
| **6+ meses** | SMTP async con Celery + Redis | Cola de tareas robusta |

---

## 5. Tabla Comparativa Django Nativo vs. Proyecto

| Aspecto | Django Nativo | Proyecto (lo que hace) | Alteración |
|---------|--------------|----------------------|------------|
| **User model** | settings.AUTH_USER_MODEL | auth.User + monkey-patch add_to_class | 2.1.1 / 2.1.3 |
| **Group model** | auth.Group con permissions M2M | auth.Group + monkey-patch home/icon | 2.1.2 |
| **Migraciones** | Versionadas en git | Gitignored, regenerar cada clone | 2.1.4 |
| **Perfil de usuario** | AbstractUser con campos propios | UserProfile OneToOne + add_to_class | 2.1.5 |
| **Timezones** | USE_TZ = True (UTC) | USE_TZ = False | 2.2.1 |
| **Internacionalización** | USE_I18N = True | USE_I18N = False | 2.2.2 |
| **Config sensible** | python-decouple / .env | from credenciales import * | 2.2.3 |
| **CSRF token** | Cookie HttpOnly=True (Django 5.1) | CSRF_HTTPONLY=True | 2.2.4 |
| **Sesiones** | 2 semanas, browser close False | 8h, browser close True | 2.2.5 |
| **Reset password** | 3 días de expiración | 1 hora | 2.2.6 |
| **Captcha** | django-recaptcha app | Custom LoginConCaptchaView | 2.2.7 / 2.3.5 |
| **SSL** | SECURE_SSL_REDIRECT = True | False (se activa en credenciales) | 2.2.8 |
| **AutoField** | AutoField (int 32 bits) | BigAutoField (int 64 bits) | 2.2.9 |
| **Login URL** | /accounts/login/ | /login/ | 2.2.10 |
| **Logging** | Console por defecto | RotatingFileHandler + 4 loggers | 2.2.11 |
| **Control de acceso** | @permission_required, has_perm, {% perms %} | 0 usos — sistema custom de roles | 2.3.1 |
| **Roles** | auth.Group dinámicos | 5 fijos en TextChoices | 2.3.2 |
| **Vinculación rol-menú** | ContentType + Permission | CSV CharField + str.split | 2.3.3 |
| **Menús por usuario** | {% perms %} en template | M2M monkey-patched en User | 2.3.4 |
| **Login view** | LoginView de Django | LoginConCaptchaView con fail-open | 2.3.5 |
| **Logout** | POST requerido (CSRF protegido) | GET permitido | 2.3.6 |
| **Señales auth** | logged_in, logged_out, login_failed | 0 suscriptores | 2.3.7 |
| **Auth backends** | ModelBackend (default) + LDAP, etc. | ModelBackend, sin config extra | 2.3.8 |
| **PermissionRequired** | PermissionRequiredMixin / @permission_required | AdminRequiredMixin custom | 2.3.9 |
| **Group form** | GroupForm de Django con name + permissions | Custom GroupForm con campos monkey-patched | 2.3.10 |
| **user_permissions admin** | Se verifican en vistas | Se muestran en admin pero no se verifican | 2.3.11 |
| **Template permissions** | {% perms %} sin queries extra | user_profile_context con queries extra | 2.3.12 / 2.7.1 |
| **Auth forms** | AuthenticationForm, UserCreationForm, etc. | Todos custom | 2.3.13 |
| **Email envío** | send_mail() + EMAIL_BACKEND | smtplib directo | 2.4.1 |
| **Email passwords** | Hasheadas o en variables de entorno | Base64 en BD | 2.4.2 |
| **Email async** | Celery / django-qsqueue / django-rq | Daemon thread sin monitoreo | 2.4.3 |
| **Email configuración** | EMAIL_* settings | ConfiguracionCorreo en BD | 2.4.4 |
| **Admin URL** | /admin/ | /admin/panel_administrativo/ | 2.5.1 |
| **User admin** | UserAdmin estándar | CustomUserAdmin con campos monkey-patched | 2.5.2 |
| **Admin views** | CRUD estándar | 4 views incrustadas + custom templates | 2.5.3 |
| **admindocs** | django.contrib.admindocs | No instalado | 2.5.4 |
| **Group admin** | GroupAdmin con permissions | No desregistrado (permisos no funcionales) | 2.5.6 |
| **Vistas CRUD** | ListView, CreateView, UpdateView, DeleteView | ~72 View con get()/post() manual | 2.6.1 / 2.6.2 |
| **Paginación** | ListView.paginate_by = 25 | 0 paginación en todo el proyecto | 2.6.3 |
| **Messages** | messages.success(), messages.error() | No usado en vistas de app | 2.6.4 |
| **Auth en URLs** | LoginRequiredMiddleware | login_required() en cada URL | 2.6.5 |
| **Context processors** | 5 default + {{ perms }} automático | +2 procesadores con queries extra | 2.7.2 |
| **Error templates** | 404.html, 500.html, 403.html | error.html único | 2.7.3 |
| **Error handlers** | handler404, handler500 en urls.py | NotFoundMiddleware | 2.8.1 / 2.8.2 |
| **Contenedores** | Dockerfile + docker-compose | No existe | 2.9.1 |
| **CI/CD** | GitHub Actions / GitLab CI / Jenkins | No existe | 2.9.2 |
| **Dependencias** | requirements.txt con versiones | requirements.txt sin versiones | 2.9.3 |
| **Gunicorn timeout** | 30s | 600s | 2.9.4 |
| **Tests** | pytest / unittest con tests | tests.py vacíos | 2.9.5 |

---

## 6. Recomendaciones Priorizadas

### 6.1 Urgente (día 1 en producción)

Son las alteraciones que **impiden la operación** con 190K afiliados o generan **riesgo de seguridad inaceptable**:

| # | Acción | Alteración resuelta | Esfuerzo |
|---|--------|-------------------|----------|
| 1 | **Implementar paginación** en todos los listados (mínimo: template tag Paginator o migrar a ListView con paginate_by) | 2.6.3 | 2-3 días |
| 2 | **Cambiar fail-open reCAPTCHA a fail-closed**: si Google no responde, denegar login y loguear | 2.3.5 | 2 horas |
| 3 | **Cambiar logout de GET a POST** (o usar LogoutView de Django con POST requerido) | 2.3.6 | 1 hora |
| 4 | **Reemplazar base64 por cifrado real** (django.core.signing.sign() o Fernet) para passwords de email | 2.4.2 | 4 horas |
| 5 | **Validar credenciales.py al startup**: agregar un check que verifique que SECRET_KEY, DATABASES y tokens requeridos estén definidos, y detenga el arranque si falta algo crítico | 2.2.3 | 1 día |

### 6.2 Corto plazo (1-2 sprints)

| # | Acción | Alteración resuelta | Esfuerzo |
|---|--------|-------------------|----------|
| 6 | **Agregar tests mínimos**: login/logout, modelos principales, 2-3 vistas críticas | 2.9.5 | 3-5 días |
| 7 | **Reducir context processors**: eliminar user_profile_context y usar {{ perms }} nativo en templates | 2.7.1 / 2.7.2 | 2-3 días |
| 8 | **Pinear requirements.txt** con versiones exactas (`pip freeze > requirements.txt`) | 2.9.3 | 1 hora |
| 9 | **Agregar señales auth**: loguear user_logged_in, user_logged_out, user_login_failed con timestamp y datos del request | 2.3.7 | 1 día |
| 10 | **Migrar handler errors**: reemplazar NotFoundMiddleware por handler404, handler500, handler403 nativos en urls.py | 2.8.1 / 2.8.2 | 1 día |
| 11 | **Eliminar filter_horizontal = ('user_permissions',)** de CustomUserAdmin o implementar verificación real | 2.3.11 | 2 horas |
| 12 | **Configurar GitHub Actions mínimo**: lint + test + deploy automático a staging | 2.9.2 | 2-3 días |

### 6.3 Mediano plazo (2-4 sprints)

| # | Acción | Alteración resuelta | Esfuerzo |
|---|--------|-------------------|--------|
| 13 | **Migrar ~72 vistas a ListView/CreateView/UpdateView/DeleteView** con paginación, filters, y messages framework | 2.6.1 / 2.6.2 / 2.6.4 | 2-3 sprints |
| 14 | **Dockerizar**: Dockerfile multi-stage + docker-compose.yml con PostgreSQL, Nginx, Gunicorn | 2.9.1 | 3-5 días |
| 15 | **Reemplazar SMTP directo**: conectar al EMAIL_BACKEND de Django usando ConfiguracionCorreo como backend custom | 2.4.1 | 2-3 días |
| 16 | **Implementar reintentos en APIs externas**: retry con backoff para SIAF, SUNICSU, Registro Civil | 2.4.3 (parcial) | 3-5 días |
| 17 | **Reemplazar sistema de roles por auth.Permission**: migrar MenuGrupo.roles_permitidos a Permission Groups | 2.3.1 / 2.3.2 / 2.3.3 | 2-3 sprints |
| 18 | **Agregar trazabilidad**: implementar LogEntry o modelo custom de auditoría para accesos a datos personales | 2.3.7 (auditoría) | 1 sprint |

### 6.4 Largo plazo (6+ meses)

| # | Acción | Alteración resuelta | Esfuerzo |
|---|--------|-------------------|--------|
| 19 | **Migrar a AUTH_USER_MODEL propio**: crear AbstractUser con todos los campos de User + UserProfile + campos monkey-patched. Requiere migración compleja (proxy model + rename table + data migration) | 2.1.1 / 2.1.3 / 2.1.5 | 1-2 meses planificado |
| 20 | **Reemplazar daemon thread por Celery + Redis/RabbitMQ** para envío de email y tareas asincrónicas | 2.4.3 | 2-3 sprints |
| 21 | **Refactorizar config crienciales**: migrar de `from credenciales import *` a django-environ con .env.dev / .env.prod | 2.2.3 | 3-5 días |
| 22 | **Migrar a PostgreSQL con connection pooling** (Pgbouncer) y configurar adecuadamente | 2.9.1 (infra) | 2-3 días |
| 23 | **Evaluar migración a Django REST Framework** si se requiere API para integraciones | Arquitectura | 2-3 meses |

---

## Glosario de términos técnicos

| Término | Explicación |
|---------|-------------|
| **AbstractUser** | Clase base de Django para crear modelos de usuario personalizados con todos los campos de auth.User ya incluidos |
| **add_to_class()** | Método de Django que agrega atributos a una clase en tiempo de ejecución (monkey-patching) |
| **AUTH_USER_MODEL** | Configuración de Django que define qué modelo se usa para representar usuarios |
| **Base64** | Codificación que convierte datos binarios a texto ASCII. NO es cifrado, cualquiera puede decodificarlo |
| **CBV / FBV** | Class-Based Views / Function-Based Views — dos estilos de escribir vistas en Django |
| **CI/CD** | Continuous Integration / Continuous Deployment — automatización de integración de código y despliegue |
| **Context processor** | Función que inyecta variables en el contexto de todas las plantillas del proyecto |
| **CSRF** | Cross-Site Request Forgery — ataque que ejecuta acciones no autorizadas en nombre de un usuario autenticado |
| **Fail-open** | Comportamiento donde un sistema de seguridad permite el acceso si el componente de verificación falla |
| **FK (Foreign Key)** | Relación donde un registro de una tabla apunta a un registro de otra |
| **Genéricas (generic views)** | Vistas pre-diseñadas de Django (ListView, CreateView, UpdateView, DeleteView) que implementan CRUD con pocas líneas |
| **handler404/handler500** | Funciones nativas de Django que se ejecutan cuando ocurre un error 404 (no encontrado) o 500 (error interno) |
| **LogEntry** | Modelo de Django que registra automáticamente las acciones realizadas en el admin (crear, modificar, eliminar) |
| **M2M (Many-to-Many)** | Relación entre dos tablas donde muchos registros de una tabla se asocian con muchos de la otra |
| **Middleware** | Capa de software que procesa cada request/response entre el servidor y la vista |
| **Monkey-patching** | Técnica que modifica clases en tiempo de ejecución agregando o reemplazando atributos |
| **N+1 queries** | Problema de performance donde se ejecuta 1 consulta para obtener N registros y luego N consultas adicionales |
| **OOM (Out of Memory)** | Error que ocurre cuando un proceso agota la memoria RAM disponible |
| **ORM** | Object-Relational Mapping — capa que traduce objetos Python a tablas de base de datos |
| **Señales (signals)** | Sistema de Django que permite que componentes se notifiquen cuando ocurren eventos |
| **SQLi (SQL Injection)** | Ataque que inyecta comandos SQL maliciosos a través de inputs de usuario |
| **TextChoices** | Clase de Django para definir campos con opciones fijas (similar a un enum) |

---

## Anexo: Referencia de alteraciones por archivo

| Archivo | Alteraciones |
|---------|-------------|
| `sistema/models.py:11-12` | 2.1.2 Monkey-patching Group |
| `sistema/models.py:53-63` | 2.3.3 CSV roles_permitidos |
| `sistema/models.py:98-100` | 2.1.1 Monkey-patching User |
| `sistema/admin.py:22-34` | 2.5.2 CustomUserAdmin |
| `sistema/admin.py:374` | 2.5.2 Unregister User |
| `sistema/context_processors.py` | 2.3.12 / 2.7.1 user_profile_context |
| `sistema/context_processors.py` | 2.7.2 user_allowed_menus |
| `sistema/middleware.py` | 2.8.1 NotFoundMiddleware |
| `sistema/views.py:45` | 2.3.6 Logout GET |
| `sistema/views.py:527` | 2.3.5 Fail-open reCAPTCHA |
| `administracion/models.py:71-76` | 2.3.2 TextChoices 5 roles |
| `administracion/models.py:311-324` | 2.4.2 Base64 passwords |
| `administracion/views.py:24` | 2.3.9 AdminRequiredMixin |
| `administracion/forms.py:238` | 2.3.10 GroupForm custom |
| `administracion/signals.py:80-83` | 2.4.3 Daemon thread |
| `administracion/services/email_service.py:6` | 2.4.1 smtplib directo |
| `config/settings.py:147` | 2.2.2 USE_I18N=False |
| `config/settings.py:148` | 2.2.1 USE_TZ=False |
| `config/settings.py:115-118` | 2.2.5 Sesiones |
| `config/settings.py:121` | 2.2.4 CSRF_HTTPONLY=True |
| `config/settings.py:261-264` | 2.2.3 From credenciales import * |
| `config/urls.py:21` | 2.5.1 Admin URL ofuscada |
| `config/urls.py:20` | 2.3.5 Login URL custom |
| `config/urls.py:23` | 2.3.6 Logout URL custom |
| `.gitignore` | 2.1.4 Migraciones gitignored |
| `requirements.txt` | 2.9.3 Sin versiones |
| `gunicorn.conf.py` | 2.9.4 Timeout 600s |
| `*.tests.py (empty)` | 2.9.5 Sin tests |

---

*Fin del informe. 35+ alteraciones documentadas en 9 categorías.*
