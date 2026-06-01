# Impacto del Sistema Actual en Desarrolladores Trainee / Junior

## Proyecto: Sistema de Gestión — Adaptación ISJ
### Fecha: 31/05/2026 | Formato: Gerencial

---

## 1. El problema en 3 líneas

El proyecto actual tiene 35 modificaciones respecto a Django estándar. Un desarrollador que terminó su formación en Django no reconoce el sistema que encuentra: los usuarios, los permisos, las pantallas de administración, el envío de emails y hasta la forma de mostrar errores funcionan distinto a lo que estudió.

Tiene que **aprenderlo todo de nuevo**. Y lo que aprende acá **no le sirve para ningún otro proyecto Django** del mercado.

---

## 2. Comparación: Proyecto actual vs. Django estándar

| Actividad común | En el proyecto actual | En Django estándar | Diferencia |
|----------------|---------------------|-------------------|:----------:|
| Crear una pantalla para listar, crear, editar y eliminar un registro | 2-3 días — escribir vista, formulario, template y URL a mano | 2 horas — Django ya lo genera automáticamente | **~10x más lento** |
| Agregar un permiso para que solo algunos usuarios vean un botón | 1 día — entender el sistema de roles, modificar CSV, revisar menú | 10 minutos — una línea en la plantilla | **~48x más lento** |
| Hacer que un usuario inicie sesión | 1-2 días — configurar captcha, entender login custom | Ya funciona desde que se crea el proyecto | **No disponible** |
| Encontrar por qué falla una pantalla | 1 día — el error se oculta en una página genérica, hay que buscar en logs | 10 minutos — Django muestra el error exacto | **~48x más lento** |
| Buscar en internet cómo resolver un problema | ❌ No existe — el sistema es único en el mundo, nadie tuvo este problema | ✅ Stack Overflow tiene 1M+ respuestas | **No tiene solución externa** |
| Agregar una tabla nueva al sistema | 3-4 días — crear modelo + vista manual + formulario + template + menú + permisos | 2 horas — Django admin la cubre automáticamente | **~16x más lento** |

---

## 3. Costo económico y de equipo

### Tiempo de onboarding (hasta ser productivo)

| Perfil | Proyecto actual | Django estándar |
|--------|:--------------:|:--------------:|
| Trainee (recién formado) | 3-4 semanas | 3-5 días |
| Junior (6-12 meses exp.) | 2-3 semanas | 2-3 días |
| Senior | 1-2 semanas | 0 — ya lo conoce |

**Costo directo:** Un trainee en el proyecto actual cuesta ~3 semanas de salario antes de producir algo útil. En Django estándar produce desde el día 3-4.

### Cada funcionalidad nueva cuesta más caro

| Funcionalidad | Proyecto actual | Django estándar |
|--------------|:--------------:|:--------------:|
| ABM de un catálogo (ej: tipo de trámite) | 2-3 días + pruebas manuales | 2 horas (admin registra el modelo) |
| Reporte con filtros | 3-5 días + paginación manual | 1 día (ListView + DataTables) |
| Integrar envío de email | 2-3 días (entender sistema custom SMTP) | 1-2 horas (send_mail de Django) |
| Nueva pantalla con permisos | 2-3 días + pruebas de roles | 0.5-1 día (generic views + @permission_required) |

### Bugs y mantenimiento

El sistema actual tiene 0 pruebas automáticas (tests). Cualquier cambio puede romper una funcionalidad existente sin que nadie lo sepa hasta que un usuario lo reporta. La ausencia de paginación hace que el sistema se caiga si se cargan más de unos cientos de registros.

---

## 4. Riesgo para el proyecto a futuro

1. **Dependencia de personas específicas.** Solo los desarrolladores que ya estuvieron en el proyecto entienden cómo funciona. Si se van, el conocimiento se pierde. No se puede reemplazar a alguien con un desarrollador Django estándar del mercado.

2. **Contratación más difícil y cara.** Un desarrollador Django común cobra X. Un desarrollador que además entienda este sistema custom vale más caro... o no existe.

3. **Actualizaciones de Django bloqueadas.** Cada nueva versión de Django (5.2, 5.3) puede romper las modificaciones custom. El equipo va a tener que elegir entre quedarse en una versión vieja (insegura) o invertir semanas en adaptar el sistema custom.

4. **El sistema no escala.** La falta de paginación, los permisos vía archivos de texto (CSV) y los envíos de email sin cola de fondo garantizan problemas operativos cuando el sistema crezca.

---

## 5. Recomendación

Migrar las aplicaciones de negocio (`afiliaciones` y `geografico_genos`) a un proyecto Django limpio, eliminando las 35 modificaciones custom. El proyecto nuevo:

- Usa Django exactamente como lo enseñan los cursos y la documentación oficial
- Permite que cualquier desarrollador Django del mercado sea productivo en días
- Tiene tests desde el día 1
- Escala sin problemas a 190.000+ afiliados
- Se actualiza con cada nueva versión de Django sin sobresaltos

**Tiempo estimado de migración: ~3 semanas (11-17 días hábiles).**

**Ahorro estimado posterior:** cada nueva funcionalidad costará 2-10x menos que en el sistema actual, y el equipo podrá crecer sin depender de personas específicas.

---

*Documento gerencial. Para detalle técnico de las 35 modificaciones, ver informe_alteraciones_django.md.*
