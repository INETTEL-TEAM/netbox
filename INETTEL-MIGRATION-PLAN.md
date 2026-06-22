# Plan maestro — Integración de **inettel** en **NetBox**

> Documento único de referencia: de dónde partimos, qué se ha analizado, qué se ha decidido y qué pasos quedan por hacer (y cómo).
>
> - **Autor:** Anna Sabater + Claude (arquitecto de soluciones / NetBox)
> - **Fecha:** 2026-06-22
> - **Estado global:** Análisis ✅ · Decisiones clave ✅ · Entorno local NetBox ✅ · Fase 0 Fundación ✅ · Fase 1 esqueleto plugin geo ✅ · Construcción ⏳ (en curso)
> - **Ubicación del repo NetBox de trabajo:** `/Users/annasabater/netbox` (NetBox 4.6.3, Django 6, Python 3.12)
> - **Plugins inettel:** `/Users/annasabater/inettel-netbox/` (workspace de paquetes; `netbox_inettel_geo` instalado en editable).

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
  - [ ] `Manufacturer` / `DeviceRole` / `Platform` base. *(pendiente — ampliar el bootstrap)*
  - [x] **Custom Fields** (`extended_fields`: EOL/EOS, drift_status, source_system, dBm Tx/Rx, BGP origin ASN, DNS). **8 campos** sobre Device/Site/Interface/Prefix/IPAddress.
  - [x] **Choice Sets** = `master_catalogs` (semilla `inettel_drift_status`). **1 choice set**.
- [ ] **Imagen Docker por empresa** (NetBox + plugins horneados) + **script de aprovisionamiento** por empresa (subdominio, BD, Redis, secrets, nginx, TLS) adaptando `provision.sh`.
- [ ] Definir estrategia de **versionado/actualización** de la imagen para N instancias.

### Fase 1 — Geo + DCIM/IPAM visual (pilares 1 y 3 base)
- [x] Esqueleto plugin `netbox_inettel_geo` (`PluginConfig`, vista `GeoDashboardView`, navegación, template). Instalado (`pip install -e`) + activado en `PLUGINS`. Dashboard en **Plugins → inettel Geo** (`/plugins/inettel-geo/`), render 200 verificado.
- [ ] Vista de **mapa drill-down** (isla MapLibre) sobre `Region`→`Site`, con estado agregado por nivel (conteo de sites/devices, salud). *(v0.2 — siguiente)*
- [ ] Validar **rack-elevation** y **cable trace** nativos cubren la exploración visual 2D del pilar 3.
- [ ] Import masivo de devices/racks/IPAM por REST o script (datos demo).

### Fase 2 — Topología (diferenciador)
- [ ] Plugin `netbox_inettel_topology`: isla **React Flow** + endpoints que construyen el grafo desde NetBox.
- [ ] **Path-tracer** (A→B) y **what-if/SPOF** con `networkx` (sin Neo4j).
- [ ] Vistas: lógica, VLAN, híbrida.

### Fase 3 — ITSM/Operaciones (lo más grande)
- [ ] Plugin `netbox_inettel_itsm`: modelos RFC/CAB (+ aprobaciones, change windows), peticiones (state machine), SLA/tiers, incidencias (P1–P4), on-call/escalado, mantenimientos, runbooks, postmortems.
- [ ] Scripts: Custom Scripts + JobRunner; gating `requires_rfc`.
- [ ] CVE/compliance como modelos de plugin; feeds externos.
- [ ] Event Rules + Webhooks hacia PagerDuty/Slack/Grafana.

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
