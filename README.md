# GestionMyPadel — plataforma multi-cliente (Chozas de Canales y otros)

`index.html` es la web de ventas de **GestionMyPadel** (el producto), y
`club.html` es la web de reservas de pistas de pádel de cada cliente,
más un panel de administración (`admin.html`). Sin build step: HTML/CSS/JS
puro + Supabase JS por CDN, así que se puede abrir tal cual o subir a
cualquier hosting estático (GitHub Pages, Netlify, Vercel...).

Este mismo código sirve para **varios clientes a la vez** (ayuntamientos,
comunidades de vecinos, clubes de pádel), cada uno con sus datos totalmente
aislados — ver la sección "Multi-tenant" más abajo. La primera organización
dada de alta, y la que se usa como referencia en el resto de este documento,
es "Pistas Municipales de Pádel de Chozas de Canales".

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

Tanto `club.html` (la web de reservas), `admin.html` (el panel de cada club)
como `plataforma.html` (tu panel de dueño de la plataforma) son instalables
como una app normal, en el móvil o el ordenador:

- **Android/Chrome/Edge**: al entrar aparece un botón **"📲 Instalar app"**
  (arriba a la derecha en la web, y en la pantalla de login o la cabecera del
  panel); al pulsarlo, el propio navegador pregunta si quieres instalarla y
  queda como un icono más en el móvil/escritorio.
- **iPhone/iPad (Safari)**: Safari no muestra ese botón (Apple no lo permite),
  pero se instala igual desde Compartir → "Añadir a pantalla de inicio".

Una vez instalada se abre en pantalla completa, con su propio icono (la
raqueta y pelota de pádel) y sin la barra de direcciones del navegador, como
cualquier otra app. La web de reservas (`club.html`) además guarda su
"cascarón" (sw-public.js) para poder abrirse aunque no haya buena conexión, y
tanto `admin.html` como `plataforma.html` reutilizan el mismo service worker
de las notificaciones (`sw-padel.js`).

## Estadísticas

En `admin.html` → pestaña «Estadísticas» tienes, con un selector de periodo
(últimos 30/90 días, último año o todo el historial):

- Resumen: reservas confirmadas, clientes distintos, dinero ganado y ticket
  medio (el uso municipal no cuenta como ingreso).
- Ingresos por mes (últimos 12 meses del periodo elegido).
- Qué duración se reserva más (1 hora / 1h30 / 2 horas), con el número de
  reservas y el porcentaje sobre el total.
- A qué horas hay más demanda, para detectar las franjas más solicitadas.
- Los clientes más frecuentes del periodo.

## Fotos de pistas y torneos

Desde `admin.html` (pestañas «Pistas» y «Torneos») puedes subir una foto por
pista y por torneo/liga con un simple selector de archivo — se guarda en el
bucket público de Supabase Storage `galeria-padel` y aparece automáticamente
en la web pública (si no hay foto, se muestra un icono/emoji por defecto).
Solo el administrador puede subir o borrar fotos; cualquiera puede verlas.

## Noticias

Cada organización tiene su propia pestaña «Noticias» en `admin.html` para
publicar novedades: título, texto y una imagen opcional (se sube al mismo
bucket `galeria-padel` que las fotos de pistas/torneos). Se puede publicar y
borrar, pero no editar — si te equivocas, borras y vuelves a publicar. En
`club.html` aparecen en una sección propia («Noticias», entre «Reservar» y
«Torneos»), ordenadas de más reciente a más antigua; si una organización no
tiene ninguna, se muestra un aviso de que todavía no hay noticias en vez de
ocultar la sección.

## Banner de publicidad

En `club.html` hay un banner flotante fijo (esquina inferior izquierda) que
enlaza a un patrocinador externo (actualmente `1105sports.com`). La imagen
vive en `assets/banner-1105sports.jpg` y el enlace de destino está en el
propio HTML (`<div id="adBanner">`). Cualquiera puede cerrarlo con la "✕";
el cierre se guarda en `localStorage` para que no vuelva a aparecer en ese
navegador. Para cambiar la imagen o el enlace, sustituye el archivo o edita
el `href` directamente — no hay panel de administración para esto, es un
banner fijo, no gestionable desde `admin.html`. Cada clic en el banner queda
registrado (ver «Estadísticas de la plataforma» más abajo) para saber cuánto
tráfico le está generando el sitio al patrocinador.

## Base de datos (Supabase)

Este proyecto reutiliza el proyecto Supabase **`paseo-perros-app`** (ref
`stnuvhlqicftjbtlybit`) de la organización, con tablas propias sufijadas en
`_padel` para no mezclarse con las tablas de esa otra app:

- `pistas_padel` — pistas del club (nombre, superficie, orden, activa).
- `duraciones_padel` — precio según cuánto tiempo se reserva (1h = 3€, 1h30 =
  4,5€, 2h = 6€ por defecto); el cliente elige duración al reservar, como en
  Playtomic. La reserva mínima es la duración más corta activa. Por defecto el
  precio es el mismo en todas las pistas, pero puede tener un precio distinto
  por pista (ver más abajo, "Precio por pista").
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

## Cuentas de cliente

Reservar una pista desde `club.html` **exige tener cuenta** (email y
contraseña, botón **"Iniciar sesión"** arriba a la derecha). Ya no existe la
opción de reservar como invitado: si alguien sin sesión pulsa un horario
libre, se le abre directamente el formulario de inicio de sesión/registro y,
al terminar, retoma automáticamente la reserva que estaba haciendo. Tener
cuenta permite:

- Que sus reservas se guarden siempre ligadas a su cuenta (`user_id`), y que
  el formulario se rellene solo con su nombre/teléfono/email.
- Ver y **cancelar sus propias reservas futuras** desde "Mi cuenta", sin
  tener que llamar o escribir al ayuntamiento.
- Tener control real de quién reserva qué, de cara a estadísticas y a evitar
  reservas fantasma o duplicadas.

### Límite de reservas activas por cliente

Para evitar que un solo vecino acapare huecos, cada cliente solo puede
tener un número máximo de reservas activas (confirmadas y todavía por
jugar) al mismo tiempo — se aplica igual si reserva desde la web o si el
admin le crea una reserva manual vinculada a su cuenta. Ese máximo es
**configurable**: en `admin.html` → pestaña «Precios» → «Ajustes
generales» puedes cambiarlo en cualquier momento (por defecto, 2). Se
aplica con un trigger en la base de datos (`check_reservas_activas_limit`,
tabla `ajustes_padel`), así que no hay forma de saltárselo desde la web
pública; si alguien lo supera, ve un mensaje claro pidiendo que cancele
una reserva antes de crear otra.

Este proyecto Supabase se comparte con la app de pasear perros
(`paseo-perros-app`), pero **ambas gestionan sus propios clientes de forma
independiente**: al registrarse desde `club.html`, la cuenta se marca
internamente como cliente de pádel (`is_padel_customer`), y tanto el
disparador que rellena `clientes_padel` como el de la app de pasear perros
(que crea sus propios perfiles) respetan esa marca — un alta en una app
nunca crea datos de cliente en la otra.

Técnicamente, `reservas_padel` tiene una columna `user_id` y usa el sistema
de usuarios de Supabase Auth — el mismo que ya usa el administrador, pero
cualquiera puede registrarse como cliente. Una política de seguridad a nivel
de fila exige que toda reserva insertada desde la web pública tenga sesión
iniciada y `user_id = auth.uid()` (no se puede reservar en nombre de otro,
ni sin cuenta); el administrador conserva acceso completo por su email, para
sus reservas manuales. Cada cliente solo puede ver/cancelar sus propias
reservas (nunca las de otros) gracias a esas mismas políticas; cancelarlas
pasa siempre por una función segura (`cancelar_reserva_cliente`) que
comprueba que la reserva es suya y que todavía no ha pasado, así que no se
puede manipular directamente. El administrador sigue viendo y gestionando
todas las reservas desde `admin.html` como hasta ahora.

Al registrarse, Supabase manda un email de confirmación; hasta que el cliente
no pulsa el enlace de ese correo no puede iniciar sesión (si lo intenta antes,
ahora se le avisa explícitamente de que falta confirmar el email, en vez de
un mensaje genérico de "contraseña incorrecta"). También hay un enlace
**"¿Has olvidado tu contraseña?"** en el login que manda un correo con un
enlace para elegir una nueva.

El acceso a `admin.html` está restringido a tu cuenta (`ernestobm2012@gmail.com`)
por email, no solo por tener sesión iniciada: si una cuenta de cliente intenta
entrar ahí, se le cierra la sesión automáticamente y se le avisa de que no
tiene acceso.

### Precio por pista

Por defecto todas las pistas cobran lo mismo por duración. Si en el futuro
alguna pista debe tener un precio distinto (por ejemplo, una pista cubierta
o con mejores vistas), en `admin.html` → pestaña «Precios» hay un check
**"Mismo precio para todas las pistas"**:

- **Marcado (por defecto):** un único precio por duración vale para todas
  las pistas, como hasta ahora.
- **Desmarcado:** debajo aparece una tabla de precios independiente para
  cada pista activa, con un precio editable por cada duración. Si una pista
  no tiene un precio propio guardado todavía para una duración concreta,
  se usa automáticamente el precio base de esa duración — así nunca se
  rompe la reserva de una pista recién creada o sin configurar.

Este precio efectivo (el de la pista elegida, o el base si no tiene uno
propio) es el que se muestra y se cobra en los tres sitios donde se puede
reservar: la web pública (`club.html`), las reservas manuales desde
`admin.html`, y el bot de WhatsApp — los tres consultan primero la pista
elegida antes de calcular el precio.

Añadir una pista nueva en el futuro no requiere ningún cambio de código: en
`admin.html` → pestaña «Pistas» → **"+ Nueva pista"** se da de alta con su
nombre, superficie y foto, y automáticamente aparece en el buscador de
horarios, en las reservas manuales, en el bot de WhatsApp y en las tablas
de precios por pista.

### Listado de clientes registrados

En `admin.html` → pestaña «Clientes» → «Cuentas dadas de alta en la web» ves
la lista de quienes se han creado una cuenta (no las reservas de invitado),
con un **número de cliente** correlativo y los datos que aportaron al
registrarse (nombre, teléfono, email, fecha de alta). Se guarda en la tabla
`clientes_padel`, rellenada automáticamente por un disparador cada vez que
alguien se registra.

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
   ocupados, igual que el calendario de `club.html`.
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

No hace falta configurar nada más: las claves necesarias ya están
incluidas en la propia Edge Function `notify-new-reservation-padel`, así
que funciona de fábrica en cuanto activas las notificaciones desde un
dispositivo.

> Cada suscripción queda ligada a la organización desde la que se activó
> (`push_subscriptions_padel.organizacion_id`), así que el administrador de
> un club solo recibe avisos de sus propias reservas, nunca de las de otro
> club.

### Avisos de solicitudes nuevas en `plataforma.html`

Igual que en `admin.html`, en `plataforma.html` hay un botón **"🔔 Activar
notificaciones"** en la cabecera — pero para ti, como dueño de la
plataforma: te avisa al momento, con notificación push, cuando alguien
manda una **solicitud de demo** (formulario de `index.html`) o una
**solicitud de alta de organización** (`alta-organizacion.html`). Usa la
misma Edge Function de Web Push (`notify-super-admin-padel`) y la misma
tabla `push_subscriptions_padel`, con `organizacion_id` en blanco para
identificar que la suscripción es tuya como plataforma, no de un club
concreto.

## Reservas manuales desde admin.html

Al crear o modificar una reserva a mano (pestaña «Reservas»), la **duración
es obligatoria** — se elige de una lista (igual que en la web pública, con
su precio), no se escribe una hora de fin suelta. Si falta la hora de
inicio o la duración, el formulario no deja guardar y avisa con un mensaje.
La hora de fin, la duración y el precio se calculan y guardan solos a
partir de la duración elegida, así ninguna reserva se queda sin esos datos
(lo que antes aparecía como "Sin registrar" en Estadísticas).

Al elegir pista y fecha aparece una **rejilla de horarios**, igual que en
la web pública: las horas ya ocupadas (o que no dejan sitio para ninguna
duración) salen tachadas y no se pueden elegir. Al modificar una reserva
ya existente, su propio hueco se descuenta de la comprobación (para poder
dejarla tal cual o solo cambiarle la duración sin que se marque como
ocupada a sí misma).

### Cliente registrado obligatorio (salvo uso municipal)

Toda reserva manual nueva tiene que **vincularse a un cliente registrado**
(campo "Cliente registrado", con buscador por nombre o teléfono sobre la
tabla `clientes_padel`); si no se elige uno, el formulario no deja guardar.
La única excepción es marcar **"Uso municipal"**, que sigue funcionando
igual que antes (sin cliente, para reservas del propio ayuntamiento).

Si quien llama por teléfono todavía no tiene cuenta, el botón **"+ Cliente
nuevo (sin cuenta todavía)"** abre un mini-formulario (nombre, teléfono,
email) que crea la cuenta al vuelo —sin que el admin pierda su propia
sesión— llamando a la Edge Function `admin-create-client-padel` (con clave
de servicio, solo utilizable por el email del administrador) y la deja ya
seleccionada para esa reserva. La persona podrá luego usar "¿Has olvidado tu
contraseña?" en la web con su email para ponerle una contraseña propia.

Los bloqueos automáticos de pista para partidos de torneo/liga (generados
al pulsar "Generar cruces") no llevan cliente vinculado — representan un
partido entre parejas, no la reserva de un cliente concreto — y no se ven
afectados por esta exigencia.

## Mantenimiento de pistas: aviso visual y conflictos con reservas

`admin.html` → pestaña «Calendario»: los días en los que alguna pista está
cerrada por mantenimiento se marcan con un icono 🔧 y un borde rojo, sin
mezclarse con los colores de reservas pendientes/confirmadas. Al pulsar
ese día se ve, además de las reservas, un aviso con la pista, las fechas
y el motivo del cierre.

Al crear un cierre de mantenimiento (pestaña «Pistas» → «+ Nuevo cierre»),
si esa pista ya tenía reservas hechas dentro de esas fechas, no se guarda
directamente: aparece la lista de reservas afectadas y, para cada una, dos
botones:

- **🔁 Cambiar a otra pista libre**: busca automáticamente si otra pista
  activa está libre exactamente a esa misma fecha y hora (y no está ella
  también en mantenimiento) y, si la encuentra, mueve la reserva ahí sin
  tocar la hora. Si no hay ninguna libre, avisa de que hay que cancelarla
  u ofrecer otro horario a mano.
- **✕ Cancelar reserva**: la marca como `cancelada`.

En los dos casos se guarda una nota en `reservas_padel.message` explicando
el motivo (cambio de pista o cancelación por mantenimiento), que el
cliente ve automáticamente la próxima vez que abre "Mi cuenta" → "Mis
reservas" en la web — no hay, de momento, un aviso proactivo por WhatsApp,
push o email; el cliente se entera al consultar la app, y el
administrador ve en pantalla el teléfono de cada reserva afectada por si
prefiere avisar también por su cuenta. El cierre de mantenimiento no se
guarda hasta que todas las reservas en conflicto quedan resueltas.

## Torneos y ligas

Desde `admin.html` → pestaña «Torneos» puedes crear un torneo (eliminatoria
directa) o una liga (todos contra todos), eligiendo pistas, fechas y fecha
límite de inscripción. La gente se apunta por parejas desde `club.html`
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

### Cuadro de consolación (repesca) en torneos

Al generar los cruces de un torneo, además del cuadro principal se arma
automáticamente un **cuadro de consolación**: quienes pierden en la primera
ronda del cuadro principal pasan a una eliminatoria secundaria (con sus
propios "byes" si hace falta) que se juega en paralelo, con su propio
campeón de consolación. Se guarda en la misma tabla `competicion_partidos_padel`
(columna `bracket`: `principal` o `consolacion`); las casillas todavía sin
resolver muestran "Ganador de X" o "Perdedor de X" según corresponda, y al
marcar un ganador en el panel se actualiza solo tanto el cuadro principal
como quién pasa a consolación.

### Resultados públicos en la web

En `club.html`, cada torneo o liga con las inscripciones ya cerradas
muestra un botón **"🏆 Ver resultados"** con el cuadro (principal +
consolación) o, en una liga, la tabla de clasificación y las jornadas —
en modo solo lectura, sin controles de admin. En las ligas, si hay
partidos con la fecha ya pasada y sin resultado registrado, aparece un
aviso destacado animando a jugarlos (el mismo aviso también lo ve el
administrador en `admin.html`).

### Ligas por divisiones/niveles

Al crear una liga puedes indicar un **número de divisiones** (por ejemplo,
2 o 3, según el nivel de las parejas). Antes de generar el calendario,
en el detalle de la competición reparte cada pareja en su división, a
mano (un desplegable por pareja) o de golpe con **"🎲 Repartir
automáticamente"** (sorteo en grupos de tamaño parecido). "Generar
cruces" arma un calendario de todos-contra-todos independiente por cada
división, cada una con su propia tabla de clasificación y jornadas —
tanto en `admin.html` como en la vista pública.

### Apuntarse exige tener cuenta

Para apuntar una pareja a un torneo o liga ahora hace falta haber iniciado
sesión (antes se podía apuntar cualquiera sin cuenta). Si alguien pulsa
"Apuntar mi pareja" sin haber iniciado sesión, se abre el modal de cuenta
para iniciar sesión o registrarse y, al terminar, retoma automáticamente el
formulario de inscripción. Cada inscripción queda ligada al `user_id` de
quien la crea (`competicion_inscripciones_padel.user_id`), tanto para saber
quién puede entrar al chat de esa competición como, en el futuro, para que
cada jugador vea sus propias inscripciones.

Las inscripciones que ya existían antes de este cambio no tienen `user_id`
(se crearon sin cuenta), así que esas parejas no verán el botón de chat
hasta que alguien las vuelva a apuntar con una cuenta o un admin les asigne
el `user_id` a mano en la base de datos.

### El jugador/a 2 también tiene que ser cliente registrado

Apuntar una pareja exige que **ambos** jugadores sean clientes con cuenta
creada, no solo quien rellena el formulario. El jugador/a 1 ya queda
vinculado automáticamente (es quien ha iniciado sesión); para el jugador/a
2 hace falta indicar su **teléfono**, y en cuanto se escribe, la web busca
si hay un cliente registrado con ese número:

- Si lo encuentra, rellena solo su nombre y muestra "✓ Vinculado a
  [nombre]" — la inscripción se guarda ligada a su cuenta real
  (`pareja2_client_id`).
- Si no lo encuentra, avisa de que esa persona debe crearse una cuenta
  primero (con ese mismo teléfono) y no deja enviar el formulario hasta
  que se resuelva o se borre el teléfono (para apuntarse sin pareja
  todavía).

En `admin.html` → pestaña «Torneos» → dentro de cada competición, el
formulario "+ Añadir pareja manualmente" funciona igual pero para los dos
jugadores: en vez de escribir el nombre a mano, se busca y se elige un
cliente ya registrado (mismo buscador que en reservas manuales), así que
las parejas que añada el administrador también quedan siempre vinculadas
a cuentas reales.

Esto está reforzado en la base de datos: un trigger
(`check_pareja2_cliente_valido`) bloquea cualquier inscripción en la que
se indique un jugador/a 2 sin vincularlo a un cliente cuyo teléfono
coincida, y evita que alguien se empareje consigo mismo.

### Ranking de clientes

Con las parejas ya vinculadas a cuentas reales, tanto `admin.html` (pestaña
«Ranking») como la web pública (sección «Ranking») muestran una
clasificación de jugadores con los puntos acumulados en todos los torneos
y ligas que ya estén **finalizados**:

- En `admin.html` → pestaña «Torneos», cada competición con cruces
  generados tiene un botón **"🏁 Marcar como finalizado"**. Hasta que no se
  pulsa, esa competición no puntúa en el ranking (aunque ya tenga
  resultados cargados) — así el administrador decide cuándo cerrarla
  oficialmente.
- **Torneos**: los puntos dependen de la ronda alcanzada en el cuadro
  principal (campeón 100, finalista 60, semifinalista 30, cuartos 15,
  octavos 8, rondas anteriores 4) y, aparte, en el cuadro de consolación si
  lo hay (campeón de consolación 20, finalista 12, semifinalista 6, resto 3).
  Una pareja puede sumar puntos de los dos cuadros si jugó ambos (por
  ejemplo, perder en cuartos del principal y luego ganar la consolación).
- **Ligas**: los puntos dependen de la posición final en la clasificación
  de su división (1º 50, 2º 35, 3º 25, 4º 15, 5º 10, resto 5).
- Los puntos de cada pareja se suman a **los dos jugadores** que la
  forman (al `user_id` del jugador/a 1 y al `pareja2_client_id` del
  jugador/a 2), así que el ranking es de clientes individuales, no de
  parejas fijas.
- Solo cuentan los jugadores vinculados a una cuenta registrada. Una pareja
  cuyo jugador/a 2 se guardó como texto libre antes de este cambio, o una
  inscripción hecha a mano sin buscador de cliente, no suma puntos para
  esa persona hasta que quede vinculada.

Los valores de puntos están definidos en el código (`admin.html` e
`club.html`, buscar `TORNEO_PRINCIPAL_POINTS` / `TORNEO_CONSOLACION_POINTS`
/ `LIGA_POSICION_POINTS`) — son un punto de partida razonable, no una
tabla configurable desde el panel todavía; si el ayuntamiento quiere
ajustarlos más adelante, se puede convertir en una tabla editable en
`ajustes_padel`.

> Nota: el torneo y la liga de ejemplo que se crearon para probar el
> funcionamiento de cuadros/ligas usan parejas ficticias sin cuenta real,
> así que no aparecerán en el ranking hasta que jueguen partidas parejas
> vinculadas a clientes de verdad.

### Chat interno del torneo/liga

Cada competición tiene un chat privado en tiempo real, solo visible para
quienes tengan una inscripción activa en ella. En `club.html`, las tarjetas
de torneos/ligas muestran un botón **"💬 Chat"** únicamente a los
participantes ya identificados; ahí pueden ver el historial de mensajes y
escribir, y los mensajes nuevos de cualquier jugador aparecen al instante
(Supabase Realtime) sin recargar la página.

Tabla `competicion_chat_mensajes` (con RLS): solo se puede leer o escribir en
el chat de una competición si existe una fila en
`competicion_inscripciones_padel` con `status = 'activa'` para ese usuario y
esa competición. No hay panel de moderación en `admin.html` por ahora.

## Aviso legal, privacidad y cookies

En el pie de página de `club.html` hay tres enlaces («Aviso legal», «Política
de privacidad» y «Política de cookies») que abren un modal con el texto
correspondiente, además del logo de **EBM Proyectos de Software** y el crédito
"Desarrollado por Ernesto Bermejo Montalvo" (imagen en `assets/logo-ebm.png`).

Al confirmar una reserva, ahora hay que marcar obligatoriamente la casilla
**"He leído y acepto la política de privacidad"** — el propio navegador
bloquea el envío del formulario si no está marcada (atributo `required`), y el
enlace de la casilla abre el mismo modal de privacidad sin perder los datos ya
rellenados en el formulario de reserva.

Los textos de Aviso legal y Política de privacidad identifican como
responsable al **Ayuntamiento de Chozas de Canales** (con placeholders
`[pendiente de completar]` en el CIF, la dirección y el contacto — ver
"Pendiente de tu parte" más abajo) y mencionan a EBM Proyectos de Software
únicamente como encargado del desarrollo técnico, no como responsable del
tratamiento de datos. La Política de cookies describe lo que la web usa hoy
(explicado más abajo) y no una plantilla genérica.

## Multi-tenant: una sola web para muchas organizaciones

Este repositorio ya no es solo "la web de Chozas de Canales": es la base de
código que se despliega, **sin copiarla**, para cualquier cliente nuevo
(otro ayuntamiento, una comunidad de vecinos, un club de pádel privado...).
Cada cliente es una **organización** con sus propios datos, completamente
aislados de los demás en la misma base de datos.

### Cómo funciona el aislamiento

- Tabla `organizaciones` (el "padre"): una fila por cliente, con `nombre`,
  `tipo` (`ayuntamiento` / `comunidad_vecinos` / `club_padel` / `otro`),
  `slug` (identificador corto para la URL) y `logo_path`.
- Tabla `organizacion_admins`: qué emails administran cada organización
  (puede haber varios admins por organización).
- Todas las tablas de datos (`pistas_padel`, `reservas_padel`,
  `clientes_padel`, `competiciones_padel`, `invoices_padel`, etc.) tienen una
  columna `organizacion_id` obligatoria, y las políticas de seguridad a nivel
  de fila (RLS) filtran o bloquean el acceso según esa columna — no es una
  convención en el código, es una barrera en la propia base de datos: aunque
  hubiera un fallo en el frontend, Postgres impediría igualmente que un
  admin de una organización lea o modifique datos de otra.
- Dos funciones ayudantes reutilizadas en casi todas las políticas:
  `is_padel_org_admin(organizacion_id)` (¿soy admin de esta organización
  concreta?) e `is_padel_super_admin()` (tu cuenta, `ernestobm2012@gmail.com`,
  con acceso a todas las organizaciones para soporte/mantenimiento).
- Verificado con pruebas directas en la base de datos: un admin de una
  organización obtiene 0 filas al intentar leer o modificar datos de otra
  organización, y las suyas propias funcionan con normalidad.

### Cómo se elige la organización

Por ahora, sin dominio propio comprado todavía, cada organización se
identifica por un parámetro en la URL: `club.html?org=slug-del-cliente`.
La web pública busca ese `slug` (función `organizacion_por_slug`), carga el
nombre/logo de esa organización y a partir de ahí todas las consultas van
filtradas por su `organizacion_id`. Si el enlace no lleva `?org=...` o el
slug no existe, se muestra un aviso claro en vez de mezclar datos por error.

Más adelante, cuando haya un dominio propio, esto se puede sustituir por
subdominios (`chozas.tudominio.com`) o dominios propios por cliente sin tocar
la lógica de aislamiento — solo cambiaría cómo se resuelve el `slug` al
entrar.

### Panel de administración con varias organizaciones

`admin.html` funciona igual para todos los clientes con el mismo código:
al iniciar sesión, si tu email administra una sola organización entra
directo; si administras varias (por ejemplo, tú como super-admin, o un
gestor que lleva varios clubes), aparece un selector para elegir con cuál
trabajar antes de entrar al panel.

Desde `admin.html` → pestaña **«Organización»**, cada cliente puede
autogestionar sus propios datos sin pedírtelo a ti:
- Cambiar el **nombre** de su organización.
- Subir, cambiar o quitar su propio **logo** (se guarda en el bucket
  `galeria-padel`, como las fotos de pistas/torneos).
- Editar sus datos de **contacto** para la web pública: dirección, teléfono,
  email, redes sociales (Instagram/Facebook/WhatsApp) y la búsqueda que se
  usa para el mapa de Google.

Todos estos cambios se reflejan al momento en su propia web pública
(`club.html`) y en su propio panel — sin tocar código. (Solo tú, desde
`plataforma.html`, puedes cambiar sus **permisos** y activarla/desactivarla
— ver más abajo.)

### Ayuntamientos: factura o tasa municipal

Si la organización es de tipo **ayuntamiento**, en la pestaña «Organización»
aparece una tarjeta extra («Facturación») donde elige qué documento emite al
cobrar:

- **Factura** (por defecto): igual que hasta ahora, con IVA.
- **Tasa municipal**: pensado para el cobro directo de una administración
  pública, sin IVA. Al elegirlo, la pestaña "Facturas" pasa a llamarse
  "Tasa municipal" en todo el panel (título, botones, campo del número,
  documento impreso...) y el campo de IVA del formulario se oculta.

Ese mismo ajuste también se puede fijar desde `plataforma.html` al dar de
alta el ayuntamiento (solo aparece ahí si el tipo de organización es
"Ayuntamiento"). Comunidades de vecinos y clubes de pádel siempre emiten
factura — la elección solo tiene sentido para administraciones públicas.

### Cobro en mano y facturación automática

Todavía no hay pasarela de pago online, así que el cobro real pasa por caja:
el cliente paga en persona (efectivo, Bizum, tarjeta con datáfono aparte...) y
tú lo registras. En cada tarjeta de reserva de la pestaña "Reservas" aparece
un botón **"💰 Cobrar"** (salvo en usos municipales, que no se facturan a un
cliente). Al pulsarlo y confirmar:

- La reserva queda marcada como **pagada** (`pagado = true`, con la fecha y
  hora exacta en `paid_at`).
- Se genera automáticamente **la factura o la tasa municipal** correspondiente
  (según lo que tenga configurado esa organización), con el concepto
  detallado — pista, fecha y franja horaria concretas, no un texto genérico —
  y el importe total de la reserva (precio + luz, si la tuvo). Si la reserva
  ya tenía una factura vinculada, no se duplica.
- Puedes elegir la forma de cobro (efectivo, Bizum o tarjeta) para que quede
  reflejada en el documento.

Cada reserva (no municipal, no cancelada) muestra además un aviso de su
estado de cobro:

- **✓ Pagado**: ya se ha cobrado.
- **⚠ Pendiente de pago**: la fecha de la reserva ya ha pasado y sigue sin
  cobrarse — hay que reclamarlo.
- **🕓 Próximo pago**: la reserva es futura y todavía no se ha cobrado (es
  normal, se cobra el día del partido o después).

Este estado no se guarda como tal en la base de datos: se calcula al vuelo
comparando `pagado` con la fecha de la reserva, así que una reserva pasa sola
de "próximo pago" a "pendiente de pago" en cuanto se cumple la fecha, sin que
nadie tenga que tocar nada.

La pestaña **Estadísticas** reflejan estos tres estados con sus propios
indicadores: "Pagos cobrados" (dinero realmente ingresado — ya no cuenta como
"ganado" lo que todavía no se ha cobrado), "Pendiente de pago" y "Próximos
pagos", cada uno con su importe y número de reservas.

### Permisos: qué módulos tiene contratados cada cliente

Reservas, Calendario, Clientes, Pistas y Precios están siempre incluidos para
cualquier organización. El resto (**Torneos y ligas**, **Ranking**,
**Estadísticas**, **Facturas**) son módulos que puedes activar o desactivar
por cliente, según lo que haya contratado.

Solo tu cuenta (`ernestobm2012@gmail.com`, la de super-admin/"padre") puede
cambiarlo, desde `plataforma.html` (ver más abajo). Al momento:

- Las pestañas correspondientes desaparecen del panel de esa organización
  en `admin.html` (para su propio administrador, no solo para ti).
- Las secciones «Torneos» y «Ranking» desaparecen igual de su web pública
  (`club.html`) si no están contratadas.
- También puedes marcar la organización como **inactiva** (por ejemplo, si
  deja de pagar): su web pública deja de funcionar (muestra un aviso) y su
  panel de administración deja de dejar entrar a su propio admin, pero tú
  sigues viéndola en `plataforma.html` (marcada "Inactiva") para poder
  reactivarla cuando quieras.

Está reforzado a nivel de base de datos: aunque alguien manipulara las
peticiones directamente, un trigger (`protect_organizacion_admin_fields`)
impide que nadie que no sea super-admin cambie los permisos o el estado
activo/inactivo de una organización — un admin de organización solo puede
seguir editando los datos de su propia organización (nombre, logo, contacto),
nada de eso.

## `plataforma.html`: el panel del "padre"

Toda la gestión que solo te corresponde a ti como dueño de la plataforma —dar
de alta clientes nuevos, crear sus administradores, decidir qué módulos tiene
cada uno, activarlos o desactivarlos— vive en un archivo **aparte** de
`admin.html`: `plataforma.html`. No es una pestaña más del panel de cada
club, es una página independiente, con su propio login (restringido también
a tu email), separada de la operativa diaria de cualquier organización.

Al igual que `admin.html`, `plataforma.html` tiene su propio **menú de
pestañas** arriba (Solicitudes de demo / Altas de organización /
Estadísticas / Facturación / Organizaciones): cada una es su propia
pantalla, no todo apilado en una sola página larguísima. En el móvil, si
hay más pestañas de las que caben en el ancho de pantalla, la barra de
pestañas se desliza horizontalmente (como en `admin.html`) en vez de
obligar a alejar el zoom para ver la página entera.

Con varias organizaciones dadas de alta, la lista no muestra todas las
fichas abiertas a la vez: hay un **desplegable** con el nombre de todas las
organizaciones y solo al elegir una se abren debajo sus datos completos
(igual de editables que antes). Cada ficha incluye también tres accesos
directos, cada uno en una pestaña nueva:
- **"Ver web pública →"**: `club.html?org=<slug>`, la web de reservas de esa
  organización, tal cual la ve cualquier vecino/cliente.
- **"Abrir panel de administración →"**: `admin.html?org=<slug>` — así entras
  al panel de un club concreto sin tener que buscarlo en el selector de
  organizaciones de `admin.html` (que solo te lo pregunta si no vienes con ese
  parámetro en la URL, o si tu cuenta administra más de una).
- **"Enviar contrato de alta →"**: `contrato.html?org=<slug>` (ver más abajo).

### Estadísticas de la plataforma

Arriba del todo, antes de la lista de organizaciones, hay un contador de
**visitas a la página principal** (`gestionmypadel.com`, es decir
`index.html`, la web de ventas del producto — no la de ningún club
concreto) con el mismo desglose de 30 días / total, y debajo una tabla con
un resumen de uso de cada organización:

- **Visitas (30 días)** y **visitas (total)**: cada vez que alguien abre la
  web pública de una organización (`club.html?org=...`) se registra un
  evento `page_view`. No distingue visitantes únicos, es un contador simple
  de cargas de página.
- **Reservas este mes**: cuántas reservas (sin contar las canceladas) tiene
  esa organización en el mes en curso.
- **Clics en el banner**: cuántas veces se ha pulsado el banner de publicidad
  de `club.html` estando en la web de esa organización.

Estos datos viven en una tabla, `analytics_eventos_padel` (evento +
organización + fecha), donde cualquier visitante anónimo puede insertar un
evento (para que la propia web pública lo registre sin necesitar login),
pero solo tu cuenta puede leerlos — ni siquiera el administrador de cada
organización ve estos números, es información exclusiva de la plataforma.
Las visitas a `index.html` se guardan en la misma tabla con
`organizacion_id` en blanco y el evento `landing_page_view`, ya que no
pertenecen a ninguna organización concreta.

Desde ahí, para dar de alta un cliente nuevo:

1. **"+ Nueva organización"**: nombre, tipo (ayuntamiento / comunidad de
   vecinos / club de pádel / otro) y el identificador para la URL (slug,
   p. ej. `mi-club-de-padel` → `club.html?org=mi-club-de-padel`). Se crea
   con la base de datos completamente en blanco: sin pistas, sin precios,
   sin clientes ni reservas — nada compartido con el resto de organizaciones.
2. Esa organización aparece debajo como una ficha completa donde rellenas
   **todo lo que la identifica**: nombre, tipo, slug, logo, dirección,
   teléfono, email de contacto, redes sociales (Instagram/Facebook/WhatsApp)
   y el texto de búsqueda para el mapa de su web pública — todo lo que antes
   estaba fijo en el código para Chozas de Canales, ahora es un dato más de
   cada organización.
3. En la misma ficha, marcas qué **permisos** tiene (Torneos, Ranking,
   Estadísticas, Facturas) y si está activa.
4. En **"Administradores"**, escribes el email y una contraseña inicial
   (mínimo 6 caracteres) de la persona que va a gestionarla, y pulsas "Crear
   administrador". Esa cuenta ya puede entrar en `admin.html` con ese email
   y esa contraseña, y podrá cambiarla ella misma más adelante desde
   "¿Has olvidado tu contraseña?" — tú no vuelves a tocarla salvo que te lo
   pidan. Si el email ya tenía cuenta (por ejemplo, la misma persona ya
   administra otro club tuyo), no se toca su contraseña: solo se le añade
   acceso a la organización nueva. El botón "×" junto a un email le quita el
   acceso a esa organización concreta (sin borrar su cuenta, por si
   administra otras).

El botón **«Borrar organización»** (al lado de "Guardar cambios") la borra
para siempre, junto con **todos** sus datos: pistas, reservas, clientes,
torneos, facturas, contratos, todo. Para evitar un borrado accidental te
pide escribir el nombre exacto de la organización antes de confirmar — si
no coincide, no borra nada.

Por seguridad, la contraseña **no se guarda ni se ve** en ningún sitio del
panel: se crea la cuenta de Supabase Auth al vuelo desde una Edge Function
(`admin-create-org-admin-padel`) que solo tu cuenta puede invocar (comprueba
tu email igual que `admin-create-client-padel`), y de ahí en adelante esa
contraseña solo la conoce quien tú se la des.

Con esto, dar de alta un club nuevo no requiere tocar código ni el dashboard
de Supabase: entras en `plataforma.html`, rellenas su ficha y le creas su
administrador, y ese cliente ya tiene su propia web y su propio panel.

## `index.html`: la web de ventas de GestionMyPadel

Además de la web de cada club (`club.html?org=...`) y de los paneles de
gestión, `index.html` es la página de ventas propia del producto en sí —
**GestionMyPadel** — y vive en la raíz del dominio `gestionmypadel.com`
(ver «Dominio propio: gestionmypadel.com» más abajo). Es la página que le
enseñas a un ayuntamiento o un club nuevo antes de darlos de alta: qué hace
el producto, planes y precios, y un formulario para pedir una demo. El pie
de página muestra `info@gestionmypadel.com` como email de contacto de
GestionMyPadel/EBM Proyectos de Software — el mismo que aparece en
`contrato.html` como correo de contacto de "EL PRESTADOR".

Planes actuales (mensuales, sin permanencia, con **2 meses gratis** para
probar antes de pagar):
- **Reservas — 39,99€/mes**: reservas online, calendario, pistas, precios y
  clientes.
- **Torneos y Ligas — 49,99€/mes**: todo lo anterior + torneos, ligas y
  ranking automático.
- **Premium — 59,99€/mes**: todo lo anterior + pasarela de pago online y
  apertura de pista con código domotizado.

> ⚠️ **Importante**: los dos únicos extras del plan Premium (pasarela de pago
> online y apertura domotizada) **todavía no están construidos** en el
> producto — el resto de funciones que aparecen en la web sí existen y
> funcionan de verdad. Antes de vender activamente el plan Premium, hay que
> decidir si se construyen esas dos integraciones o si se ajusta lo que
> promete esa página mientras tanto.

Quien rellena el formulario de la landing queda guardado en la tabla
`leads_gestionmypadel` (con RLS: cualquiera puede enviar una solicitud, pero
solo tu cuenta puede leerlas). Se ven y gestionan desde `plataforma.html`,
en la sección **«Solicitudes de demo»**, arriba del todo: puedes marcar cada
una como nuevo/contactado/convertido/descartado, o borrarla. Cada solicitud
tiene también dos botones para mandarle el enlace de alta al cliente
(**"📋 Copiar"**, que copia `alta-organizacion.html` al portapapeles, y
**"✉️ Email"**, que abre tu programa de correo con un mensaje ya escrito
para ese contacto) — de momento no se manda solo, tienes que darle tú al
botón, ver más abajo por qué.

## `alta-organizacion.html`: que el propio cliente rellene sus datos

Cuando ya has hecho la demo y el cliente quiere seguir adelante, en vez de
teclear tú sus datos en `plataforma.html`, le mandas el enlace de
`alta-organizacion.html` (sin login) para que los rellene él mismo: nombre,
tipo (ayuntamiento / comunidad de vecinos / club de pádel / otro),
dirección, teléfono, email de contacto, Instagram, Facebook, WhatsApp y,
si quiere, su logo.

Al enviarlo se guarda en la tabla `solicitudes_organizacion_padel` (mismo
patrón de RLS que los leads: cualquiera puede enviar una solicitud, solo tu
cuenta puede leerlas) y el logo se sube al bucket público `galeria-padel`
(carpeta `solicitudes/`, con una política de storage que permite subir ahí
sin necesidad de login).

En `plataforma.html`, en la sección **«Solicitudes de alta de
organización»**, ves cada solicitud con su logo y un botón **«Crear
organización →»**: al pulsarlo se crea de golpe la organización con todos
esos datos ya rellenos (incluido el logo) y la marca **inactiva** hasta que
tú revises el slug, le asignes un plan y le crees el acceso de
administrador — todo eso, igual que con cualquier organización, en su
ficha de más abajo.

## Facturación: qué te debe cada organización

En `plataforma.html` hay una sección **«Facturación»** con una tabla
resumen de todas las organizaciones (plan, cuota mensual +IVA, total
cobrado, total pendiente y si el mes en curso está al día, pendiente o
**atrasado**), y dentro de la ficha de cada organización un bloque de
**«Facturación»** con el detalle mes a mes:

- **«+ Generar cobro de este mes»**: crea el cobro del mes en curso con el
  importe del plan contratado (39,99 / 49,99 / 59,99 € — sin IVA, igual que
  en el resto de la web). No deja generar dos cobros del mismo mes para la
  misma organización.
- Cada cobro tiene un desplegable **Pendiente / Pagado**. Al marcarlo como
  **Pagado**:
  1. Se genera automáticamente la **factura en PDF** (con el desglose de
     base + IVA, o solo la tasa si la organización cobra como tasa
     municipal) y se guarda en el bucket privado `facturas-plataforma-padel`.
  2. Se intenta **enviar la factura por email** al `email_contacto` de la
     organización, con un enlace de descarga válido 7 días.
  3. Si el envío automático no está disponible (ver más abajo), te lo dice
     en el propio aviso, pero la factura queda igualmente generada y
     puedes descargarla con el botón **«PDF»** y mandarla tú a mano; el
     botón **«Enviar email»** te deja reintentarlo luego cuando quieras.

Todo esto vive en la tabla `cobros_organizacion_padel` (solo tu cuenta
puede leerla o escribirla — ni el administrador de cada club ve esto,
es tu contabilidad como dueño de la plataforma).

El envío automático de facturas por email ya está **activado** (Edge
Function `send-invoice-email-padel` con tu cuenta de
[Resend](https://resend.com)): al marcar un cobro como pagado, o al pulsar
«Enviar email», el cliente recibe el correo con el enlace de descarga de
su factura, enviado desde `facturacion@gestionmypadel.com`. Si el dominio
todavía no está verificado del todo en Resend, el aviso en pantalla te
dirá el motivo exacto en vez de fallar en silencio.

## `contrato.html`: alta de cliente con firma digital

Cuando un cliente nuevo se va a dar de alta, `plataforma.html` tiene un
enlace **«Enviar contrato de alta →»** en la ficha de cada organización, que
lleva a `contrato.html?org=<slug>`. Es una página pública (sin login) que le
mandas al cliente para que la rellene y firme él mismo:

1. Antes de mandar el enlace, en `plataforma.html` tienes que fijar el
   **«Plan contratado»** de esa organización (Reservas / Torneos y Ligas /
   Premium) — el contrato usa ese plan para calcular la cuota mensual exacta
   (39,99 / 49,99 / 59,99 € + IVA). Si no se ha asignado plan todavía, la
   página se lo dice al cliente y no le deja continuar.
2. El cliente ve el texto completo del contrato a la izquierda (se actualiza
   en vivo según va rellenando sus datos) y un formulario a la derecha:
   nombre/razón social, NIF/CIF, domicilio, email, teléfono e IBAN (para la
   domiciliación cuando empiece a pagar).
3. Firma con el ratón o el dedo en un lienzo (`<canvas>`), sin necesidad de
   ninguna librería externa de firma.
4. Al confirmar, se genera un PDF en el propio navegador (con la librería
   `jsPDF`) con el contrato completo, la firma incrustada y la fecha/hora de
   la firma; se sube al bucket privado `contratos-padel` de Supabase Storage,
   se guarda un registro en la tabla `contratos_padel` (nombre, NIF, IBAN,
   plan, precio, fecha) y el cliente se descarga su copia al momento.
4. Desde `plataforma.html`, en la propia ficha de la organización, tienes el
   listado de **«Contratos firmados»** con un botón para descargar cada PDF
   (mediante una URL firmada temporal — el bucket es privado y solo tu cuenta
   puede leerlo, ya que los contratos incluyen el IBAN del cliente).

El contrato incluye, entre otras, estas cláusulas (revísalas con un abogado
antes de usarlo con clientes reales, esto es una plantilla, no asesoría
legal):
- **2 meses gratis** y, a partir del tercer mes, cobro recurrente mensual
  automático según el plan, salvo baja.
- **Baja durante la prueba gratuita**: hay que avisar con 7 días de
  antelación a que termine la gratuidad; si no se avisa, se entiende que el
  cliente continúa y se activa el cobro del tercer mes en adelante.
- **Baja una vez de pago**: hay que avisar con 1 mes de antelación a la
  fecha de facturación; si se avisa con menos margen, se cobra igualmente
  el mes siguiente.
- Referencia a la protección de datos: los datos del propio contrato se
  tratan para gestionar la relación contractual y la facturación; y, cuando
  el cliente sea una administración pública (o en general trate datos de
  terceros a través de la plataforma), se remite al **Contrato de Encargado
  de Tratamiento de Datos** (documento aparte, ya existe una plantilla en
  `Contrato_Encargado_Tratamiento_EBM.docx`), que debe firmarse como Anexo I.

> ⚠️ Esta plantilla de contrato (y sus plazos de baja) no ha sido revisada
> por un abogado — solo recoge lo que me has pedido en la conversación.

## `manual.html`: manual de uso con capturas

Página estática (sin login, sin conexión a Supabase) con el manual de uso
ilustrado de la plataforma: qué es cada pestaña de `admin.html` y cada
sección de `club.html`, con una captura de pantalla real y los pasos de
«Cómo se usa» para cada una.

Tanto `admin.html` como `plataforma.html` tienen un botón **«❓ Ayuda»** en
la cabecera que abre `manual.html` en una pestaña nueva. Las imágenes van
incrustadas en base64 dentro del propio HTML, así que el archivo es
autocontenido (no depende de ninguna carpeta de imágenes aparte) y funciona
igual en local que ya publicado en GitHub Pages.

## Cómo lo pruebas en local

Solo hace falta un servidor estático simple (los módulos ES no funcionan con `file://`):

```bash
python3 -m http.server 8000
# abre http://localhost:8000
```

## Publicar la web

Se publica sola con **GitHub Pages**: cada `git push` a `main` dispara el
workflow `.github/workflows/deploy-pages.yml`, que sube el repositorio tal
cual (sin build) y lo despliega. En unos segundos queda en
`https://ernestobm2012-tech.github.io/Pista_padel/` (con el nombre exacto
del repositorio — GitHub Pages distingue mayúsculas de minúsculas en esa
URL).

### Dominio propio: gestionmypadel.com

El dominio `gestionmypadel.com` (comprado en Cloudflare) apunta a este mismo
sitio de GitHub Pages en vez de a la URL `github.io`. Dos partes:

1. **En Cloudflare → DNS**, con el proxy en "DNS only" (nube gris, no
   naranja — si no, GitHub Pages no puede emitir el certificado HTTPS):
   - 4 registros `A` en `@` a las IPs de GitHub Pages: `185.199.108.153`,
     `185.199.109.153`, `185.199.110.153`, `185.199.111.153`.
   - 1 registro `CNAME` en `www` a `ernestobm2012-tech.github.io`.
2. **En GitHub → Settings → Pages → Custom domain**: `gestionmypadel.com`.
   El archivo `CNAME` en la raíz del repositorio ya lo deja configurado
   desde el código; una vez GitHub verifique el DNS, activa **"Enforce
   HTTPS"**.

`index.html` (la web de ventas de GestionMyPadel) queda así en la raíz del
dominio — `gestionmypadel.com` — y la app de reservas de cada cliente en
`gestionmypadel.com/club.html?org=<slug>`.

### Posicionarte en Google (SEO)

Del lado del código ya está todo listo para que Google pueda rastrear e
indexar `index.html` (la web de ventas):

- **`robots.txt`** en la raíz: permite el rastreo de la web pública y
  bloquea explícitamente los paneles privados (`admin.html`,
  `plataforma.html`, `contrato.html`, `alta-organizacion.html`,
  `manual.html`), que además ya llevan `<meta name="robots"
  content="noindex, nofollow">` cada uno.
- **`sitemap.xml`** en la raíz: por ahora solo lista `index.html` (la
  página que quieres que aparezca en las búsquedas).
- **Meta tags de `index.html`**: `<link rel="canonical">`, título y
  descripción ya existentes, y las etiquetas Open Graph / Twitter Card
  (`og:title`, `og:description`, `og:image`...) para que se vea bien
  cuando alguien comparta el enlace en WhatsApp, redes sociales, etc.

Lo que falta es **cosa tuya, en tu cuenta de Google** (no puedo hacerlo yo):

1. Entra en [Google Search Console](https://search.google.com/search-console)
   con tu cuenta de Google y añade la propiedad `gestionmypadel.com`.
2. Verifica que eres el dueño del dominio — la forma más simple aquí es
   añadiendo el registro TXT que te proponga Search Console en Cloudflare
   → DNS (parecido a como añadimos los del correo).
3. Una vez verificado, en el menú **«Sitemaps»** de Search Console, envía
   la URL `https://gestionmypadel.com/sitemap.xml`.
4. En **«Inspección de URLs»**, pega `https://gestionmypadel.com/` y pulsa
   **«Solicitar indexación»** para acelerar que Google la rastree por
   primera vez (si no, puede tardar días o semanas en encontrarla sola).

A partir de ahí, aparecer bien posicionado (arriba en los resultados) ya no
depende del código, sino del tiempo, de cuántas páginas enlacen a la tuya y
de si la gente busca términos que la web responde bien — Search Console te
irá diciendo qué búsquedas te traen visitas y con qué posición media.

## Pendiente de tu parte / a revisar

- **Enlaces ya compartidos con `index.html?org=...` han cambiado de
  dirección.** Al mover la web de ventas a la raíz del dominio, la antigua
  web de reservas (`index.html`) pasó a llamarse `club.html`. Si ya le has
  dado a Chozas de Canales (o a cualquier vecino) un enlace del tipo
  `.../index.html?org=chozas-de-canales`, ahora es
  `.../club.html?org=chozas-de-canales` — conviene avisarles o volver a
  compartirlo.
- Sustituye los datos de contacto de ejemplo (teléfono, email, dirección, redes
  sociales, mapa) por los reales en `club.html` (sección «Contacto»).
- Ya se han creado 2 pistas y los 3 precios por duración de ejemplo (1h=3€,
  1h30=4,5€, 2h=6€) en Supabase — ajústalos desde `admin.html` → pestaña
  «Precios» a los reales.
- El extra de "Luz" tiene un precio de ejemplo — ajústalo desde `admin.html`
  → pestaña «Precios» → «Extras».
- El horario de apertura está fijado de 08:00 a 23:00, con horarios de inicio
  cada 30 minutos (constantes `OPEN_TIME`, `CLOSE_TIME`, `START_STEP_MINUTES`
  en `club.html`); cámbialo si tu club tiene otro horario.
- Este proyecto está pensado para poder migrar más adelante a un stack con build
  (ej. Next.js) si el negocio crece — Supabase seguiría siendo la base de datos.
- El «Aviso legal» y la «Política de privacidad» (enlaces en el pie de página de
  `club.html`) llevan datos de ejemplo con `[pendiente de completar]` en el CIF,
  la dirección y el email de contacto del ayuntamiento — complétalos (o pide que
  los revise alguien con conocimientos legales) antes de publicar, ya que son
  textos de referencia, no un documento legal certificado. La «Política de
  cookies» sí describe con exactitud lo que usa hoy la web (sesión, banner de
  publicidad, caché de la PWA) — actualízala si en el futuro añades analítica o
  publicidad con cookies de terceros.
- Falta una **«Política de privacidad» propia de GestionMyPadel/EBM Proyectos
  de Software** (a nivel de proveedor, distinta de la de cada club en
  `club.html`) que recoja cómo tratas los datos de tus clientes: los que
  rellenan el formulario de la landing (`leads_gestionmypadel`) y los que
  firman el contrato de alta (`contratos_padel`: nombre/razón social, NIF,
  domicilio, email, teléfono e IBAN). El propio contrato de alta ya menciona
  esta política y la base legal (ejecución del contrato, art. 6.1.b RGPD) —
  falta publicarla como página (por ejemplo en `landing.html`) y enlazarla
  desde ahí y desde `contrato.html`.
