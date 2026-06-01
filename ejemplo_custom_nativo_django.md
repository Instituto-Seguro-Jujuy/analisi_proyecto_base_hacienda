# Ejemplo: `SolicitudLista` + `SolicitudAjax` → `ListView` con `paginate_by=25`

## Antes: Código actual (proyecto base)

### Vista `SolicitudLista` + `SolicitudAjax` (~60 líneas)

```python
# views.py — DOS vistas separadas
class SolicitudLista(View):
    template_name = 'afiliaciones/solicitud_lista.html'

    def get(self, request):
        context = {
            'titulo': 'Gestión de Solicitudes',
            'estados': Solicitud.Estado.choices,
            'usuarios': User.objects.filter(
                id__in=Solicitud.objects.values_list('usuario_id', flat=True)
            ).distinct()
        }
        return render(request, self.template_name, context)


class SolicitudAjax(View):
    def dispatch(self, request, *args, **kwargs):
        return self._data(request)

    def _data(self, request):
        data_request = request.POST if request.method == 'POST' else request.GET

        filtro_estado = data_request.get('estado', '').strip()
        filtro_usuario = data_request.get('usuario', '').strip()
        filtro_busqueda = data_request.get('busqueda', '').strip()

        qs = Solicitud.objects.select_related('persona', 'tipo_afiliado', 'tipo_tramite')

        if filtro_estado:
            qs = qs.filter(estado=filtro_estado)
        if filtro_usuario:
            qs = qs.filter(usuario_id=filtro_usuario)
        if filtro_busqueda:
            qs = qs.filter(
                Q(persona__per_nro_doc__icontains=filtro_busqueda) |
                Q(persona__apellido__icontains=filtro_busqueda) |
                Q(persona__nombre__icontains=filtro_busqueda)
            )

        data = []
        for s in qs:   # ← CARGA TODAS LAS SOLICITUDES SIN PAGINACIÓN
            data.append({
                'id': s.id,
                'per_cui': s.persona.per_cui,
                'solicitante': s.persona.apellido + ' ' + s.persona.nombre,
                'estado': s.get_estado_display(),
                # ... más campos
            })
        return JsonResponse(data, safe=False)  # ← VUELCA TODO EN UNA SOLA RESPUESTA
```

### URLs

```python
# urls.py — DOS rutas
path('solicitudes/', SolicitudLista.as_view(), name='solicitud_lista'),
path('solicitudes/ajax/', SolicitudAjax.as_view(), name='solicitud_ajax'),
```

### Template `solicitud_lista.html` (~445 líneas)

La plantilla tiene DataTables con JS que llama al endpoint AJAX. Sin paginación real: DataTables pinta el HTML pero el servidor mandó **todas las filas juntas** en una sola respuesta JSON.

---

## Después: Django `ListView` con paginación

### Una sola vista (~20 líneas)

```python
from django.views.generic import ListView

class SolicitudLista(ListView):
    model = Solicitud
    template_name = 'afiliaciones/solicitud_lista.html'
    context_object_name = 'solicitudes'
    paginate_by = 25                     # ← UNA LÍNEA Y YA TIENE PAGINACIÓN

    def get_queryset(self):
        qs = Solicitud.objects.select_related('persona', 'tipo_afiliado', 'tipo_tramite')

        # Filtros vienen por GET (limpio, sin POST para leer)
        busqueda = self.request.GET.get('busqueda', '').strip()
        estado = self.request.GET.get('estado', '').strip()
        usuario = self.request.GET.get('usuario', '').strip()

        if busqueda:
            qs = qs.filter(
                Q(persona__per_nro_doc__icontains=busqueda) |
                Q(persona__apellido__icontains=busqueda)
            )
        if estado:
            qs = qs.filter(estado=estado)
        if usuario:
            qs = qs.filter(usuario_id=usuario)
        return qs

    def get_context_data(self, **kwargs):
        ctx = super().get_context_data(**kwargs)
        ctx['estados'] = Solicitud.Estado.choices
        ctx['titulo'] = 'Gestión de Solicitudes'
        return ctx
```

### URLs

```python
# urls.py — UNA sola ruta
path('solicitudes/', SolicitudLista.as_view(), name='solicitud_lista'),
```

### Template (~50 líneas de tabla + paginación nativa)

```html
<div class="container-fluid">
  <div class="card">
    <div class="card-body">

      <!-- Filtros -->
      <form method="get" class="row g-3 mb-3">
        <div class="col-lg-4">
          <input type="text" name="busqueda" class="form-control"
                 placeholder="Buscar por apellido, nombre o DNI..."
                 value="{{ request.GET.busqueda }}">
        </div>
        <div class="col-lg-3">
          <select name="estado" class="form-control">
            <option value="">Todos los estados</option>
            {% for value, label in estados %}
            <option value="{{ value }}"
              {% if request.GET.estado == value %}selected{% endif %}>
              {{ label }}
            </option>
            {% endfor %}
          </select>
        </div>
        <div class="col-lg-2 d-flex align-items-end">
          <button type="submit" class="btn btn-primary me-1">
            <i class="fa fa-search"></i> Filtrar
          </button>
          <a href="." class="btn btn-outline-secondary">
            <i class="fa fa-eraser"></i>
          </a>
        </div>
      </form>

      <!-- Tabla server-side -->
      <div class="table-responsive">
        <table class="table table-striped table-bordered">
          <thead>
            <tr>
              <th>CUIL</th>
              <th>Solicitante</th>
              <th>F. Ingreso</th>
              <th>T. Afiliado</th>
              <th>T. Trámite</th>
              <th>Estado</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            {% for s in solicitudes %}
            <tr>
              <td>{{ s.persona.per_cui }}</td>
              <td>{{ s.persona.apellido }}, {{ s.persona.nombre }}</td>
              <td>{{ s.fecha_ingreso|date:"d/m/Y" }}</td>
              <td>{{ s.tipo_afiliado.nombre|default:"—" }}</td>
              <td>{{ s.tipo_tramite.nombre|default:"—" }}</td>
              <td>
                <span class="badge"
                  style="background:{% if s.estado == 'A' %}#198754{% elif s.estado == 'P' %}#dc3545{% else %}#e5ce00{% endif %}; color:white;">
                  {{ s.get_estado_display }}
                </span>
              </td>
              <td>
                <a href="{% url 'afiliaciones:solicitud_form' s.pk %}"
                   class="btn btn-sm btn-outline-orange">
                  <i class="fa fa-eye"></i>
                </a>
              </td>
            </tr>
            {% empty %}
            <tr><td colspan="7" class="text-center">No hay solicitudes.</td></tr>
            {% endfor %}
          </tbody>
        </table>
      </div>

      <!-- Paginación nativa de Django -->
      {% if is_paginated %}
      <nav>
        <ul class="pagination justify-content-center mb-0">
          {% if page_obj.has_previous %}
          <li class="page-item">
            <a class="page-link" href="?page=1{% if request.GET.busqueda %}&busqueda={{ request.GET.busqueda }}{% endif %}{% if request.GET.estado %}&estado={{ request.GET.estado }}{% endif %}">&laquo;</a>
          </li>
          <li class="page-item">
            <a class="page-link" href="?page={{ page_obj.previous_page_number }}{% if request.GET.busqueda %}&busqueda={{ request.GET.busqueda }}{% endif %}{% if request.GET.estado %}&estado={{ request.GET.estado }}{% endif %}">Anterior</a>
          </li>
          {% endif %}

          <li class="page-item active">
            <span class="page-link">
              Página {{ page_obj.number }} de {{ page_obj.paginator.num_pages }}
              ({{ page_obj.paginator.count }} solicitudes)
            </span>
          </li>

          {% if page_obj.has_next %}
          <li class="page-item">
            <a class="page-link" href="?page={{ page_obj.next_page_number }}{% if request.GET.busqueda %}&busqueda={{ request.GET.busqueda }}{% endif %}{% if request.GET.estado %}&estado={{ request.GET.estado }}{% endif %}">Siguiente</a>
          </li>
          <li class="page-item">
            <a class="page-link" href="?page={{ page_obj.paginator.num_pages }}{% if request.GET.busqueda %}&busqueda={{ request.GET.busqueda }}{% endif %}{% if request.GET.estado %}&estado={{ request.GET.estado }}{% endif %}">&raquo;</a>
          </li>
          {% endif %}
        </ul>
      </nav>
      {% endif %}

    </div>
  </div>
</div>
```

---

## Lo que cambia en resultados

| Aspecto | Antes (proyecto base) | Después (Django limpio) |
|---------|----------------------|------------------------|
| **Líneas de código** | ~60 (2 vistas) + ~445 (template JS) | ~20 (1 vista) + ~70 (template HTML) |
| **Archivos involucrados** | `views.py` + `urls.py` + `template.html` | `views.py` + `urls.py` + `template.html` |
| **Paginación** | ❌ No existe. DataTables la simula en frontend, servidor envía TODO | ✅ 1 línea: `paginate_by = 25`. Servidor envía 25 por página |
| **Carga de datos** | `SELECT * FROM solicitud` → JSON de 190K filas → navegador se cuelga | `SELECT ... LIMIT 25` → HTML de 25 filas → rápido |
| **Mantenimiento** | Cambiar filtros: editar vista + template + JS | Cambiar filtros: editar `get_queryset()` solamente |
| **JS en template** | 180+ líneas de JS (DataTables, modals, filtros) | Cero JS (DataTables opcional para adornos) |
| **Curva de aprendizaje** | Tiene que entender `SolicitudAjax(View)` + protocolo DataTables | Es `ListView`, lo ve en cualquier tutorial de Django |
| **URLs** | 2 paths (lista + ajax) | 1 path (lista con paginación incluida) |
| **CSRF** | POST con `csrfmiddlewaretoken` para leer datos | GET (idempotente, sin CSRF) |
| **Navegación** | Sin URLs únicas por página (todo AJAX) | `?page=N`, compartible, bookmarkeable, back button funciona |
