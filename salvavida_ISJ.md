# Plan de Migración — Proyecto Django Limpio para ISJ

## Objetivo
Migrar las apps `afiliaciones` (32 modelos) y `geografico_genos` (4 modelos) del proyecto base "Gobierno de Jujuy" a un proyecto Django 5.1 limpio, eliminando toda la deuda técnica del sistema custom de auth, admin, email y configuraciones. Portar colores, loader y branding del ISJ.

---

## Estado actual de lo que se migra

| App | Modelos | Vistas | Templates | Archivos |
|-----|---------|--------|-----------|----------|
| `afiliaciones` | 32 (15 catálogos + 8 core + 4 transacciones + 3 estados + 2 Domicilio) | 21 (12 CBV + 9 FBV) | 7 | forms.py, utils.py, admin.py, management command |
| `geografico_genos` | 4 (Provincia > Departamento > Localidad > Barrio) | 0 | 0 | admin.py + CSV translator en views.py |
| Branding | Navy #13304D, 8 temas, Poppins/Ubuntu | Loader canvas neural + orbs + logo pulse + ring | base.html, login.html | theme-config.css, main.css, logos, favicon |
| ISJ Branding | ❌ No existe — todo dice "Gobierno de Jujuy" | — | — | — |

---

## Estructura del nuevo proyecto

```
nuevo_isj/
├── manage.py
├── requirements.txt
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── afiliaciones/
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── admin.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── forms.py
│   │   ├── utils.py
│   │   ├── tests.py
│   │   ├── management/commands/
│   │   └── templates/afiliaciones/
│   └── geografico_genos/
│       ├── __init__.py
│       ├── apps.py
│       ├── models.py
│       ├── admin.py
│       ├── views.py
│       └── tests.py
├── static/
│   └── assets/
│       ├── css/          # theme-config.css + main.css
│       ├── images/       # logos ISJ, favicon
│       ├── fonts/        # Poppins, Ubuntu
│       └── js/           # solicitud_simple.js
└── templates/
    ├── base.html
    ├── registration/login.html
    └── afiliaciones/     # 7 templates
```

---

## Fase 1 — Scaffolding (1-2 días)

### Paso a paso

```bash
# 1. Crear proyecto
django-admin startproject config .
mkdir -p apps/afiliaciones apps/geografico_genos
mkdir -p static/assets/{css,images,fonts,js}
mkdir -p templates/registration templates/afiliaciones

# 2. Settings esenciales
```

**config/settings.py — puntos clave:**
```python
USE_TZ = True
TIME_ZONE = 'America/Argentina/Buenos_Aires'
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'apps.afiliaciones',
    'apps.geografico_genos',
]
# Nada de sistema, administracion, servicios_web
```

**Archivos a copiar del proyecto actual al nuevo:**

| Origen (proyecto base) | Destino (nuevo proyecto) |
|------------------------|-------------------------|
| `static/assets/css/theme-config.css` | `static/assets/css/theme-config.css` |
| `static/assets/css/main.css` | `static/assets/css/main.css` |
| `static/assets/images/logo-blanco.png` | `static/assets/images/logo-blanco.png` (reemplazar por ISJ) |
| `static/assets/images/logo-azul.png` | `static/assets/images/logo-azul.png` (reemplazar por ISJ) |
| `static/assets/images/favicon.ico` | `static/assets/images/favicon.ico` (reemplazar por ISJ) |
| `static/assets/fonts/` | `static/assets/fonts/` |
| `templates/base.html` | `templates/base.html` (simplificar) |
| `templates/registration/login.html` | `templates/registration/login.html` |

**Modificaciones a base.html:**
- Eliminar sidebar dinámico con menús desde BD (`menu.html`, `menuAyuda.html`, `setting.html`)
- Reemplazar sidebar por uno estático con links a las apps del ISJ
- Eliminar dependencia de context processors (`user_allowed_menus`, `user_profile_context`)
- Simplificar navbar (quitar notificaciones, tutoriales si no aplican)
- **Conservar**: loader neural canvas, orbs, logo pulse, scroll-to-top button, transiciones CSS

**Colores:** `--theme-primary: #13304D` (Navy). Si ISJ tiene otro color corporativo, se cambia acá.

---

## Fase 2 — Portar `geografico_genos` (1 día)

### Archivos

| Origen | Destino | Cambios |
|--------|---------|---------|
| `geografico_genos/models.py` | `apps/geografico_genos/models.py` | Copy exacto |
| `geografico_genos/admin.py` | `apps/geografico_genos/admin.py` | Copy exacto |
| `geografico_genos/views.py` | `apps/geografico_genos/views.py` | Copy exacto (CSV translator) |
| `geografico_genos/apps.py` | `apps/geografico_genos/apps.py` | Ajustar `name = 'apps.geografico_genos'` |

```bash
python manage.py makemigrations geografico_genos
python manage.py migrate
```

Crear management command `cargar_geografia_genos.py` para seed de datos (portar del script raíz `cargar_geografia_genos.py` si existe).

---

## Fase 3 — Portar modelos de `afiliaciones` (2-3 días)

### Archivos

| Origen | Destino | Cambios |
|--------|---------|---------|
| `afiliaciones/models.py` | `apps/afiliaciones/models.py` | -PersonaReparticion (deprecated) |
| `afiliaciones/admin.py` | `apps/afiliaciones/admin.py` | Copy exacto |
| `afiliaciones/forms.py` | `apps/afiliaciones/forms.py` | Copy exacto |
| `afiliaciones/apps.py` | `apps/afiliaciones/apps.py` | Ajustar `name` |
| `afiliaciones/utils.py` | `apps/afiliaciones/utils.py` | Refactor email |
| `afiliaciones/management/commands/` | `apps/afiliaciones/management/commands/` | Copy exacto |

### Usuarios: auth.User de Django, sin custom model

No se crea un modelo de usuario propio. Se usa `auth.User` de Django tal cual.

Si en el futuro se necesitan datos extra (CUIL, repartición del usuario), se agrega un modelo `UserProfile(OneToOneField(auth.User))` — es el patrón estándar de Django, no requiere `AUTH_USER_MODEL` ni monkey-patching.

**Roles y grupos:** se manejan desde el admin de Django con `auth.Group` y `auth.Permission`. No se necesita campo `rol` en el usuario.

### Ajustes a modelos existentes

**1. Eliminar `PersonaReparticion`** — marcado como deprecated, reemplazado por `RAfiliado`

**2. `Solicitud.usuario`** — queda como `FK(auth.User)`, sin cambios

**3. FKs a `geografico_genos`** — se mantienen igual:
- `Domicilio.barrio_fk` → FK(`geografico_genos.Barrio`)
- `DomicilioJuridica` → igual
- `Domicilio.localidad_fk` → FK(`geografico_genos.Localidad`)

**4. Migraciones:** se generan frescas (no se heredan del proyecto base)

```bash
python manage.py makemigrations afiliaciones
python manage.py migrate
python manage.py cargar_datos_afiliaciones
```

---

## Fase 4 — Refactorizar vistas de `afiliaciones` (3-5 días)

### Estrategia por tipo de vista

| Vista actual (View manual) | Nueva (generic view) | Esfuerzo |
|---------------------------|---------------------|----------|
| `ParametroLista` + `ParametroAjax` | `ListView` + DataTables | 2-3 horas |
| `ParametroMensaje` + `ParametroMensajeAccion` | `CreateView` / `UpdateView` | 2-3 horas |
| `SolicitudLista` + `SolicitudAjax` | `ListView` con `paginate_by=25` | 3-4 horas |
| `SolicitudFormulario` + `SolicitudGuardar` | `CreateView` + `FormView` | 4-5 horas |
| `SolicitudGestionar` | Se mantiene como `View` (lógica de negocio) | 0 |
| `SolicitudAprobar` | Se mantiene como `View` (lógica de negocio) | 0 |
| `SolicitudEnviarCorreo` | Se mantiene como `View` (envío email) | 0 |
| `SolicitudObservacion` | Se mantiene como `View` (historial) | 0 |
| `solicitud_simple` (wizard) | Se mantiene como FBV | 0 |
| `api_buscar_cuil` | Se mantiene como FBV | 0 |
| `api_obtener_requisitos` | Se mantiene como FBV | 0 |
| `api_crear_solicitud` | Se mantiene como FBV | 0 |
| `VerificarDatos` (OCR) | Se mantiene como FBV | 0 |
| 5 APIs geográficas | Se mantienen como FBV | 0 |

### Autorización (cambios importantes)

| Antes (proyecto base) | Después (Django nativo) |
|----------------------|------------------------|
| `@login_required` en cada URL | `LoginRequiredMixin` en CBVs / `@login_required` en FBVs |
| `AdminRequiredMixin` custom | `@permission_required('afiliaciones.view_solicitud')` |
| `user_profile_context` con es_* | `{{ perms.afiliaciones }}` en templates |
| Verificación manual de rol en vistas | Django `PermissionRequiredMixin` |

### URLs

Simplificar: eliminar `login_required()` de cada path, usar `LoginRequiredMixin` en CBVs.

```python
# Antes (proyecto base):
path('solicitudes/', login_required(SolicitudLista.as_view()), name='solicitud_lista')

# Después (Django limpio):
path('solicitudes/', SolicitudLista.as_view(), name='solicitud_lista')
# LoginRequiredMixin está en la clase
```

---

## Fase 5 — Branding ISJ (2-3 días)

### Cambios de texto

| Archivo | Texto actual ("Gobierno de Jujuy") | Nuevo ("ISJ") |
|---------|-----------------------------------|---------------|
| `templates/base.html` - navbar brand | "Gobierno de Jujuy" | "Instituto Seguro Jujuy" |
| `templates/base.html` - loader text | — | Opcional |
| `templates/registration/login.html` | Títulos y branding gobierno | "ISJ - Sistema de Afiliaciones" |
| `templates/afiliaciones/*.html` | Referencias a gobierno | ISJ |
| Footer en templates | "Dirección de Sistemas — MHF — Gobierno de Jujuy" | "Dirección de Sistemas — ISJ" |
| `utils.py` - PDF | Logo gobierno en informe | Logo ISJ |

### Cambios de imágenes

| Archivo | Reemplazar por |
|---------|---------------|
| `static/assets/images/logo-blanco.png` | Logo ISJ blanco (navbar, loader) |
| `static/assets/images/logo-azul.png` | Logo ISJ azul (login, home) |
| `static/assets/images/favicon.ico` | Favicon ISJ |

### Loader y transiciones (se conservan sin cambios)

Archivos que contienen el loader y no se modifican:
- `theme-config.css` (líneas 1569-1689: estilos del loader, scroll-to-top, transiciones)
- `base.html` (líneas 40-114: HTML del loader + JS del canvas neural)
- `base.html` (línea 198-209: modal spinner secundario)
- `base.html` (líneas 735-813: scroll-to-top button)

---

## Fase 6 — Email y APIs (2-3 días)

### Email (reemplazar sistema actual)

```python
# En settings.py
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
# Opcional: EMAIL_HOST, EMAIL_PORT configurados en credenciales

# Modelo ConfiguracionCorreo portado de administracion
# Pero con cifrado REAL usando django.core.signing, NO base64
from django.core.signing import Signer
signer = Signer()
# Guardar: signer.sign(password)
# Leer: signer.unsign(encrypted)

# Envío con send_mail()
from django.core.mail import send_mail
send_mail(asunto, mensaje_texto, remitente, [destinatario], html_message=html)
```

**Async opcional:** Si se necesita email asincrónico, usar `django-qsqueue` (cola en BD, sin Redis) en vez de daemon thread.

### APIs externas (SIAF, SUNICSU, Registro Civil)

Crear servicios livianos con `requests` + `tenacity`:

```python
import requests
from tenacity import retry, stop_after_attempt, wait_exponential

class SiafService:
    BASE_URL = settings.API_SIAF_URL
    
    @retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))
    def get_reparticiones(self):
        resp = requests.get(f'{self.BASE_URL}/reparticiones',
                          headers={'Authorization': f'Bearer {settings.TOKEN_API_SIAF}'})
        resp.raise_for_status()
        return resp.json()
```

---

## Lo que NO se porta (reemplazado por Django nativo)

| Componente del proyecto base | Reemplazo en Django limpio |
|-----------------------------|---------------------------|
| `sistema/` (app completa) | Django admin + auth nativos |
| `administracion/` (~55 vistas CRUD) | Django admin |
| `servicios_web/` | Clientes API livianos con requests + tenacity |
| `UserProfile` + monkey-patching (5 add_to_class) | `auth.User` — sin cambios |
| `MenuGrupo` + `roles_permitidos` CSV | `auth.Group` + `auth.Permission` |
| `allowed_menus` M2M | `{{ perms }}` en templates |
| `user_allowed_menus` + `user_profile_context` | `{{ perms }}` en templates |
| `NotFoundMiddleware` | `handler404` + `handler500` nativos en urls.py |
| `LoginConCaptchaView` + logout GET | `LoginView` + `LogoutView` (POST) de Django |
| `EmailService` + smtplib + base64 | `send_mail()` + `EMAIL_BACKEND` |
| Daemon thread para email async | `django-qsqueue` o sincrónico directo |
| `ConfiguracionCorreo` con base64 | `ConfiguracionCorreo` con `django.core.signing` |
| `from credenciales import *` | `python-decouple` con `.env` |
| `USE_TZ=False` | `USE_TZ=True` (UTC) |
| Migraciones gitignored | Migraciones versionadas en git |
| `requirements.txt` sin versiones | `requirements.txt` con versiones pinneadas |

---

## Tiempo estimado total

| Fase | Días | Depende de |
|------|:----:|-----------|
| 1 — Scaffolding + tema + loader | 1-2 | Nada |
| 2 — Portar geografico_genos | 1 | Fase 1 |
| 3 — Portar modelos afiliaciones | 2-3 | Fase 1, 2 |
| 4 — Refactorizar vistas afiliaciones | 3-5 | Fase 3 |
| 5 — Branding ISJ | 2-3 | Fase 1 (logos, textos) |
| 6 — Email + APIs | 2-3 | Fase 1 |
| **Total** | **11-17 días hábiles** | ~3 semanas |

---

## Resumen de archivos portados

| Origen (proyecto base) | Destino (nuevo proyecto) | Modificaciones |
|------------------------|-------------------------|---------------|
| `afiliaciones/models.py` | `apps/afiliaciones/models.py` | -PersonaReparticion (deprecated) |
| `afiliaciones/admin.py` | `apps/afiliaciones/admin.py` | Copy exacto |
| `afiliaciones/forms.py` | `apps/afiliaciones/forms.py` | Copy exacto |
| `afiliaciones/views.py` | `apps/afiliaciones/views.py` | Refactor CBV→generic views, permisos Django |
| `afiliaciones/urls.py` | `apps/afiliaciones/urls.py` | Ajustar namespaces |
| `afiliaciones/utils.py` | `apps/afiliaciones/utils.py` | Refactor email a send_mail() |
| `geografico_genos/models.py` | `apps/geografico_genos/models.py` | Copy exacto |
| `geografico_genos/admin.py` | `apps/geografico_genos/admin.py` | Copy exacto |
| `geografico_genos/views.py` | `apps/geografico_genos/views.py` | Copy exacto |
| `static/assets/css/theme-config.css` | `static/assets/css/theme-config.css` | Copy exacto (cambiar color si ISJ tiene otro) |
| `static/assets/css/main.css` | `static/assets/css/main.css` | Copy exacto |
| `static/assets/images/*.png` | `static/assets/images/` | Reemplazar logos por ISJ |
| `static/assets/fonts/*` | `static/assets/fonts/` | Copy exacto |
| `static/assets/js/solicitud_simple.js` | `static/assets/js/solicitud_simple.js` | Copy exacto |
| `templates/base.html` | `templates/base.html` | Simplificar sidebar, cambiar textos |
| `templates/afiliaciones/*.html` | `templates/afiliaciones/` | Ajustar template tags ({{ perms }} en vez de es_*) |
| `templates/registration/login.html` | `templates/registration/login.html` | Cambiar textos, branding ISJ |
| `afiliaciones/management/commands/` | `apps/afiliaciones/management/commands/` | Copy exacto |
| `requirements.txt` | `requirements.txt` | Pinear versiones exactas |
| `.gitignore` | `.gitignore` | NO ignorar migraciones |

---

## Pendientes para definir

1. **Color corporativo del ISJ** — si es Navy (#13304D) u otro. Se cambia en `--theme-primary` de `theme-config.css`.
2. **Logos del ISJ** — blanco (navbar/loader), azul (login), favicon. Placeholder hasta tenerlos.
3. **Loader** — ¿se mantiene el canvas neural network exacto o se simplifica?
4. **APIs externas** — ¿SIAF, SUNICSU, Registro Civil son necesarias para el ISJ o solo las APIs de afiliaciones?
5. **Notificaciones por email** — ¿críticas desde el día 1 o pueden esperar a tener Celery/qsqueue?
6. **geografico_genos** — ¿arbol completo (Provincia > Departamento > Localidad > Barrio) necesario para ISJ? ¿o solo provincias/localidades de Jujuy?
