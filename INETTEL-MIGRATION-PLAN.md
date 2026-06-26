# Plan maestro — Integración de **inettel** en **NetBox**

> Documento único de referencia: de dónde partimos, qué se ha analizado, qué se ha decidido y qué pasos quedan por hacer (y cómo).
>
> - **Autor:** Anna Sabater + Claude (arquitecto de soluciones / NetBox)
> - **Fecha:** 2026-06-22
> - **Estado global:** Análisis ✅ · Decisiones ✅ · Entorno local ✅ · Fase 0 Fundación ✅ · Fase 1 Geo ✅ · Fase 2 Topología (grafo+path+SPOF+what-if+VLAN+filtros) ✅ · Fase 3 ITSM `Incident`+`ChangeRequest`+`MaintenanceWindow`+`SLA` ✅ +`ServiceRequest` · Pilar 2 scoping por tenant ✅ · Fase 0 deploy por empresa ✅ · Construcción ⏳ (en curso)
> - **Ubicación del repo NetBox de trabajo:** `/Users/annasabater/netbox` (NetBox 4.6.3, Django 6, Python 3.12)
> - **Plugins inettel:** `/Users/annasabater/inettel-netbox/` (workspace de paquetes; `netbox_inettel_geo` y `netbox_inettel_topology` instalados en editable).

---

## 0. TL;DR (resumen ejecutivo)

1. **Esto NO es una migración de datos.** `inettel-app` está ~85% en *mocks* (sin datos reales en producción). Es un **re-plataformado del *diseño* de inettel sobre NetBox**.
2. **NetBox ya hace mejor varios pilares** que el estado actual de inettel: jerarquía geográfica multinivel (`Region` recursiva), permisos en servidor, rack-elevation 2D, custom fields.
3. **Lo único maduro y sin equivalente** en NetBox es el **grafo lógico de topología** (path-trace / what-if / SPOF) → se porta como plugin.
4. **Decisiones tomadas:** (1) **una instancia NetBox por empresa**, (2) **frontend híbrido** (NetBox+plugins + 2-3 pantallas "wow" en Next.js), (3) **Operaciones/NOC se reconstruyen dentro de NetBox**.
5. **Construcción por fases:** Fundación → Geo → Topología → ITSM/Operaciones → OSP/Reports → App "wow".

---

## 1. Punto de partida — Entorno local NetBox (YA MONTADO ✅)

Todo esto ya está instalado y funcionando en este Mac (`darwin`, Homebrew, Docker disponible).

### 1.1 Software instalado (Homebrew)
- `python@3.12` (NetBox usa 3.12 como base; el sistema tenía 3.14)
- `postgresql@16` (keg-only → binarios en `/opt/homebrew/opt/postgresql@16/bin`)
- `redis`

### 1.2 Servicios (arrancan solos al iniciar sesión)
```bash
brew services start postgresql@16
brew services start redis
```

### 1.3 Base de datos
- DB: `netbox` · rol: `netbox` · password: `netbox` · host: `localhost:5432`
- Creada con `createdb -O netbox netbox` (rol con LOGIN, owner de la BD)

### 1.4 Entorno Python
- venv: `~/.venv/netbox` (Python 3.12.13)
- Dependencias: `pip install -r requirements.txt` (incluye `psycopg[c]` compilado contra `libpq` de postgresql@16)

### 1.5 Configuración
- Fichero: `netbox/netbox/configuration.py` (**gitignored**, solo local)
- Contenido clave: `ALLOWED_HOSTS = ['localhost','127.0.0.1','*']`, `CSRF_TRUSTED_ORIGINS = ['http://localhost:8000','http://127.0.0.1:8000']`, `SECRET_KEY` generado, `DEBUG = True`, `DEVELOPER = True`
- DB y Redis (db 0 tasks / db 1 caching) apuntando a localhost.

### 1.6 Migraciones y superusuario
- `python manage.py migrate` ✅
- Superusuario: **`admin` / `admin`** (email `admin@example.com`)

### 1.7 Arrancar / parar el servidor
```bash
source ~/.venv/netbox/bin/activate
cd /Users/annasabater/netbox/netbox
python manage.py runserver           # http://localhost:8000
```
- **Acceso:** http://localhost:8000 · usuario `admin` · pass `admin`
- **Aviso CSRF (403 "CSRF token from POST incorrect"):** suele ser una *cookie antigua* en `localhost:8000`. Solución: ventana de **incógnito** o borrar cookies de `localhost:8000` y recargar el login. Accede siempre por el **mismo host** con el que cargaste la página. Si usas el navegador integrado de VS Code / puerto reenviado, el origin cambia → usa el navegador normal o añade esa URL a `CSRF_TRUSTED_ORIGINS`.
- Avisos no fatales conocidos: `staticfiles.W004` (falta `project-static/docs`, no afecta) y `API_TOKEN_PEPPERS not defined` (tokens v2 API deshabilitados; opcional).

---

## 2. Punto de partida — Repos de inettel (origen del análisis)

| Ruta | Qué es | Relevancia |
|---|---|---|
| `/Users/annasabater/Inettel/Documents/inettel-app` | **Producto real** (monorepo Turborepo/pnpm; `apps/web` = la app) | **Alta** — fuente de verdad del diseño |
| `/Users/annasabater/Inettel/Documents/inettel` | Web de **marketing** (Next.js + formularios → email, sin BD) | Baja — solo referencia de marca |
| `/Users/annasabater/Inettel` (raíz) | POC Python `neomodel`/Neo4j (`test_neomodel.py`) | Desechable |

**Stack de inettel-app:** Next.js 16 (App Router, Turbopack) + React 19 + Tailwind 4 (sin shadcn; Radix crudo + primitivas propias) + Drizzle/PostgreSQL 16 (~90 tablas) + **Neo4j 5** (topología) + NextAuth v5 + RLS. Librerías visuales reales: `maplibre-gl`, `react-map-gl`, `@xyflow/react` (React Flow), `recharts`, `next-intl` (ES/EN/CA). **No hay `three.js` (cero 3D).**

---

## 3. Hallazgos del análisis (condensado)

### 3.1 Estado real
- **~85% mocks.** Esquema y "chrome" UI existen; la mayoría de páginas consumen `src/lib/mocks/*` y los endpoints de escritura son *stubs*. **No hay datos productivos que migrar.**
- **Assets portables reales:** el esquema de datos, la matriz RBAC, las máquinas de estado de ciclo de vida, el audit hash-chain (SHA-256) y, sobre todo, las **consultas de grafo de topología** (Neo4j: path-trace, SPOF, what-if).

### 3.2 Modelo de datos (Drizzle, ~90 tablas, multi-origen)
Dominios: **Tenancy/Identity**, **Inventario (DCIM)**, **IPAM**, **OSP/fibra**, **Servicios/WAN/Circuitos**, **Operaciones**, **Cambios (RFC/CAB)**, **Peticiones**, **Reports**, **Admin**, **Multi-origin/Lifecycle/Audit/Vault/Outbox**.
- Todas las tablas de dominio: UUID PK, `tenant_id` FK, `deleted_at` (soft-delete), `extended_fields` JSONB, `source_system_id`.

### 3.3 Jerarquía geográfica (⚠️ clave para el pilar 1)
**El modelo de inettel es PLANO**: `Site` (con `country`/`city` como texto y `lat`/`lng`) con `parent_site_id` (1 nivel) → `tech_room` (sala) → `rack` → `device` → `module`/`port`.
- **NO existen** tablas de continente/país/comunidad/comarca/provincia. El "drill-down mundial" **no está implementado** (mock + un SVG placeholder que admite "MapLibre deferred").
- **NetBox `dcim.Region` es un árbol recursivo de profundidad arbitraria → mejor base que la de inettel.**

### 3.4 Multi-tenancy (pilar 2)
- Diseño: Postgres **RLS** + `tenant_id` + infra **una stack Docker por empresa** (`provision.sh`: BD/Neo4j/Redis/MinIO/subdominio/TLS propios).
- **Realidad auditada:** **78 fugas cross-tenant + 1 IDOR**; la app conecta con rol *superuser* que **bypassa RLS**; RBAC aplicado **solo en UI** (ningún endpoint comprueba permisos en servidor).
- NetBox `Tenant` es **solo una etiqueta**, no aísla. → La decisión "1 instancia por empresa" resuelve esto de verdad.

### 3.5 RBAC
- 8 roles primarios + 5 secundarios, permisos `module:resource:action` con overrides `+/-`. **Enforcement solo en UI** (gap crítico de inettel). NetBox aplica ObjectPermissions **en servidor** por defecto → cierra el gap.

### 3.6 Visual (pilar 3) — madurez real
| Feature | inettel | NetBox |
|---|---|---|
| Mapa mundial drill-down | **No existe** (mock/SVG placeholder) | Solo pin Leaflet por site → **construir plugin** |
| Rack-elevation | Real pero DOM/Tailwind, inferior | **Nativa SVG, superior → usar la de NetBox** |
| Sala / pasillo frío-caliente | **No existe** | No existe → greenfield si se requiere |
| Port strip / grid de puertos | Real + API | Listas de interfaces → portar widget (valor medio) |
| Cableado / fibra (diagrama) | **No existe** (tablas) | **Cable trace nativo** |
| **Grafo lógico de topología** | **Lo más maduro** (React Flow + Neo4j) | **No existe → portar como plugin (la joya)** |
| Dashboards/KPIs/charts | recharts + bespoke | Dashboard configurable → portar selectivo |
| 3D | **No existe (cero three.js)** | No existe |

### 3.7 Infra
- Postgres + Neo4j + Redis + MinIO + Loki + Prometheus + Mimir + Grafana (dev compose). Prod: 1 stack por empresa (`provision.sh`) — pero **no hay Dockerfile de la app** comiteado y **no hay CI en inettel-app**.

### 3.8 Deuda/bugs heredados a NO repetir
- `bcrypt.compare` con argumentos invertidos (auth rota).
- Backdoor de dev: password `'dev'` → sesión `super_admin`.
- `signIn` callback es un stub (sin checks de seguridad).
- **78 fugas cross-tenant + 1 IDOR** (RLS bypass por rol superuser + GUCs en transacción distinta a la query).
- Secretos reales comiteados (`apps/web/.env.local`).
- **Cero tests**; sin CI/CD en el producto.

---

## 4. Decisiones tomadas

| # | Decisión | Elección | Implicación |
|---|---|---|---|
| 1 | **Multi-tenancy** | **1 instancia NetBox por empresa** | Aislamiento real a nivel infra; dentro de cada instancia NO hace falta constraint de tenant por modelo. El `Tenant` nativo queda libre para los **sub-clientes** de esa empresa (patrón `client_tenant_id`). |
| 2 | **Frontend** | **Híbrido** | NetBox + plugins (islas React) para el 95%; app Next.js externa solo para 2-3 pantallas "wow" (mapa mundial, NOC wall, dashboard ejecutivo) sobre API REST/GraphQL. |
| 3 | **Operaciones/NOC/ITSM** | **Reconstruir dentro de NetBox** | Plugin ITSM/NOC completo (RFC/CAB, peticiones, SLA, incidencias, on-call, mantenimientos, postmortems, CVE). Es el workstream más pesado. |

---

## 5. Arquitectura objetivo

- **Fundación NetBox 4.6 nativo:** DCIM/IPAM/Tenancy/Circuits/Wireless/VPN + Custom Fields + Tags + Event Rules/Webhooks. Cubre ~70% del modelo sin escribir casi código.
- **Una instancia por empresa:** misma imagen Docker (con plugins horneados), config por empresa, tras nginx con subdominio + TLS (adaptar `provision.sh` de inettel a NetBox).
- **Plugins propios** para los gaps de valor.
- **Topología sin Neo4j por instancia:** derivar el grafo de los `Cable`/`Interface`/LLDP de NetBox y resolver path-trace/SPOF con `networkx` en Python (menos ops, NetBox ya es SoT del cableado).
- **Operaciones externas que NO se reconstruyen** (Loki/Prometheus/Grafana): se **embeben/consultan** desde vistas de plugin; alertas vía Event Rules + Webhooks.
- **App Next.js "wow"** sobre la API de NetBox (token/SSO), reutilizando marca inettel.

### 5.1 Portfolio de plugins
| Plugin (paquete) | Cubre | Peso |
|---|---|---|
| `netbox_inettel_geo` | Mapa `Region`→`Site` con drill-down + estado agregado por nivel (pilar 1) | Ligero |
| `netbox_inettel_topology` | Grafo lógico (React Flow) + path-tracer + what-if/SPOF (`networkx`) | Medio |
| `netbox_inettel_itsm` | RFC/CAB, peticiones, SLA, incidencias, on-call, mantenimiento, postmortems, CVE/compliance | **Pesado** |
| `netbox_inettel_osp` | Fibra: hilos/tubos/empalmes/puntos técnicos/rutas | Medio |

> Posible 5º conjunto: bundle de **Custom Fields/Choice Sets** (no es plugin, es config/migración de datos) para `extended_fields`, dBm ópticos, BGP, DNS, EOL/EOS, drift, etc.

---

## 6. Mapeo inettel → NetBox

### 6.1 NATIVO (≈70% — solo configurar)
| inettel | NetBox nativo |
|---|---|
| `tenants` | `tenancy.Tenant` (+ `TenantGroup`); `client_tenant_id` → sub-tenants |
| **Jerarquía geográfica** (a construir) | **`dcim.Region`** recursiva = Continente→País→CCAA→Comarca/Provincia; `sites.type` → **`dcim.SiteGroup`** |
| `sites` | `dcim.Site` (`region`, `group`, `latitude`/`longitude`, `physical_address`, `tenant`) |
| `tech_rooms` | `dcim.Location` (jerárquica dentro del site) |
| `racks` | `dcim.Rack` (+ `RackRole`, `RackType`) |
| `devices` | `dcim.Device`; vendor→`Manufacturer`, vendor+model→`DeviceType`, categoría→`DeviceRole`, OS→`Platform` |
| `device_modules` | `dcim.Module` + `ModuleType` |
| `device_ports` | `dcim.Interface` (+ `FrontPort`/`RearPort` para patch) |
| `cable_runs`/`patch_panels` | `dcim.Cable` + patch-panel como Device con puertos pass-through (cable trace nativo) |
| `wlcs`/`ssids`/`wireless_aps` | app `wireless`: `WirelessLAN(Group)`, `WirelessLink`; APs como `Device` |
| `vrfs`/RT | `ipam.VRF` + `ipam.RouteTarget` |
| `vlans`/fhrp | `ipam.VLAN`(+`VLANGroup`) + `ipam.FHRPGroup` |
| `subnets` (árbol CIDR) | `ipam.Prefix`/`Aggregate`/`RIR` + `ipam.ASN` |
| `ip_assignments`/`ip_reservations` | `ipam.IPAddress` / `ipam.IPRange` |
| `tunnels` | app `vpn`: `Tunnel`, `IKEPolicy`, `IPSecPolicy/Profile` |
| `wan_carriers`/`wan_circuits` | `circuits.Provider` / `circuits.Circuit` + `CircuitType` + `CircuitTermination` |
| `tags`, `extended_fields`, `master_catalogs` | **Tags**, **Custom Fields**, **Choice Sets** |
| `audit_events`(parcial), `lifecycle_state_changes` | **Changelog (ObjectChange)** + **Journaling** |
| `notification_rules`, webhooks | **Event Rules + Webhooks** |
| `source_systems` | **Data Sources / sync** o ETL externo por REST |
| `scripts`/`script_executions` | **Custom Scripts + JobRunner** |

### 6.2 PLUGIN personalizado (gaps con valor)
`topology` (grafo lógico + path-trace + what-if/SPOF) · `geo` (mapa drill-down) · `itsm` (RFC/CAB + peticiones + SLA + incidencias + on-call) · `osp` (fibra a nivel de hilo) · audit hash-chain + export CEF/SIEM.

### 6.3 INTEGRAR externo (NO reconstruir core NMS)
Syslog (Loki/LogQL), métricas (Prometheus/Mimir/Grafana — ya en su stack), CVE feeds (NVD/CISA KEV), config-drift/backup. Conectar vía Event Rules + Webhooks; surface en vistas de plugin.

> **Motor de cambios por IA ("hablar con la red").** Es una integración externa de esta
> categoría (actúa sobre los equipos: propone → aplica con commit-confirmed → verifica →
> rollback). **Vive en `netlab`, no en NetBox** (NetBox es SSoT, no actuador, y corremos 1
> instancia por empresa). NetBox solo aporta la **UI** (plugin, isla React: mapa + chat +
> panel de diff + rollback) mediante una **vista-proxy** que reenvía a la API de netlab con
> un *service token*; el navegador habla solo con NetBox. Decisión completa (reparto de
> componentes, contrato de API, guardarraíles): **`netlab/docs/decisions/0002-ai-change-engine.md`**.

---

## 7. Roadmap por fases (pasos accionables)

> Convención: `[ ]` pendiente · `[~]` en curso · `[x]` hecho. Marca según avancemos.

### Fase 0 — Fundación + provisioning
- [x] NetBox 4.6 corriendo en local (ver §1).
- [x] **Script de fundación** idempotente → comando `python manage.py inettel_bootstrap` (en el plugin `netbox_inettel_geo`). Re-ejecutable (get_or_create, +0 al repetir).
  - [x] Árbol `Region` multinivel (Continente→País→Comunidad→Provincia; semilla Europa→España→Cataluña→provincias + Portugal/América). **19 regiones**.
  - [x] `SiteGroup` para `sites.type` (CPD, Oficina, PoP, Sala técnica, Tienda, Agente, Cajero, Internacional, Mástil, Repetidor, Subestación). **11 grupos**.
  - [x] `Manufacturer` / `DeviceRole` / `Platform` base — catálogo en `inettel_bootstrap` (7 fabricantes: Cisco/Juniper/Arista/MikroTik/Fortinet/Huawei/Generic · 8 roles con color · 7 plataformas enlazadas a su fabricante). Idempotente, verificado.
  - [x] **Custom Fields** (`extended_fields`: EOL/EOS, drift_status, source_system, dBm Tx/Rx, BGP origin ASN, DNS). **8 campos** sobre Device/Site/Interface/Prefix/IPAddress.
  - [x] **Choice Sets** = `master_catalogs` (semilla `inettel_drift_status`). **1 choice set**.
- [x] **Kit de despliegue por empresa** en `/Users/annasabater/inettel-netbox/deploy/`: `Dockerfile` (`netboxcommunity/netbox:v4.6-5.0.1` + los 3 plugins horneados), `docker-compose.yml` (stack aislado netbox+worker+postgres+redis, interpolado por `.env`), `provision.sh <slug> [empresa] [dominio] [puerto]` (genera secretos → up → migrate → `inettel_bootstrap` → crea Tenant → imprime URL/credenciales) y `deprovision.sh` (`--purge`).
- [x] **Build + smoke test REAL ejecutado** ✓: imagen `inettel-netbox:latest` (1.3 GB) construida; `provision.sh acme … 8081` levantó el stack (4 servicios *healthy*), aplicó migraciones, sembró fundación y creó el Tenant; rutas de plugins responden 302 y `settings.PLUGINS` = los 3 plugins. **Aprendizajes:** (1) el tag base real es `v4.6-5.0.1` (no existía mi `v4.6-3.4.0`); (2) la imagen 5.x **usa Python 3.14 y `uv`** (sin `pip` en el venv) → instalar con `uv pip install --python /opt/netbox/venv/bin/python …`.
- [x] **Estrategia de versionado**: misma imagen `inettel-netbox:<tag>` para todas las instancias; `NETBOX_VARIANT` fija la base 4.6; roll-forward por oleadas + backup de volumen `postgres` por empresa (documentado en `deploy/README.md`).
- [x] **Reverse-proxy + TLS por subdominio (Traefik)** — `deploy/proxy/docker-compose.yml` (edge Traefik v3 con Let's Encrypt TLS-ALPN + redirect HTTP→HTTPS, red compartida `inettel-edge`) + `deploy/docker-compose.proxy.yml` (override que añade labels Traefik `Host(<DOMAIN>)`/`certresolver=le` al servicio netbox). `provision.sh` escribe `DOMAIN`. Validado con `docker compose config` (labels renderizan OK). Mantiene intacto el modo standalone (puerto). Replica el paso nginx/TLS del `provision.sh` de inettel.
- [ ] (Prod real, opcional) Secretos en gestor (Vault/SOPS) en vez de `.env`; backups por empresa.

### Fase 1 — Geo + DCIM/IPAM visual (pilares 1 y 3 base)
- [x] Esqueleto plugin `netbox_inettel_geo` (`PluginConfig`, vista `GeoDashboardView`, navegación, template). Instalado (`pip install -e`) + activado en `PLUGINS`. Dashboard en **Plugins → inettel Geo** (`/plugins/inettel-geo/`), render 200 verificado.
- [x] Vista de **mapa drill-down** (isla MapLibre) sobre `Region`→`Site` (v0.2). Centroides de región en custom fields `inettel_latitude/longitude` (sembrados en el bootstrap); endpoint JSON `/plugins/inettel-geo/data/` sirve el árbol nivel a nivel con conteo agregado de sites/devices; mapa con breadcrumb + zoom in/out. Verificado: Mundo→Europa→España/Portugal.
- [x] **Mapa "vivo" — salud por incidentes** (v0.3): cada nodo (región/site) se anota con la peor severidad de **incidentes activos** (site directo o vía `device.site`); el marcador se recolorea por severidad + etiqueta `⚠N`, y la salud **se propaga hacia arriba** por el árbol de regiones. `views._site_health_map()` con import ITSM protegido (degrada si no está). Verificado: BCN-DC1 y Europa → ⚠2 P1 (rojo). *Cierra el pilar 1: "estado de los activos en cada nivel".*

### Pilar 2 — Multi-tenancy (scoping por tenant dentro de la instancia)
- [x] **Endpoints de plugin permission-aware**: geo (`GeoDataView`/`_site_health_map`) y topología (`build_graph`/`vlan_payload`/`_incident_annotations` + path/spof/whatif) aplican `.restrict(user, 'view')` → respetan ObjectPermissions de NetBox.
- [x] **Comando `inettel_demo_tenancy`** (idempotente): crea tenants **Acme** (BCN+MAD) y **Globex** (LIS), asigna `site.tenant`/`device.tenant`, y crea usuarios no-superuser `acme`/`globex` con **ObjectPermissions restringidos por tenant** (Site/Device por `tenant__slug`; Incident por `site|device.site → tenant`; Change/Maintenance por `site.tenant`).
- [x] **Verificado (aislamiento real en una instancia)**: admin ve los 3 sites/73 nodos; **acme** solo BCN+MAD/14 nodos; **globex** solo LIS/7 nodos — en mapa **y** topología. Complementa la decisión "1 instancia por empresa" (aislamiento duro) con scoping de sub-clientes dentro de cada instancia.
- [ ] (Opcional) Aplicar el mismo `.restrict()` a futuras vistas custom; constraints por `TenantGroup`; middleware para fijar tenant activo del usuario.
- [x] **Rack-elevation nativa con datos** (Pilar 3): comando `inettel_demo_racks` crea 1 rack/site (`<site>-R1`, tenant heredado del site) y coloca los 7 devices con `position`/`face=front`. Verificado: elevación SVG (`/api/dcim/racks/<pk>/elevation/?render=svg`) renderiza los devices; ficha de rack 200; scoping por tenant (admin 3 racks · acme 2 · globex 1; rack de otro tenant → 404). **Requirió `collectstatic`** (el render SVG lee `STATIC_ROOT/rack_elevation.css` del disco). Cable-trace nativo disponible sobre los cables sembrados.
- [x] **Datos demo multi-site**: comando `inettel_demo_topology` (BCN-DC1) + `inettel_demo_sites` (MAD-DC1 Madrid, LIS-POP1 Lisboa) — topología de 7 devices/site con nombres prefijados (componentes separados), VLANs e **incidentes de severidad variada** para lucir el mapa: BCN rojo (P1) · MAD naranja (P2) · LIS azul (P4). Idempotentes. Verificado: salud por site/región y filtro `?site=` aísla 7/7.

### Fase 2 — Topología (diferenciador)
- [x] Plugin `netbox_inettel_topology` (v0.1): grafo lógico **Cytoscape.js** (CDN, sin build step; *desviación justificada* de React Flow) + endpoints que construyen el grafo en vivo desde `dcim.CableTermination` (cachea `_device`), **sin Neo4j**. Instalado editable (trae `networkx`), activo en `PLUGINS`. Página en **Plugins → inettel Topology** (`/plugins/inettel-topology/`).
- [x] **Path-tracer** (A→B, `nx.shortest_path`) y **SPOF** (articulación + bridges, `nx.articulation_points`/`nx.bridges`). Endpoints `graph/`, `path/`, `spof/`. Verificado con datos demo: SPOF devices = core1/dist1/dist2; ruta acc1→acc3 = acc1→dist1→dist2→acc3 (3 hops).
- [x] **Seeder de demo** `python manage.py inettel_demo_topology` (idempotente): site `BCN-DC1` en región Barcelona (con coords → aparece también en el mapa geo) + 7 devices + 7 cables con SPOFs evidentes.
- [x] **What-if** (v0.2): marcar nodos como caídos (clic) y simular → reporta devices *severed* del fabric principal + nº de componentes resultantes (`nx.connected_components`). Endpoint `whatif/?down=`. Verificado: caer core1 aísla edge1; caer dist1 aísla acc1/acc2.
- [x] **Vista por VLAN** (v0.2): grafo de membresía L2 device↔VLAN (untagged/tagged), toggle Physical/VLAN. Seeder ampliado con 3 VLANs (servers/mgmt/users). Verificado: 7 devices + 3 VLAN nodes.
- [x] **Filtros por site/tenant en la UI** (v0.2): selector de site (solo sites con devices); endpoints aceptan `?site=`/`?tenant=`. Verificado.
- [x] **Topología ↔ ITSM** (v0.3): los nodos device con **incidentes activos** se resaltan con anillo de color = peor severidad (P1 rojo / P2 naranja…) + etiqueta `⚠N`; contador en las stats. `graph.py._incident_annotations()` consulta el plugin ITSM con import protegido (degrada si no está); excluye incidentes resueltos/cerrados. Verificado: core1 (P1) y dist1 (P2) marcados, acc1 (resuelto) no. *Conecta Fase 2 ↔ Fase 3 en una sola pantalla.*
- [x] **Pestaña Topology en el Site** (cohesión): `tabs.py` registra `SiteTopologyView` (`ViewTab`, `hide_if_empty`) en `dcim.Site` → embebe el grafo Cytoscape **filtrado al site** (anillos de incidente + botón SPOF) y enlace "Open full topology" que **pre-filtra** la página completa (lee `?site=`). Registrado en `PluginConfig.ready()`. Verificado: tab 200, scoping (acme→200 en su site, globex→404).
- [x] **Export del grafo** (v0.3): botones **Export PNG** (`cy.png` full/scale 2, fondo blanco) y **Export JSON** (descarga `lastData` del payload) en la página de topología; nombre de fichero según modo+scope. Verificado (render + presencia; descarga client-side).
- [x] **Vista híbrida físico+VLAN** (v0.4): modo `hybrid` (`graph.py:hybrid_payload`) que superpone aristas físicas (cables) + membresía VLAN (device↔VLAN, azul discontinuo, `etype`); botón Hybrid en la UI; path-trace/SPOF/what-if siguen disponibles (operan sobre el fabric físico). Verificado: BCN-DC1 → 7 devices + 3 VLANs, 7 aristas físicas + 10 VLAN.
- [ ] (Opcional) what-if multi-nodo con persistencia de escenario.

### Fase 3 — ITSM/Operaciones (lo más grande) — *en curso*
- [x] Plugin `netbox_inettel_itsm` (v0.1) + **modelo `Incident`** (P1–P4 + ciclo de vida) como `NetBoxModel` con superficie completa: model, choices, filterset (con `_id`), forms (model/filter/bulk-edit/import), table (ChoiceFieldColumn coloreado), views (list/detail/add/edit/delete/bulk), template de detalle, **REST API** (serializer/viewset/`NetBoxRouter`), **búsqueda global** (SearchIndex), navegación, migración generada por Django + aplicada, y tests (3 OK). Verificado: UI list/detail/add/import 200, API CRUD/filtros, search indexa. Permisos en servidor (ObjectPermissions) por defecto → cierra el gap RBAC de inettel.
- [x] **ChangeRequest (RFC/CAB)** (v0.2): `NetBoxModel` con type (normal/standard/emergency), risk (low/med/high), status (draft→…→implemented/rejected/…), site/tenant, requester/approver y **ventana de cambio** (`scheduled_start/end` con validación `clean()`). Superficie completa (filterset/forms/table/views/template/REST API/search/nav) + migración 0002 + tests (7 OK total). Verificado UI/API/filtros/search. RFC demo sembrada (Upgrade core1 firmware, scheduled, ventana +1d).
- [x] **MaintenanceWindow** (v0.3): ventanas de mantenimiento (status, impacto, site, **FK a ChangeRequest** → `maintenance_windows`, ventana start/end con validación). Superficie completa + API + search + nav. Migración 0003.
- [x] **SLA** (v0.3): niveles gold/silver/bronze con objetivos response/resolution (min), FK tenant. Superficie completa + API + search + nav. Migración 0003. Tests totales: **12 OK**. Verificado UI/API/filtros/search; demo: mantenimiento ligado a la RFC + SLA "Gold 24x7".
- [x] **Integración ITSM ↔ infraestructura** (v0.4): pestañas (`ViewTab` + `ObjectChildrenView`) en modelos del core — **Device** → tab *Incidents*; **Site** → tabs *Incidents / Change Requests / Maintenance*. Cada tab muestra los objetos ITSM relacionados con badge de conteo y **botón "Add" prefilado** (p. ej. `incident_add?device=<pk>&site=<pk>`). Registradas vía `tabs.py` importado en `PluginConfig.ready()` (antes de construir el URLconf de dcim). Verificado: tabs 200, badge, prefill y enlaces visibles en las fichas. *Esto es el "pegamento" que une los 4 pilares.*
- [x] **ServiceRequest (peticiones)** (v0.4): `NetBoxModel` con type (access/provisioning/information/change/other), priority (low/normal/high/urgent), status (new→approved→in_progress→fulfilled/rejected/cancelled), tenant/site, requester/assignee. Superficie completa + REST API (`/api/plugins/inettel-itsm/service-requests/`) + search + nav. Migración 0004. Tests totales: **15 OK**. Verificado UI/API/filtros/search. Completa el trío ITIL **Incident/Change/Request**.
- [x] **Postmortem** (v0.5): `NetBoxModel` ligado a `Incident` (FK `postmortems`), con status (draft/in_review/published), `occurred`, summary, root_cause, resolution, action_items. Superficie completa + REST API (`/api/plugins/inettel-itsm/postmortems/`, filtro `incident_id`) + search + nav + **pestaña "Postmortems" en la ficha de Incident** (badge + Add prefilado). Migración 0005. Tests totales: **18 OK**. Verificado UI/API/search/tab. *Cierra el ciclo de vida de la incidencia.*
- [x] **GraphQL para los 6 modelos ITSM** (v0.5): `graphql.py` con tipos `NetBoxObjectType` (FKs con anotaciones `lazy` a dcim/tenancy/users + intra-plugin), filtros `NetBoxModelFilter`, `Query` y `schema=[ITSMQuery]` (autodescubierto por el path `graphql.schema`). Verificado: schema construye + query real devuelve datos anidados (incident→site, postmortem→incident). **Ruff: all checks passed.** Ahora cada modelo cumple el checklist completo `add-model` (model+serializer+filterset+forms+table+views+**GraphQL**+tests).
- [x] **Runbook** (v0.6): `NetBoxModel` de procedimientos (category incident-response/maintenance/provisioning/recovery/general, status draft/published, `content`, scope opcional por `device_role`/`tenant`). Superficie completa + REST API + **GraphQL** + search + nav. Migración 0006. Tests totales: **21 OK**, ruff limpio. Verificado UI/API/GraphQL/search.
- [x] **On-call / escalado** (v0.7): `OnCallSchedule` (rotación, tenant, enabled) + `OnCallShift` (FK schedule, assignee, level primary/secondary/escalation, ventana start/end con `clean()`). Superficie completa ×2 + REST API + **GraphQL** + search + nav + **pestaña Schedule→Shifts**. Migración 0007. Tests totales: **24 OK**, ruff limpio. Verificado UI/API/GraphQL/tab. **→ Modelos ITSM del plan: COMPLETOS (8 modelos).**
- [x] **Custom Scripts + gating `requires_rfc`** (v0.8): helper reutilizable `automation.py` (`approved_changes`/`has_approved_change`/`require_approved_change` → cambios en estado approved/scheduled/in_progress, opcional por site/tenant) + Custom Script `GatedDecommission` (`scripts/gated_decommission.py`, loader en `SCRIPTS_ROOT/inettel.py`) que se **niega a actuar sin RFC aprobada** para el site del device (con override `force`). Tests del gate (3) + **verificado vía `runscript`**: LIS sin cambio → *Script failed (Blocked by requires_rfc)*; BCN core1 con cambio scheduled → *Authorised by change → decommissioning*. 27 tests OK, ruff limpio.
- [x] **CVE/compliance** (v0.9): modelo `ComplianceFinding` (FK Device, `cve_id`, severity crit/high/med/low/info, status open/ack/remediated/false-positive, `fixed_version`, detected). Superficie completa + REST API + **GraphQL** + search + nav (grupo *Security*) + **pestaña "Compliance" en Device** (badge + Add prefilado). Migración 0008. **30 tests OK**, ruff limpio. Verificado UI/API/GraphQL/tab/search (demo: CVE-2024-6387 en core1). *(feeds externos NVD/CISA: pendiente, se cargarían por API/script).*
- [x] **Event Rules + Webhooks** (v0.8): comando `inettel_seed_eventrules` (idempotente, `--url` configurable) crea un Webhook `inettel-alerts` + 2 Event Rules nativas de NetBox: `inettel-incident-alerts` (Incident create/update con severity∈P1/P2) y `inettel-change-emergency` (ChangeRequest type=emergency). Verificado: object_types/action_object/event_types correctos y `eval_conditions` (P1→sí, P3→no; emergency→sí, normal→no). Listo para apuntar a PagerDuty/Slack/Grafana.

> Notas de entorno (Fase 3): el rol Postgres `netbox` recibió `CREATEDB` (necesario para tests). Los plugins se añadieron también a `configuration_testing.py` (`PLUGINS`). La config principal incluye además `netbox_inettel_engine` (trabajo en paralelo, no cubierto por este plan).

### Fase 4 — OSP + Reports + Audit
- [ ] Plugin `netbox_inettel_osp` (hilos/tubos/empalmes/puntos técnicos/rutas).
- [ ] Reports programados (JobRunner + export templates).
- [ ] Audit **hash-chain** (tamper-evident) + export **CEF/SIEM**.

### Fase 5 — App Next.js "wow"
- [ ] App externa (reutiliza marca/UI inettel) sobre API NetBox: **mapa mundial**, **NOC wall**, **dashboard ejecutivo**.
- [ ] Auth: token/SSO de NetBox; respetar ObjectPermissions vía API.

---

## 8. Principios y convenciones (NetBox)

- `manage.py` vive en `netbox/`, no en la raíz. Tests con `NETBOX_CONFIGURATION=netbox.configuration_testing`.
- **No** escribir migraciones a mano → `python manage.py makemigrations`.
- Todo modelo nuevo: model + serializer + filterset (con `<field>_id` explícitos) + form + table + views (`register_model_view`) + URL + tests + docs.
- Ruff: line length 120, comillas simples.
- Toda la lógica nueva va en **plugins** (`netbox/netbox/plugins/` es la API estable); no tocar core.
- **No repetir la deuda de inettel:** permisos **en servidor** (no solo UI), nada de backdoors, secretos fuera de git, **tests desde el día 1**, CI.

---

## 9. Ficheros clave de inettel (referencia futura)

- **Esquema (fuente de verdad):** `inettel-app/apps/web/src/lib/db/schema/*.ts` (`inventory`, `ipam`, `osp`, `services`, `wan`, `changes`, `operations`, `requests`, `reports`, `admin`, `tenants`, `users`, `audit`, `vault`, `multiorigin`, `lifecycle`).
- **SQL/RLS:** `inettel-app/db/postgres/{migrations,rls-policies.sql,lifecycle-triggers.sql}`.
- **Topología (lo portable):** `apps/web/src/lib/db/neo4j.ts`, `queries/topology-graph.ts`; `db/neo4j/queries/{path-tracer,spofs,what-if,dependency-map}.cypher`.
- **RBAC/auth:** `apps/web/src/lib/rbac/{roles,permissions,check}.ts`; `lib/auth/*`.
- **Componentes visuales reales:** `components/widgets/topology-map.tsx` (MapLibre), `components/widgets/topology-flow.tsx` (React Flow), `components/viz/rack-diagram.tsx`, `components/inventario/port-manager-grid.tsx`.
- **Infra:** `inettel-app/infra/production/{provision.sh,deprovision.sh,template/docker-compose.yml}`.
- **Docs/auditoría:** `inettel-app/AUDIT-INETTEL-APP.md`, `inettel-app/docs/*`, `docs/TENANT-ISOLATION-AUDIT.md`.

---

## 10. Próximos pasos inmediatos (al retomar)

1. **Fase 0 — configuración de fundación:** script idempotente (Regiones + SiteGroups + Custom Fields + Choice Sets).
2. **Fase 1 — esqueleto `netbox_inettel_geo`:** `PluginConfig` + vista + navegación.

> Pendiente de confirmar el orden: **(1) fundación**, **(2) esqueleto plugin geo**, o **ambas a la vez**.

---

### Anexo — Comandos rápidos
```bash
# Arrancar entorno
brew services start postgresql@16 redis
source ~/.venv/netbox/bin/activate
cd /Users/annasabater/netbox/netbox
python manage.py runserver            # http://localhost:8000  (admin/admin)

# Tests (cuando los haya)
export NETBOX_CONFIGURATION=netbox.configuration_testing
python manage.py test --keepdb --parallel 4

# Lint
ruff check                            # desde la raíz del repo
```
