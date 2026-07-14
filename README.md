# Pistas Municipales de Pádel — Chozas de Canales

Sitio de una sola página (`index.html`) para reservar las pistas municipales de
pádel de Chozas de Canales online, más un panel de administración
(`admin.html`). Sin build step: HTML/CSS/JS puro + Supabase JS por CDN, así que
se puede abrir tal cual o subir a cualquier hosting estático (GitHub Pages,
Netlify, Vercel...).

## Qué incluye

- Portada con pistas, calendario de disponibilidad (por pista + fecha) y
  reserva con duración a elegir (1h / 1h30 / 2h, cada una con su precio, al
  estilo Playtomic) directamente desde el navegador, con opción de añadir luz.
- Panel de administración (`admin.html`, protegido con login) para confirmar o
  cancelar reservas, gestionar pistas, precios, extras (luz) y cierres de
  mantenimiento, y emitir facturas simples.
- Todo el contenido de pistas/precios se lee de la base de datos, así que puedes
  cambiarlo sin tocar código.
- Se puede **instalar como app** (tanto la web de reservas como el panel de
  administración), con su propio icono, pantalla completa sin barra del
  navegador y funcionamiento básico sin conexión.

## Instalar como app

Tanto `index.html` (la web de reservas) como `admin.html` (el panel) son
instalables como una app normal, en el móvil o el ordenador:

- **Android/Chrome/Edge**: al entrar aparece un botón **"📲 Instalar app"**
  (arriba a la derecha en la web, y en la pantalla de login o la cabecera del
  panel); al pulsarlo, el propio navegador pregunta si quieres instalarla y
  queda como un icono más en el móvil/escritorio.
- **iPhone/iPad (Safari)**: Safari no muestra ese botón (Apple no lo permite),
  pero se instala igual desde Compartir → "Añadir a pantalla de inicio".

Una vez instalada se abre en pantalla completa, con su propio icono (la
raqueta y pelota de pádel) y sin la barra de direcciones del navegador, como
cualquier otra app. La web de reservas (`index.html`) además guarda su
"cascarón" (sw-public.js) para poder abrirse aunque no haya buena conexión, y
el panel (`admin.html`) reutiliza el service worker de las notificaciones
(`sw-padel.js`).

## Fotos de pistas y torneos

Desde `admin.html` (pestañas «Pistas» y «Torneos») puedes subir una foto por
pista y por torneo/liga con un simple selector de archivo — se guarda en el
bucket público de Supabase Storage `galeria-padel` y aparece automáticamente
en la web pública (si no hay foto, se muestra un icono/emoji por defecto).
Solo el administrador puede subir o borrar fotos; cualquiera puede verlas.

## Base de datos (Supabase)

Este proyecto reutiliza el proyecto Supabase **`paseo-perros-app`** (ref
`stnuvhlqicftjbtlybit`) de la organización, con tablas propias sufijadas en
`_padel` para no mezclarse con las tablas de esa otra app:

- `pistas_padel` — pistas del club (nombre, superficie, orden, activa).
- `duraciones_padel` — precio según cuánto tiempo se reserva (1h = 3€, 1h30 =
  4,5€, 2h = 6€ por defecto); el cliente elige duración al reservar, como en
  Playtomic. La reserva mínima es la duración más corta activa.
- `mantenimientos_padel` — cierres temporales de una pista (fecha inicio/fin +
  motivo); mientras una pista está en mantenimiento no se puede reservar esas
  fechas.
- `extras_padel` — extras opcionales de la reserva (de momento, "Luz").
- `reservas_padel` — cada solicitud de reserva (estado `confirmada` en cuanto
  se guarda si la hora está libre), incluye duración, precio, y si el cliente
  ha pedido luz y a qué precio se congeló en ese momento.
- `disponibilidad_padel` (vista) — solo pista/fecha/hora de reservas activas,
  para pintar los horarios libres/ocupados (sin datos personales).
- `invoices_padel` — facturas simples (numeración correlativa por año).

Las reservas hechas desde la web, WhatsApp o el panel se **confirman solas**
(estado `confirmada`) si la hora sigue libre en ese momento — nadie tiene que
aceptarlas a mano. Por si dos personas reservan la misma pista y hora a la vez,
hay una restricción a nivel de base de datos (`reservas_padel_no_overlap`, con
`btree_gist`) que impide físicamente que se guarden dos reservas activas
solapadas en la misma pista; la segunda petición recibe un error y la app le
pide elegir otro horario.

Seguridad (Row Level Security):
- Cualquiera puede **leer** pistas, duraciones/precios, mantenimientos, extras y
  disponibilidad.
- Cualquiera puede **crear** una reserva (INSERT), pero **nadie puede leer, editar
  ni borrar** reservas ni facturas desde el navegador salvo el administrador
  logueado (`ernestobm2012@gmail.com`, mismo usuario que ya usas en Supabase).

Para cambiar precios por duración, pistas, extras o mantenimientos: hazlo desde
`admin.html` (pestañas «Pistas» y «Precios»), o directamente en el dashboard de
Supabase → Table Editor.

## Bot de WhatsApp

Hay un bot de WhatsApp que hace reservas solo, sin pasar por la web. Vive como
una Edge Function de Supabase (`whatsapp-webhook-padel`, en el mismo proyecto
`stnuvhlqicftjbtlybit`), no en este repositorio, así que se actualiza desde el
dashboard de Supabase → Edge Functions.

Funciona con la API de WhatsApp Cloud de Meta:
1. El cliente escribe al número de WhatsApp del ayuntamiento/pistas.
2. El bot le pregunta pista → fecha → hora libre → duración (con su precio)
   → nombre → jugadores → si quiere luz, y al confirmar crea la reserva
   directamente en `reservas_padel` ya **confirmada** (si la hora sigue libre
   en ese momento), igual que si reservara por la web.
3. Respeta los mantenimientos (`mantenimientos_padel`) y los horarios ya
   ocupados, igual que el calendario de `index.html`.
4. Si el cliente escribe algo que no encaja en el guion (una pregunta libre),
   el bot puede responder con IA (Claude) si se configura `ANTHROPIC_API_KEY`;
   si no, simplemente pide que llames por teléfono.

El estado de cada conversación se guarda en `wa_conversations_padel`
(pista/fecha/hora que se van eligiendo) y el historial de mensajes en
`wa_messages_padel`. Ninguna de las dos es accesible desde el navegador (solo
la Edge Function, con la service role key, puede leerlas).

### Puesta en marcha (una vez, en Meta for Developers)

1. Crea una cuenta en [developers.facebook.com](https://developers.facebook.com/)
   y una App de tipo "Business" con el producto **WhatsApp** añadido.
2. En WhatsApp → API Setup, verás un número de prueba gratuito de Meta (o
   añade y verifica el número real del ayuntamiento). Copia el
   **Phone number ID** y genera un **token de acceso** (para producción,
   uno permanente vía System User, no el temporal de 24h).
3. En Supabase → tu proyecto → Edge Functions → `whatsapp-webhook-padel` →
   Secrets, añade:
   - `WHATSAPP_TOKEN` — el token de acceso de Meta.
   - `WHATSAPP_PHONE_NUMBER_ID` — el Phone number ID.
   - `WHATSAPP_VERIFY_TOKEN` — te lo inventas tú (cualquier cadena secreta),
     tiene que coincidir con el paso siguiente.
   - `ANTHROPIC_API_KEY` — opcional, solo si quieres que responda preguntas
     libres con IA.
4. En Meta → WhatsApp → Configuration → Webhook, pon como URL:
   `https://stnuvhlqicftjbtlybit.supabase.co/functions/v1/whatsapp-webhook-padel`
   y como "Verify token" el mismo valor que pusiste en `WHATSAPP_VERIFY_TOKEN`.
   Suscríbete al campo `messages`.
5. Escribe al número de WhatsApp desde tu móvil — el bot debería responder con
   el menú.

Antes de escribir tráfico real, cambia también `OWNER_PHONE` dentro del código
de la función por el teléfono real de contacto (ahora mismo tiene un número de
ejemplo).

## Notificaciones de reserva nueva

Cada vez que se crea una reserva (desde la web, WhatsApp o el panel), un
disparador de la base de datos avisa a una Edge Function
(`notify-new-reservation-padel`) que manda una **notificación push** al
navegador/móvil del administrador — sin coste, sin necesidad de app ni de
correo, usando el estándar Web Push.

Para verlas solo hay que entrar en `admin.html` y pulsar el botón
**"🔔 Activar notificaciones"** que aparece arriba a la derecha (una vez,
en cada dispositivo desde el que quieras recibirlas) y aceptar el permiso
que pide el navegador. La suscripción se guarda en la tabla
`push_subscriptions_padel`; si el navegador la invalida (por ejemplo, se
desinstala el sitio) se borra sola la próxima vez que falle un envío.

### Puesta en marcha (una vez, en Supabase)

Como con el bot de WhatsApp, hay que configurar unos secretos en el
dashboard de Supabase, porque no se pueden crear automáticamente desde
aquí. En **Supabase → Edge Functions → `notify-new-reservation-padel` →
Secrets**, añade:

- `VAPID_PUBLIC_KEY` = `BEdx5e2At2Fz3X9ROFHto57PbEkQVO-el9uj_lEtfcCy15u221x-TBqI_KDKECChZk6Jd_iTg8zwkMZ-iq1rnWU`
- `VAPID_PRIVATE_KEY` = `mN5uBeDxH2fv05NbgzJBNVxABfJLRlv1We7gEul4380`
- `RESERVATION_WEBHOOK_SECRET` = `2dd0e0fb6ecf1b5203e2a4b689b57e86008ea4017b2dd8b9`

(`VAPID_SUBJECT` es opcional; por defecto usa tu email de administrador.)
Estas claves ya están generadas específicamente para este proyecto — no
hace falta cambiarlas, solo copiarlas tal cual a los secretos de la
función. Sin este paso, la reserva se sigue creando con normalidad, pero
la notificación fallará silenciosamente (no bloquea nada).

## Torneos y ligas

Desde `admin.html` → pestaña «Torneos» puedes crear un torneo (eliminatoria
directa) o una liga (todos contra todos), eligiendo pistas, fechas y fecha
límite de inscripción. La gente se apunta por parejas desde `index.html`
(sección «Torneos»), hasta esa fecha límite.

Cuando cierras inscripciones, el botón **"🎲 Generar cruces"**:
- Sortea las parejas y arma el cuadro (con "byes" repartidos si el número de
  parejas no es potencia de 2) o el calendario de la liga (método del círculo,
  todos contra todos).
- Asigna fecha, hora y pista real a cada partido dentro del rango de la
  competición, sin chocar con reservas o mantenimientos ya existentes — y
  **bloquea automáticamente esas pistas en el calendario** (se ven como
  reservas normales, con origen "torneo"/"liga").
- En el torneo, las rondas siguientes empiezan con casillas "Ganador de
  Cuartos de final" etc.; al marcar el ganador de un partido en el panel, el
  cuadro avanza solo y actualiza el nombre en la reserva bloqueada.
- En la liga se calcula una tabla de clasificación simple (partidos jugados,
  ganados, perdidos) a medida que registras resultados.

Tablas: `competiciones_padel`, `competicion_pistas_padel`,
`competicion_inscripciones_padel` (con vista pública sin teléfonos,
`inscripciones_padel_publico`, para mostrar el cuadro/apuntados en la web) y
`competicion_partidos_padel`. Cada partido con pista/fecha asignada crea una
fila en `reservas_padel` (columna `competicion_partido_id`) para que bloquee
el hueco igual que cualquier otra reserva.

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

- Configura los secretos de la Edge Function `notify-new-reservation-padel`
  (ver sección «Notificaciones de reserva nueva» más arriba) para que
  lleguen avisos de reservas nuevas a tu móvil/navegador.
- Sustituye los datos de contacto de ejemplo (teléfono, email, dirección, redes
  sociales, mapa) por los reales en `index.html` (sección «Contacto»).
- Ya se han creado 2 pistas y los 3 precios por duración de ejemplo (1h=3€,
  1h30=4,5€, 2h=6€) en Supabase — ajústalos desde `admin.html` → pestaña
  «Precios» a los reales.
- El extra de "Luz" tiene un precio de ejemplo — ajústalo desde `admin.html`
  → pestaña «Precios» → «Extras».
- El horario de apertura está fijado de 08:00 a 23:00, con horarios de inicio
  cada 30 minutos (constantes `OPEN_TIME`, `CLOSE_TIME`, `START_STEP_MINUTES`
  en `index.html`); cámbialo si tu club tiene otro horario.
- Este proyecto está pensado para poder migrar más adelante a un stack con build
  (ej. Next.js) si el negocio crece — Supabase seguiría siendo la base de datos.
