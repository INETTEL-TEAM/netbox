# inettel × NetBox — Resumen claro del proyecto

> Documento de estado fácil de leer. Para el detalle técnico completo, ver
> [`INETTEL-MIGRATION-PLAN.md`](./INETTEL-MIGRATION-PLAN.md).
>
> **Fecha:** 2026-06-24 · **Estado:** los 4 pilares funcionando + despliegue por empresa probado.

---

## 1. ¿Qué estamos haciendo? (en una frase)

Llevar todo el producto **inettel** (gestión de red) a **NetBox**, aprovechando lo que NetBox
ya hace bien y añadiendo **3 plugins propios** para lo que le falta. Resultado: una plataforma
**centralizada, visual y multi-empresa**.

> Importante: inettel estaba **~85% en "mocks"** (sin datos reales). Esto **no es migrar datos**,
> es **rehacer el diseño de inettel sobre NetBox**, que es más sólido (permisos en servidor,
> jerarquía geográfica real, rack-elevation nativa, API/GraphQL, etc.).

---

## 2. Cómo verlo ahora mismo (2 minutos)

1. Servidor local: **http://localhost:8000**
2. Usuarios de prueba:

| Usuario | Contraseña | Qué ve |
|---|---|---|
| `admin` | `admin` | **Todo** (las 3 empresas) |
| `acme` | `acme` | Solo **Barcelona + Madrid** (empresa Acme) |
| `globex` | `globex` | Solo **Lisboa** (empresa Globex) |

3. Menús a probar (en el lateral, sección **Plugins**):
   - **inettel Geo → Drill-down Map** → mapa mundial con zoom; marcadores en rojo/naranja según incidentes.
   - **inettel Topology → Topology Graph** → grafo de red; botones **SPOFs** y **what-if**; vista **VLAN**.
   - **inettel ITSM** → Incidents · Change Requests · Maintenance · SLAs · Service Requests.
   - **DCIM → Sites → BCN-DC1** → pestañas **Incidents/Changes/Maintenance/Topology** + sus **racks** (vista 2D).

> Consejo: para probar `acme`/`globex` usa una ventana de **incógnito** (evita líos de cookies).

---

## 3. Lo que YA está hecho ✅

### Pilar 1 — Jerarquía geográfica y mapas
- Árbol **Mundo → Continente → País → Comunidad → Provincia → Site** (usando `Region` de NetBox).
- **Mapa interactivo** con zoom-in/zoom-out (MapLibre) y migas de pan.
- **Estado de salud por nivel**: cada región/site se pinta según la **peor incidencia activa**
  (P1 rojo, P2 naranja…), y la salud **sube** por el árbol (desde el mapa mundial ya ves dónde hay fuego).

### Pilar 2 — Multi-empresa (aislamiento)
- **1 instancia NetBox por empresa** (aislamiento duro) — ver despliegue en el punto 4.
- Dentro de una instancia: **sub-clientes como Tenants** con **permisos por tenant**.
- Al entrar, cada usuario **solo ve los datos de su empresa** (mapa, topología, dispositivos, ITSM).
  Comprobado: `acme` ve 14 nodos (BCN+MAD), `globex` 7 (LIS), `admin` todo.

### Pilar 3 — Exploración visual
- **Rack-elevation 2D nativa**: racks con sus equipos colocados (vista de armario).
- **Grafo de topología interactivo** (Cytoscape) derivado del cableado real de NetBox **sin Neo4j**:
  - **Path-trace** (ruta más corta A→B), **SPOF** (puntos únicos de fallo), **what-if** (simular caídas),
    **vista por VLAN**, y **anillos de incidente** sobre los nodos.
- **Cable-trace** nativo de NetBox sobre los enlaces.

### Pilar 4 — Flexibilidad del modelo
- **Custom Fields** y **Choice Sets** sembrados (EOL/EOS, dBm ópticos, BGP, drift, coordenadas…).
- Todo construido como **plugins** (la API estable de NetBox) → cambios ágiles sin tocar el core.

### Operaciones / ITSM (plugin `netbox_inettel_itsm`) — 8 modelos
| Modelo | Para qué |
|---|---|
| **Incident** | Incidencias P1–P4 con ciclo de vida |
| **ChangeRequest** | Cambios (RFC/CAB): tipo, riesgo, aprobación, ventana |
| **MaintenanceWindow** | Mantenimientos (impacto, ventana, ligado a un cambio) |
| **SLA** | Niveles de servicio (gold/silver/bronze, tiempos de respuesta) |
| **ServiceRequest** | Peticiones de servicio (acceso/provisión…) con flujo de aprobación |
| **Postmortem** | Análisis post-incidente (causa raíz, resolución, acciones), ligado a su Incident |
| **Runbook** | Procedimientos documentados (base de conocimiento), por categoría/rol |
| **OnCallSchedule + OnCallShift** | Rotaciones de guardia y turnos (primary/secondary/escalation) |

Cada modelo tiene UI completa, **API REST**, **GraphQL**, **búsqueda global**, y respeta permisos por tenant.
Además, **pestañas en Device, Site e Incident** muestran objetos relacionados (incidencias/cambios/
mantenimientos en Device/Site; postmortems en Incident), con botón "crear" pre-rellenado.

### Despliegue por empresa (producción) — carpeta `inettel-netbox/deploy/`
- **Imagen Docker** con los 3 plugins horneados (sobre `netboxcommunity/netbox`).
- **`provision.sh <empresa>`**: levanta un stack aislado (NetBox + worker + Postgres + Redis),
  migra, siembra la base y crea el Tenant. **`deprovision.sh`** para apagar/borrar.
- **Probado de verdad**: imagen construida + empresa "acme" levantada y plugins cargando en contenedor.

### Datos de demostración (3 ubicaciones)
- **BCN-DC1** (Barcelona), **MAD-DC1** (Madrid), **LIS-POP1** (Lisboa): cada una con 7 equipos
  cableados, VLANs, un rack con los equipos colocados, e incidentes de severidad variada.
- Comandos que lo crean (todos re-ejecutables sin duplicar):
  `inettel_bootstrap`, `inettel_demo_topology`, `inettel_demo_sites`, `inettel_demo_racks`, `inettel_demo_tenancy`.

### Calidad
- **284 tests en verde** + ruff limpio en los 3 plugins. Cobertura completa al estándar NetBox `add-model`:
  - ITSM: `test_models.py` (lógica/validaciones/gating) · `test_api.py` (REST CRUD/bulk/brief/**GraphQL** de los 10 modelos, con baseline de conteo de queries) · `test_filtersets.py` (filtros choice/FK/búsqueda).
  - Topología: `test_graph.py` (build_graph, path-trace, SPOF, what-if, VLAN, híbrido, anillos de incidente).
- GraphQL reestructurado al paquete estándar `graphql/{types,filters,schema}.py` (resolución de tipos/filtros por convención NetBox).
- Migraciones generadas por Django (no a mano).

---

## 4. Entorno local (dónde vive cada cosa)

| Cosa | Dónde |
|---|---|
| NetBox (código) | `/Users/annasabater/netbox` (v4.6.3) |
| Nuestros plugins | `/Users/annasabater/inettel-netbox/` (geo, topology, itsm) |
| Kit de despliegue | `/Users/annasabater/inettel-netbox/deploy/` |
| Entorno Python | `~/.venv/netbox` · BD `netbox` · Redis (vía Homebrew) |
| Arrancar | `source ~/.venv/netbox/bin/activate && cd netbox/ && python manage.py runserver` |

---

## 5. Lo que FALTA ⏳ (todo opcional a partir de aquí)

### ITSM
- [x] **On-call / escalado** (rotaciones + turnos) — hecho.
- [x] **Runbooks** (procedimientos documentados) — hecho.
- *(Todos los modelos ITSM del plan están completos.)*

### Topología (extras)
- [ ] Vista **híbrida** físico+VLAN superpuesta.
- [ ] **Exportar** el grafo (PNG/JSON).

### Producción "de verdad"
- [ ] **Reverse-proxy** (nginx/Traefik) con **TLS por subdominio** (`empresa.inettel.com`).
- [ ] Secretos en gestor (**Vault/SOPS**) en vez de ficheros `.env`.
- [ ] Estrategia de **backups** por empresa y actualización de imagen por oleadas (documentada, falta ejecutar).

### Fundación (menores)
- [x] **Manufacturer / DeviceRole / Platform** base en el bootstrap — hecho (7/8/7), idempotente.
- [x] **GraphQL** para los modelos ITSM — hecho (los 6 modelos), schema verificado.

### Integraciones externas (cuando toque)
- [ ] Conectar métricas/logs (Grafana/Loki/Prometheus) y feeds CVE vía Event Rules + Webhooks.

---

## 6. Notas / cosas a tener en cuenta

- Hay un **4º plugin, `netbox_inettel_engine`** (trabajo en paralelo, integra un servicio "netlab"
  en `:8800` y añade un dashboard de ciclo de vida IOS). **No forma parte de este plan**; está
  respetado e intacto.
- Cuando hubo que **migrar la BD**, se generó con `makemigrations` (nunca a mano), según las reglas del repo.
- Gotcha del despliegue: la imagen base de NetBox 5.x usa **Python 3.14 + `uv`** (sin `pip`), ya resuelto en el Dockerfile.

---

## 7. Próximo paso (cuando retomemos)

Recomendado: **Postmortem ligado a Incident** (aporta relación entre modelos y cierra el ciclo de
incidencias). Alternativas: on-call/escalado, o la plantilla de reverse-proxy + TLS para producción real.
