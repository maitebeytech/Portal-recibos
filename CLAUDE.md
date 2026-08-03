# Portal de Recibos — Grupo GNP

Aplicación web para que un estudio contable (Grupo GNP) gestione la firma
digital de recibos de sueldo de sus empresas cliente. Multi-tenant: un
estudio (super admin) administra varias empresas, cada una con sus propios
empleados y recibos.

## Estado general

Prototipo funcional conectado a una base de datos real (Supabase), con
autenticación real, hosteado gratis en GitHub Pages. Todavía **no existe
la app del empleado** — todo lo que hace un empleado (ver su recibo, firmar)
se simula hoy desde el panel del admin operativo.

## Stack

- **Frontend**: un solo archivo HTML (`index.html`) con JS vanilla, sin build,
  sin framework. Todo el código vive ahí adentro (HTML + CSS + JS en un archivo).
- **Backend**: Supabase (Postgres + Auth + Storage + 1 Edge Function).
- **Hosting**: GitHub Pages (gratis, sin límite de publicaciones).
  Repo: `maitebeytech/Portal-recibos`, rama `main`, archivo `index.html` en la raíz.
  URL pública: `https://maitebeytech.github.io/Portal-recibos/`
- **Librerías cargadas por CDN** (no hay npm ni build step):
  `@supabase/supabase-js@2`, `xlsx` (SheetJS, para importar Excel),
  `pdf.js` (leer texto de PDFs), `pdf-lib` (separar páginas y estampar marca de agua).

## Supabase

- **Project URL**: `https://zddlpkalpubxpbyevqsm.supabase.co`
- La **anon key** está hardcodeada en el HTML — es seguro, está pensada para
  eso. La **service_role key nunca** debe estar en el frontend; solo la usa
  la Edge Function del lado del servidor.
- RLS (Row Level Security) está activo en todas las tablas. Hay una función
  SQL `mi_rol()` y `mis_empresas()` que usan las políticas para saber quién
  es el usuario logueado y qué le corresponde ver.

### Tablas

- **empresas**: `id, nombre, cuit, activa, bloquear_si_no_firma,
  dias_aviso_automatico, marca_agua_habilitada, permitir_firma_no_conforme, creado_en`
  - Los últimos dos switches (`marca_agua_habilitada`, `permitir_firma_no_conforme`)
    **hoy solo son visuales** — se guardan en la base pero todavía no cambian
    ningún comportamiento real. Falta cablearlos.
- **perfiles**: `id (= auth.users.id), email, rol ('superadmin' | 'admin_operativo'
  | 'visualizador'), creado_en`
- **asignaciones**: `usuario_id, empresa_id` — qué empresas puede ver/gestionar
  cada admin operativo o visualizador.
- **empleados**: `id, empresa_id, nombre, cuil, mail, invitacion_estado
  ('pendiente'|'invitado'), pin_hash, activo, creado_en`
  - ⚠️ `pin_hash` se guarda **en texto plano**, no hasheado. Es un simulador
    temporal. Hay que arreglar esto antes de que haya empleados reales.
  - `activo`: dar de baja un empleado nunca borra nada, solo pone esto en `false`.
- **recibos**: `id, empleado_id, mes, archivo_limpio_path, archivo_marca_path,
  firmado, firmado_en, ultimo_aviso_en, ultimo_aviso_tipo, visualizado_en, creado_en`
  - `visualizado_en`: columna ya creada, **todavía sin usar** — es para cuando
    exista la app del empleado y necesitemos registrar cuándo abrió el recibo
    sin firmarlo (estado "visualizado pero no firmado").
  - `mes` es texto libre (ej: "julio de 2026"), generado con
    `toLocaleDateString('es-AR', {month:'long', year:'numeric'})` al importar.
    No es una fecha real, es una etiqueta de período.

### Storage

- Bucket **`Recibos`** (con R mayúscula — así quedó creado, el código lo
  respeta así). Privado, no público.
- Cada recibo genera dos archivos: uno limpio y uno con marca de agua
  quemada de verdad en el PDF (no es un overlay de HTML, así no se puede
  sortear descargando desde el visor nativo del navegador).
- Texto de la marca de agua: **pendiente de definir**, hoy dice
  "NO FIRMADO - SOLO VISUALIZACION" como placeholder.

### Edge Function: `crear-usuario`

Corre en Supabase (no en el navegador) porque necesita la `service_role key`
para crear logins reales. Verifica que quien llama sea superadmin, después:
- Crea el usuario en `auth.users` (con `admin.inviteUserByEmail`, manda
  invitación real por mail usando el sistema de mails que trae Supabase).
- Inserta en `perfiles` y `asignaciones`.
- Soporta `enviarInvitacion: true/false` — si es `false`, crea el login con
  una contraseña temporal aleatoria y NO manda mail (queda pendiente
  invitarlo después).

El código completo de la función está en Supabase → Edge Functions →
`crear-usuario`. Si hay que tocarlo, hay que editarlo ahí directamente
(no vive en este repo de GitHub).

### Historial de migraciones SQL

Las queries se guardan en el SQL Editor de Supabase con nombres tipo
`1-Tablas`, `2-Reglas`, etc. Algunas se van "pisando" (reescribiendo) para
no acumular decenas de queries — así que el número no siempre corresponde
a algo que siga existiendo tal cual. Lo importante es el estado final de
la base, no el historial de queries.

Últimos cambios de esquema (por si hay que rehacerlos en otro entorno):
```sql
alter table empleados add column if not exists activo boolean not null default true;
alter table recibos add column if not exists visualizado_en timestamptz;
alter table empresas add column if not exists marca_agua_habilitada boolean not null default true;
alter table empresas add column if not exists permitir_firma_no_conforme boolean not null default false;
```

## Roles y qué ve cada uno

- **Super admin**: solo Empresas y Usuarios. No ve nada operativo (ni
  empleados ni recibos) — eso es tarea exclusiva del admin operativo.
  - Empresas → submenú: Visualización (Activas / Inactivas, solo lectura)
    y Gestión (Agregar nueva / Dar de baja / Reactivar).
  - Usuarios → submenú: Administrar usuarios (edición en línea: rol por
    desplegable, empresas asignadas con chips + botón "+ empresa") /
    Crear nuevo usuario (mail + rol → modales encadenados preguntando si
    asignar empresa(s) y si mandar invitación ya).
- **Admin operativo**: entra directo si tiene una sola empresa asignada;
  si tiene varias, primero elige en una pantalla selectora ("¿A qué
  empresa querés ingresar?"). Arriba dice "Trabajando en: [empresa]" con
  botón "Cambiar de empresa".
  - Empleados → Ver datos (solo nombre/CUIL/mail, con lápiz para editar
    en línea) / Agregar nuevo (Carga manual o Importar Excel, y al
    terminar pregunta si enviar invitación a la app) / Dar de baja
    (con su Reactivar).
  - Recibos → Importar recibos (sube el PDF único, lo separa por CUIL,
    previsualización con ojito antes de confirmar) / Estado de recibos
    recientes (período actual + "Pendientes [período anterior]" para lo
    que quedó sin firmar de meses viejos, botón de aviso manual) /
    Historial de recibos (desplegable de período).
  - Reportes: resumen de firmas + aviso manual/automático.
  - Configuración: switches (bloqueo de recibo siguiente si no firmó,
    días de aviso automático, marca de agua, permitir firma no conforme
    — estos dos últimos sin efecto real todavía).
- **Visualizador**: mismo mecanismo de selector de empresa que admin
  operativo, pero dice "Visualizando:" en vez de "Trabajando en:". Solo
  lectura: Ver datos de empleados, Estado de recibos recientes, Historial,
  Reportes. Sin botones de agregar/editar/eliminar/firmar.

## Diseño visual

- Paleta azul marino (pedido explícito: "es lo que solemos usar para todo").
- Tipografías: Lora (serif, para títulos) + Inter (sans, cuerpo) + IBM Plex
  Mono (para CUIT/CUIL, números).
- Logo real de Grupo GNP incrustado en base64 en el HTML (pantallas de
  login y header).
- Barra lateral azul, colapsable a solo íconos (dibujos SVG minimalistas,
  no emojis), arranca minimizada en cada login, se expande al clickear
  cualquier ícono o la flechita.
- El estado "Firmado" se muestra como un sello (bordeado, rotado
  levemente), no una etiqueta de color plana — es el capricho visual
  del diseño, todo lo demás es disciplinado.
- Pantalla vacía al entrar ("Elegí una opción del menú para acceder"),
  centrada, sin recuadro, en gris suave.

## Pendientes conocidos (no son bugs, son alcance futuro)

1. **App del empleado** — no existe. Hoy el admin operativo simula la
   firma desde su propio panel. Es la pieza más grande que falta.
2. **PIN de firma en texto plano** — hay que hashearlo antes de que
   existan empleados reales.
3. **Validez legal de la firma electrónica** — nunca se confirmó con un
   abogado laboral ni con el estudio. No dar por sentado que alcanza.
4. **Revisión de seguridad por IT** — pendiente, recomendado antes de
   cargar datos reales de clientes/empleados.
5. **Texto de la marca de agua** — placeholder, falta definir la
   redacción final.
6. **Switches de Configuración sin efecto** — `marca_agua_habilitada` y
   `permitir_firma_no_conforme` se guardan pero no hacen nada todavía.
7. **`visualizado_en`** — columna creada, sin usar (depende de la app
   del empleado).
8. **Envío real de avisos** (recordatorios a empleados) — hoy solo
   actualiza una fecha en la base, no manda ningún mail/notificación real.
9. **Botón "ver recibos por empresa" para super admin** — se descartó a
   propósito (el super admin no hace nada operativo), pero quedó anotado
   como posible pedido futuro.
10. **Integración con Tango** — decisión tomada: NO se va a hacer. El
    import de Excel/PDF manual es el diseño definitivo, no un parche
    temporal.

## Cómo publicar un cambio

1. Editar `index.html` (todo el código vive en ese único archivo).
2. Verificar que el `<script>` compile (no hay build step, es JS plano).
3. Subir el archivo a `github.com/maitebeytech/Portal-recibos`,
   reemplazando el `index.html` existente (Add file → Upload files,
   arrastrar, Commit changes directo a `main`).
4. GitHub Pages publica solo, gratis, sin límite — no hay créditos que
   se gasten (a diferencia de Netlify, que se dejó de usar por eso).
