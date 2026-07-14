# Pistas Municipales de Pádel — Chozas de Canales

Sitio de una sola página (`index.html`) para reservar las pistas municipales de
pádel de Chozas de Canales online, más un panel de administración
(`admin.html`). Sin build step: HTML/CSS/JS puro + Supabase JS por CDN, así que
se puede abrir tal cual o subir a cualquier hosting estático (GitHub Pages,
Netlify, Vercel...).

## Qué incluye

- Portada con pistas, precios por franja horaria y calendario de disponibilidad
  (por pista + fecha) para reservar un turno de 90 minutos directamente desde el
  navegador, con opción de añadir luz a la reserva.
- Panel de administración (`admin.html`, protegido con login) para confirmar o
  cancelar reservas, gestionar pistas, precios, extras (luz) y cierres de
  mantenimiento, y emitir facturas simples.
- Todo el contenido de pistas/precios se lee de la base de datos, así que puedes
  cambiarlo sin tocar código.

## Base de datos (Supabase)

Este proyecto reutiliza el proyecto Supabase **`paseo-perros-app`** (ref
`stnuvhlqicftjbtlybit`) de la organización, con tablas propias sufijadas en
`_padel` para no mezclarse con las tablas de esa otra app:

- `pistas_padel` — pistas del club (nombre, superficie, orden, activa).
- `precios_padel` — tarifas por franja (`laborable` / `fin_semana_festivo`).
- `festivos_padel` — días festivos, para aplicar la tarifa de fin de semana.
- `mantenimientos_padel` — cierres temporales de una pista (fecha inicio/fin +
  motivo); mientras una pista está en mantenimiento no se puede reservar esas
  fechas.
- `extras_padel` — extras opcionales de la reserva (de momento, "Luz").
- `reservas_padel` — cada solicitud de reserva (estado `pendiente` por defecto),
  incluye si el cliente ha pedido luz y a qué precio se congeló en ese momento.
- `disponibilidad_padel` (vista) — solo pista/fecha/hora de reservas activas,
  para pintar los horarios libres/ocupados (sin datos personales).
- `invoices_padel` — facturas simples (numeración correlativa por año).

Seguridad (Row Level Security):
- Cualquiera puede **leer** pistas, precios, festivos, mantenimientos, extras y
  disponibilidad.
- Cualquiera puede **crear** una reserva (INSERT), pero **nadie puede leer, editar
  ni borrar** reservas ni facturas desde el navegador salvo el administrador
  logueado (`ernestobm2012@gmail.com`, mismo usuario que ya usas en Supabase).

Para cambiar precios, pistas, extras o mantenimientos: hazlo desde `admin.html`
(pestañas «Pistas» y «Precios»), o directamente en el dashboard de Supabase →
Table Editor.

## Cómo lo pruebas en local

Solo hace falta un servidor estático simple (los módulos ES no funcionan con `file://`):

```bash
python3 -m http.server 8000
# abre http://localhost:8000
```

## Publicar la web

La forma más rápida y gratuita es **GitHub Pages**:
- En GitHub → Settings → Pages → Source: rama `main`, carpeta `/root`.
- En unos minutos tendrás la web en
  `https://ernestobm2012-tech.github.io/pista_padel/`.

## Pendiente de tu parte / a revisar

- Sustituye los datos de contacto de ejemplo (teléfono, email, dirección, redes
  sociales, mapa) por los reales en `index.html` (sección «Contacto»).
- Ya se han creado 2 pistas y 3 precios de ejemplo en Supabase — ajústalos desde
  `admin.html` a los reales.
- El extra de "Luz" tiene un precio de ejemplo (4 €) — ajústalo desde `admin.html`
  → pestaña «Precios» → «Extras».
- El horario de reserva está fijado de 08:00 a 23:00 en turnos de 90 minutos
  (constantes `OPEN_TIME`, `CLOSE_TIME`, `SLOT_MINUTES` en `index.html`); cámbialo
  si tu club tiene otro horario o duración de turno.
- Este proyecto está pensado para poder migrar más adelante a un stack con build
  (ej. Next.js) si el negocio crece — Supabase seguiría siendo la base de datos.
