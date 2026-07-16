# Pistas de Pádel — plataforma multi-cliente (Chozas de Canales y otros)

Sitio de una sola página (`index.html`) para reservar pistas de pádel online,
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

## Banner de publicidad

En `index.html` hay un banner flotante fijo (esquina inferior izquierda) que
enlaza a un patrocinador externo (actualmente `1105sports.com`). La imagen
vive en `assets/banner-1105sports.jpg` y el enlace de destino está en el
propio HTML (`<div id="adBanner">`). Cualquiera puede cerrarlo con la "✕";
el cierre se guarda en `localStorage` para que no vuelva a aparecer en ese
navegador. Para cambiar la imagen o el enlace, sustituye el archivo o edita
el `href` directamente — no hay panel de administración para esto, es un
banner fijo, no gestionable desde `admin.html`.

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

Reservar una pista desde `index.html` **exige tener cuenta** (email y
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
independiente**: al registrarse desde `index.html`, la cuenta se marca
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
reservar: la web pública (`index.html`), las reservas manuales desde
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

No hace falta configurar nada más: las claves necesarias ya están
incluidas en la propia Edge Function `notify-new-reservation-padel`, así
que funciona de fábrica en cuanto activas las notificaciones desde un
dispositivo.

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

En `index.html`, cada torneo o liga con las inscripciones ya cerradas
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
`index.html`, buscar `TORNEO_PRINCIPAL_POINTS` / `TORNEO_CONSOLACION_POINTS`
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
quienes tengan una inscripción activa en ella. En `index.html`, las tarjetas
de torneos/ligas muestran un botón **"💬 Chat"** únicamente a los
participantes ya identificados; ahí pueden ver el historial de mensajes y
escribir, y los mensajes nuevos de cualquier jugador aparecen al instante
(Supabase Realtime) sin recargar la página.

Tabla `competicion_chat_mensajes` (con RLS): solo se puede leer o escribir en
el chat de una competición si existe una fila en
`competicion_inscripciones_padel` con `status = 'activa'` para ese usuario y
esa competición. No hay panel de moderación en `admin.html` por ahora.

## Aviso legal, privacidad y cookies

En el pie de página de `index.html` hay tres enlaces («Aviso legal», «Política
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
identifica por un parámetro en la URL: `index.html?org=slug-del-cliente`.
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

Desde `admin.html` → pestaña **«Organización»**, cada cliente puede:
- Cambiar el **nombre** de su organización.
- Subir, cambiar o quitar su propio **logo** (se guarda en el bucket
  `galeria-padel`, como las fotos de pistas/torneos).

Ambos cambios se reflejan al momento en la cabecera de su propia web pública
y de su propio panel — sin tocar código ni pedirte que lo hagas tú.

### Dar de alta un cliente nuevo

Hoy por hoy, dar de alta una organización nueva (ayuntamiento, comunidad o
club) se hace insertando su fila en `organizaciones` y su primer admin en
`organizacion_admins` desde el dashboard de Supabase (no hay todavía un
formulario público de auto-registro, algo deliberado: son clientes que
contratas tú, no un SaaS de alta libre). A partir de ahí, ese cliente ya
tiene su propia web (`index.html?org=su-slug`) y su propio panel, sin
desplegar nada nuevo ni tocar una sola línea de código — y puede personalizar
su nombre y logo él mismo desde `admin.html`.

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
- El «Aviso legal» y la «Política de privacidad» (enlaces en el pie de página de
  `index.html`) llevan datos de ejemplo con `[pendiente de completar]` en el CIF,
  la dirección y el email de contacto del ayuntamiento — complétalos (o pide que
  los revise alguien con conocimientos legales) antes de publicar, ya que son
  textos de referencia, no un documento legal certificado. La «Política de
  cookies» sí describe con exactitud lo que usa hoy la web (sesión, banner de
  publicidad, caché de la PWA) — actualízala si en el futuro añades analítica o
  publicidad con cookies de terceros.
