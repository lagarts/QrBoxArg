# QR BOX ARG — Documento de Seguimiento del Proyecto

> **Eslogan:** "Escaneá. Encontrá. Organizá."
> **Propósito:** mantener el contexto entre sesiones. Todo cambio de decisión se registra acá.
> **Regla de oro:** nada se implementa sin aprobación explícita del propietario.

---

## 0. Estado de conexiones (revisado 2026-11-10)

| Servicio | Estado | Detalle |
|---|---|---|
| Directorio de trabajo | ✅ Vacío | `QrBoxArg` sin archivos. No hay riesgo de sobrescritura |
| GitHub | ✅ Sesión activa | Cuenta `lagarts` (activa), `covacapp2-dev` (disponible). Sin repos creados |
| Supabase CLI | ❌ Sin sesión | `Invalid access token`. Requiere `supabase login` antes de crear proyecto |
| Vercel | ✅ Sesión activa | Team `sergios-projects-f010a49b`. Proyectos existentes: tecnocovac, stock-safe, intermedio, kiosko-go, barberturnos, alarma-bull, restova, covacweb, intermedio-lovat. **No hay proyecto QrBoxArg** |
| Tooling local | ✅ | git 2.52.0 / gh 2.95.0 / supabase 2.114.0 / vercel 62.2.0 / node 24.13.0 / npm 11.6.2 |

**Pendiente de aprobación:** elegir cuenta GitHub, crear repo, crear proyecto Supabase, crear proyecto Vercel.

---

## 1. Stack tecnológico

| Capa | Elección | Motivo |
|---|---|---|
| Framework | Next.js 15 (App Router) + TypeScript | RSC, Server Actions, óptimo en Vercel |
| Estilos | Tailwind CSS + shadcn/ui | Minimalista, rápido, consistente |
| DB / Auth / Storage | Supabase (PostgreSQL + RLS) | Multi-usuario, políticas de seguridad por fila |
| Hosting | Vercel | Integración nativa con Next.js |
| PWA | `manifest.json` + Service Worker propio | Instalable, sin dependencias pesadas |
| QR (render) | `qrcode` (SVG vectorial en servidor) | Sin deformación, escalable a cualquier tamaño |
| PDF | `@react-pdf/renderer` (o `pdf-lib`) | Control total de layout, QR vectorial |
| Escaneo (cámara) | `BarcodeDetector` nativo + fallback `html5-qrcode` | Sin instalación previa en el celular |
| Validación | `zod` | Schemas compartidos cliente/servidor |
| Email invitaciones | Resend (capa preparada, no activada hasta aprobación) | Invitaciones por correo |

**Identidad visual:** azul eléctrico `#0066FF`, azul marino `#0A1F44`, blanco `#FFFFFF`, grises neutros de apoyo. Tipografía principal: Inter (UI) + Space Grotesk (marca). El diseñador de etiquetas ofrecerá además: Inter, Montserrat, Oswald, Bebas Neue, Roboto, Playfair Display.

---

## 2. Arquitectura general

```
┌──────────────┐   escanea con cámara normal   ┌─────────────────────────┐
│  Celular del │ ────────────────────────────▶ │  Next.js  (Vercel)      │
│  usuario     │   https://dominio/q/CÓDIGO   │  /q/[code]  (pública)   │
└──────────────┘                               │  /app/*    (privada)    │
                                               │  /admin/*  (dueño)      │
                                               └───────────┬─────────────┘
                                                           │ Service Role (solo server)
                                               ┌───────────▼─────────────┐
                                               │  Supabase               │
                                               │  · Auth                 │
                                               │  · PostgreSQL + RLS     │
                                               │  · Storage (fotos)      │
                                               └─────────────────────────┘
┌──────────────┐  genera lotes + PDF  ┌──────────┴──────────┐
│ Admin/panel  │ ───────────────────▶ │  Server Actions /   │
│              │ ◀─── descarga PDF ── │  Route Handlers     │
└──────────────┘                      └─────────────────────┘
```

**Principios:**
1. El QR contiene **únicamente** una URL opaca: `https://DOMINIO/q/CODIGO`. Nunca datos privados.
2. Conocer la URL **no** da derecho a nada: la visibilidad la decide RLS + permisos.
3. Todo render de QR y de PDF ocurre en el **servidor** (calidad, seguridad, consistencia).
4. El Service Role nunca llega al navegador. El frontend solo usa `anon key` + RLS.

---

## 3. Estructura de carpetas

```
QrBoxArg/
├── SEGUIMIENTO-PROYECTO.md        ← este documento
├── package.json
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── .env.local                     ← (gitignored) claves reales
├── .env.example                   ← solo nombres de variables
├── public/
│   ├── manifest.json              ← PWA
│   ├── sw.js                      ← Service Worker
│   └── icons/                     ← iconos PWA (192, 512, maskable)
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx               ← landing pública
│   │   ├── q/[code]/page.tsx      ← ficha al escanear (pública/compartida)
│   │   ├── login/  signup/  auth/callback/
│   │   ├── app/                   ← área privada (menú de 7)
│   │   │   ├── page.tsx           ← dashboard
│   │   │   ├── inventarios/
│   │   │   ├── inventarios/[id]/
│   │   │   ├── etiquetas/nueva/   ← crear etiqueta
│   │   │   ├── etiquetas/lotes/   ← historial de lotes
│   │   │   ├── diseniador/        ← diseñador visual
│   │   │   ├── escanear/          ← cámara
│   │   │   ├── buscar/            ← buscador general
│   │   │   ├── compartir/         ← invitaciones y roles
│   │   │   └── cuenta/            ← perfil y plan
│   │   └── admin/                 ← panel exclusivo del dueño
│   │       ├── page.tsx           ← métricas
│   │       ├── usuarios/
│   │       ├── lotes/
│   │       ├── etiquetas/
│   │       ├── precios/
│   │       └── actividad/
│   ├── components/
│   │   ├── ui/                    ← shadcn
│   │   ├── qr/                    ← render QR, lector de cámara
│   │   ├── designer/              ← lienzo, propiedades, plantillas
│   │   ├── pdf/                   ← layout de impresión, preview
│   │   └── inventory/             ← tarjetas, filtros, búsqueda
│   ├── lib/
│   │   ├── supabase/              ← client, server, admin (service role)
│   │   ├── qr/                    ← generación de códigos únicos
│   │   ├── pdf/                   ← motor de PDF e impresión
│   │   ├── validation/            ← schemas zod
│   │   └── utils/
│   ├── types/                     ← tipos generados desde la DB
│   └── middleware.ts              ← protección de rutas /app y /admin
└── supabase/
    ├── migrations/                ← SQL versionado (nunca destructivo)
    └── seed.sql                   ← datos iniciales (planes, precios)
```

---

## 4. Modelo de base de datos

### 4.1 Tablas

**Perfiles y planes**
- `profiles` — `id (→ auth.users)`, `full_name`, `avatar_url`, `is_admin`, `created_at`
- `plans` — `id (free/premium...)`, `price_ars`, `max_labels`, `max_inventories`, `storage_mb`, `shared_users`, `features jsonb`, `active`
- `subscriptions` — `user_id`, `plan_id`, `status`, `mp_subscription_id`, `current_period_end`

**Inventarios y permisos**
- `inventories` — `id`, `owner_id`, `name`, `type`, `description`, `created_at`
- `inventory_members` — `inventory_id`, `user_id`, `role (owner|admin|editor|viewer)` — PK compuesta
- `invitations` — `id`, `inventory_id`, `email`, `role`, `token_hash`, `status (pending|accepted|revoked|expired)`, `expires_at`

**Etiquetas (núcleo del negocio)**
- `label_templates` — `id`, `user_id`, `name`, `category (box|kitchen|food|tool|custom)`, `width_cm`, `height_cm`, `layout jsonb` (posiciones, tipografías, colores, campos)
- `label_batches` (lotes) — `id`, `user_id`, `inventory_id`, `batch_name` ("Lote A"), `quantity`, `numbering_start`, `template_id`, `print_config jsonb`, `status`, `created_at`
- `labels` — **tabla crítica**
  - `id uuid PK`
  - `code text UNIQUE NOT NULL` ← identificador opaco e inmutable
  - `batch_id → label_batches`
  - `inventory_id → inventories` (nullable hasta asignar)
  - `box_number int` ← numeración visible (reinicia por lote)
  - `status ('generated' | 'active' | 'blocked')`
  - `visibility ('private' | 'shared' | 'public')`
  - `claimed_by uuid`, `activated_at`, `blocked_at`
  - `UNIQUE (batch_id, box_number)`

**Ficha digital**
- `items` — `label_id UNIQUE`, `name`, `description`, `category`, `location`, `quantity`, `unit`, `date_elaboration`, `date_packing`, `date_opening`, `date_expiration`, `lot_number`, `notes`, `custom_fields jsonb`, `updated_by`, `updated_at`
- `item_photos` — `item_id`, `storage_path`, `sort`, `is_cover`
- `item_history` — `item_id`, `actor_id`, `action`, `changes jsonb`, `created_at`

**Administración y comercio**
- `admin_activity_log` — `admin_id`, `action`, `target_type`, `target_id`, `detail jsonb`
- `label_stock` — tipo de plancha (50/100/200), material, `price_ars`, `stock`
- `sales` — `user_id`, `type (physical|subscription)`, `amount_ars`, `mp_payment_id`, `status`

### 4.2 Índices y restricciones

- `labels.code` → UNIQUE + índice (lookup de escaneo O(1))
- `labels(batch_id, box_number)` → UNIQUE
- `labels.status`, `labels.visibility`, `labels.inventory_id` → índices
- `items.label_id` → UNIQUE
- `inventory_members(inventory_id, user_id)` → UNIQUE
- `code` generado con `crypto.getRandomValues`, alfabeto Base32 (Crockford), 12 caracteres ≈ **60 bits de entropía** → no predecible, no enumerable
- CHECK en todos los enums de estado/rol
- FKs con `ON DELETE RESTRICT` en etiquetas (nunca se borra un QR activo por accidente) y `CASCADE` solo en fotos/historial

### 4.3 Relaciones (resumen)

```
profiles 1───* inventories
profiles 1───* inventory_members *───1 inventories
inventories 1───* labels 1───1 items 1───* item_photos
label_batches 1───* labels
label_templates 1───* label_batches
labels 1───* item_history
plans 1───* subscriptions *───1 profiles
```

---

## 5. Políticas RLS

| Tabla | Lectura | Escritura | Notas |
|---|---|---|---|
| `profiles` | dueño; admins | dueño; admins | |
| `plans` | público (lectura) | solo service role | |
| `subscriptions` | dueño; admins | solo service role | cobros vía backend |
| `inventories` | dueño + miembros | dueño + admin/editor | |
| `inventory_members` | miembros del inventario | solo owner/admin | auto-asignación al crear |
| `invitations` | owner/admin del inventario | owner/admin | token hasheado |
| `label_templates` | dueño | dueño | |
| `label_batches` | dueño; admins | dueño; admins | |
| `labels` | dueño, miembros con acceso, **admin** | owner/admin (update) | INSERT solo vía Server Action |
| `items` | según `visibility` + permisos del inventario | editor+ | |
| `item_photos` | igual que `items` | editor+ | |
| `item_history` | dueño/miembros | insert desde servidor | append-only |
| `admin_activity_log` | solo admins | solo service role | |
| `label_stock` / `sales` | dueño; admins | solo service role | |

**Escaneo público (clave):** no se crea policy SELECT anónima sobre `labels`. Se expone una función `SECURITY DEFINER`:

```sql
create function public.get_public_label(p_code text)
returns table (...) language sql security definer set search_path = public as $$
  select ... from labels l join items i on i.label_id = l.id
  where l.code = p_code
    and l.status = 'active'
    and l.visibility = 'public';
$$;
```

- `private` → 404 (no se revela que existe)
- `shared` → solo si el escáner está autenticado y es miembro/invitado
- `public` → solo campos marcados como públicos; el historial privado jamás se incluye

**Regla de seguridad:** conocer la URL nunca alcanza para apropiarse de una etiqueta. La reclamación exige sesión + prueba de compra/código de activación.

---

## 6. Flujo de generación de QR

```
1. Admin define: tamaño etiqueta, plantilla, cantidad (50/100/200/personalizada),
   numeración (inicia en 1, reinicia por lote), configuración de impresión.
2. Server Action (autenticada + autorizada):
   a. genera N códigos: crypto.getRandomValues → Base32 12 chars
   b. reintenta ante colisión (UNIQUE)
   c. INSERT masivo en labels (batch_id, box_number 1..N, status='generated')
   d. registra lote en label_batches + log de actividad
3. Render QR en servidor: código → SVG vectorial (módulos nítidos, sin pixeles rotos)
4. Layout de impresión: rejilla por hoja (papel, márgenes, espaciado, orientación,
   cantidad por hoja) → PDF vectorial
5. Descarga del PDF + historial de lotes consultable
6. Cada QR queda inmutable: el contenido cambia, el código nunca
```

---

## 7. Flujo de activación

```
GENERADO ──(compra / asignación / código de activación)──▶ ACTIVO ──(admin)──▶ BLOQUEADO
   │                                                            │
   │  sin ficha, sin visibilidad                                 │  editable por propietario
   │                                                             │  escaneable según visibility
   └── el QR siempre resuelve a /q/[code] (página "etiqueta sin asignar")
```

1. **Compra:** plancha física adquirida → el admin asigna el lote a la cuenta del comprador (automático con Mercado Pago en el futuro; manual hoy).
2. **Reclamo alternativo:** el usuario introduce el *código de activación* impreso en la plancha (8 dígitos) desde "Crear etiqueta" → se asocian las etiquetas a su cuenta. Conocer el QR no alcanza.
3. **Uso:** el usuario pega el sticker, escanea y completa la ficha.
4. **Bloqueo:** el admin puede bloquear etiquetas (fraude, pérdida, reventa). Bloqueada → página de aviso.
5. Los QR de lotes nunca se reutilizan ni se regeneran.

---

## 8. Diseño de interfaz

**Menú principal (7 botones grandes):**
1. 📦 Mis inventarios
2. 📷 Escanear QR
3. ➕ Crear etiqueta
4. 🎨 Diseñar etiquetas
5. 🔍 Buscar objetos
6. 👥 Compartir inventarios
7. 👤 Mi cuenta

- Mobile-first, targets grandes (≥48px), navegación por tabs inferior en celular
- Estados vacíos con ilustración + CTA
- Carga optimista y sincronización en tiempo real (supabase realtime) para inventarios compartidos
- Landing pública con el eslogan y explicación del producto físico

**Rutas:** `/` · `/login` · `/signup` · `/q/[code]` · `/app/*` · `/admin/*`

---

## 9. Plan de desarrollo por fases

| Fase | Contenido | Entregable / Gate |
|---|---|---|
| **1** | Análisis, arquitectura, conexiones | ✅ Este documento + tu aprobación |
| **2** | DB, auth, RLS, storage, migraciones iniciales | Tablas + policies revisadas por vos |
| **3** | Generador de identificadores y QR únicos | Endpoint que genera y resuelve códigos |
| **4** | Lotes, numeración BOX, administración de lotes | Generar lote 50/100/200 + historial |
| **5** | Diseñador de etiquetas + plantillas A–E | Preview real en cm + 5 plantillas |
| **6** | Inventarios, fotos, fichas digitales, búsqueda | CRUD completo + buscador global |
| **7** | Escaneo, permisos, uso compartido | Escáner + roles + invitaciones + RLS auditada |
| **8** | PDF e impresión | PDF descargable + prueba de escaneo impreso |
| **9** | Panel admin + estadísticas | `/admin` con métricas y log |
| **10** | Planes, suscripciones, Mercado Pago (solo preparación) | Planes configurables, **sin cobros reales** |
| **11** | Arquitectura de IA (preparación, sin APIs de costo) | Interfaces desacopladas, app sin IA |
| **12** | Pruebas, auditoría de seguridad, despliegue | dominio + PWA instalable |

**Gate por fase:** no se avanza hasta tu OK.

---

## 10. Decisiones técnicas registradas

| # | Decisión | Fecha | Estado |
|---|---|---|---|
| D-001 | QR = URL opaca `/q/[code]`, códigos Base32 12 chars, nunca datos | 2026-11-10 | Propuesto |
| D-002 | Numeración visible `BOX n` por lote; identificador interno único global | 2026-11-10 | Propuesto |
| D-003 | Datos privados nunca en el QR ni en PDF de terceros | 2026-11-10 | Propuesto |
| D-004 | Visibilidad: private / shared / public, resuelta por RLS vía función `get_public_label` | 2026-11-10 | Propuesto |
| D-005 | PDF y QR siempre renderizados en servidor (calidad + seguridad) | 2026-11-10 | Propuesto |
| D-006 | Sin cobros reales ni APIs de pago/IA sin autorización explícita | 2026-11-10 | **Vigente** |
| D-007 | Mercado inicial: Argentina, precios en ARS configurables desde `/admin` | 2026-11-10 | Propuesto |
| D-008 | No migraciones destructivas; SQL versionado en `supabase/migrations/` | 2026-11-10 | **Vigente** |

---

## 11. Registro de sesión

### Sesión 1 — 2026-11-10
- ✅ Revisado directorio (vacío), GitHub, Supabase, Vercel, tooling
- ✅ Arquitectura, estructura, modelo de datos, RLS, flujos, UI y fases documentados
- ⏳ **ESPERANDO APROBACIÓN para FASE 2**
- ❌ Sin crear repo, sin crear proyecto, sin migraciones, sin código
