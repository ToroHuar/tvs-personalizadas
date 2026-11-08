# Modo Papás — diseño

**Fecha:** 2026-11-07 · **Estado:** APROBADO por Toro (2026-11-07 13:28) · **Maqueta de referencia:** `modo-papas/maqueta-cine-v4.html` (capturas `modo-papas/cine-v4-*.png`)

## 1. Para qué y para quién

Los papás de Toro ("los Gorditos") tienen un Google TV. El papá ve poco y se pierde con el control: no sabe volver a los canales, no encuentra su programa, le sobran botones y la pantalla de Google lo distrae con anuncios.

**Éxito =** pueden ver lo suyo **sin pedir ayuda**, ya sea con el control o hablándole al móvil, y la pantalla **les da alegría** al verla.

Lo que dijo Toro (decisiones tomadas):
- Su móvil es Android. El móvil lo usan **ellos y Toro**; Toro a veces está en la casa y a veces lejos.
- Ven Netflix y otras plataformas, YouTube y canales en vivo con la **app Claro Video**.
- Estilo **Cine de noche**: original, único, con profundidad, nunca plano ni genérico.
- **Toda animación tiene que ser fluida**, y el rendimiento es parte de que se sienta bien.
- Las fotos de la familia van **de fondo y en el salvapantallas**.
- El saludo es "Buenos días / Buenas tardes / Buenas noches, **Gorditos**".
- La voz es **obligatoria**. Las frases se entienden en el propio TV y, si no, con **Claude** como respaldo.
- **La voz es para que el papá CONTROLE el TV hablando** (todo: moverse, abrir, entradas, volumen, pausa, buscar), **desde el celular o desde el micrófono del control**. No es que el TV le narre lo que hace (corrección de Toro, 2026-11-07 14:1x).

## 2. Piezas

| Pieza | Dónde | Qué hace |
|---|---|---|
| **Modo Papás** (pantalla de inicio) | TV, fork de Free My TV | Inicio propio: carrusel de apps, "Lo que estabas viendo", fotos |
| **Micrófono del control** | TV | El botón del micrófono abre NUESTRA escucha (no el Asistente de Google) y manda lo dicho al Intérprete |
| **Salvapantallas de fotos** | TV | Fotos de la familia a pantalla completa con fundidos, hora y pie de foto |
| **Servidor del mando** | TV | Recibe órdenes del móvil por la red de la casa o por Tailscale |
| **Intérprete** | TV | Convierte una frase en una acción (local, y Claude como respaldo) |
| **Abridor** | TV | Ejecuta la acción: abrir una app o un título, volumen, pausa, inicio |
| **App "Mando"** | Android | Botones gigantes y el botón 🎤 Hablar; responde hablando |

Todo vive en **un fork de Free My TV en el GitHub de Toro**, con el código nuevo en archivos propios para poder seguir trayendo las mejoras del proyecto original. La app del móvil va en el mismo repositorio.

## 3. Pantalla de inicio (TV)

- **Arriba:** el saludo según la hora ("Buenas tardes, **Gorditos**"), con una foto redonda, la fecha, y a la derecha la hora y el tiempo en grande.
- **Centro: carrusel de TODAS las apps instaladas**, incluida la tienda Google Play.
  - **"TV en vivo · Claro Video" siempre va primera**, y es la que está enfocada al encender el TV o al apretar Inicio.
  - **La app enfocada queda fija en el centro**, grande, con el color de su marca, brillo de cristal, un destello que la cruza, un **aro blanco que late**, una sombra de su color, un reflejo y la etiqueta **"▲ ESTÁS AQUÍ"**. Debajo van su nombre y para qué sirve, en letras grandes.
  - Las apps de los lados se giran hacia el centro, se hunden en 3D y se ven más oscuras.
- **Abajo: "Lo que estabas viendo"**, con póster, título, capítulo y progreso. ▼ baja a esta fila y OK retoma donde iba (usa el Continue Watching de Free My TV, `WatchNext`).
- **Fondo:** una foto de la familia desenfocada, con un zoom lento y teñida del color de la app enfocada, más una viñeta y grano de cine. Cambia cada 20 s con un fundido. El pie de foto va abajo a la derecha.
- **Todo lo del televisor sigue a mano** (pedido por Toro). Al final del carrusel van:
  - **Tienda de apps.**
  - **Entradas:** un panel grande con TODAS las entradas (HDMI 1/2/3…, AV, antena), con el nombre del aparato conectado (por HDMI-CEC) y un punto verde si tiene señal. Usa las entradas de `HomeApps` de Free My TV.
  - **Configuración:** los ajustes del TV o el panel Quick Settings de Free My TV.
  - **Accesibilidad:** abre la accesibilidad del propio TV (TalkBack, letra grande, alto contraste, subtítulos, lupa) con `Settings.ACTION_ACCESSIBILITY_SETTINGS`.
- **Todos los botones del mando conservan su función:** Entradas, Ajustes, Netflix, YouTube, volumen, micrófono… Se configura con *Remote Buttons* de Free My TV.
- **Para que no se pierdan:**
  - Inicio siempre vuelve aquí.
  - En el inicio, Atrás no hace nada.
  - Solo están protegidos los ajustes PROPIOS del Modo Papás (fichas, fotos, voz): se abren manteniendo OK 5 s o desde el móvil de Toro. Los ajustes del TV siguen libres.
  - Sin anuncios, sin sugerencias de Google y sin precios.
- **Voz guía (opcional, APAGADA por defecto):** que el TV diga en voz alta dónde está el foco. No es lo que pidió Toro; queda solo como opción en los ajustes del Modo Papás.

## 4. Salvapantallas

- Entra con un fundido de 1,6 s tras el tiempo sin uso que configure Android, o desde el botón del móvil.
- Muestra las fotos nítidas a pantalla completa. Cada foto se funde con la siguiente en unos 2,4 s y tiene un zoom lento que alterna de dirección; cambia cada unos 9 s.
- Lleva la hora gigante, la fecha, el tiempo y un pie de foto.
- Cualquier botón lo cierra con un fundido.
- **Origen de las fotos:** el sistema de Free My TV (`Photos`/`PhotoInbox`): hasta 100 fotos que Toro manda desde su PC, y opcionalmente desde Immich. El pie de foto sale de un texto que pone Toro, o de la fecha de la foto si no lo pone.

## 5. Animación y rendimiento

- **Regla: fluidez antes que efectos.** Objetivo: 60 fps en el TV de los papás.
- **Carrusel:** las fichas son piezas fijas que solo cambian de posición y tamaño (620 ms, arranque rápido y frenada suave). La ficha que da la vuelta por el borde no cruza la pantalla. El color del fondo cambia en 0,9 s.
- Solo se animan cosas baratas para el TV (posición, escala, opacidad). El desenfoque del fondo se calcula una vez por foto, no en cada cuadro.
- **Medición:** frames de más de 16,6 ms con `dumpsys gfxinfo` en el TV real. Si no llega, se recorta en este orden: grano → reflejo → giro 3D → desenfoque en vivo. La fluidez no se recorta nunca.

## 6. Control por voz (celular y micrófono del control)

**Objetivo:** que el papá haga **con la voz todo lo que hace con el control**. Hay dos formas de hablarle, y las dos van al mismo Intérprete:

**A) Micrófono del control del TV:**
- El servicio de accesibilidad de Free My TV (`RemoteKeyService`, que ya ve las teclas del mando antes que cualquier app) captura el botón 🎤 (`KEYCODE_SEARCH`, `KEYCODE_VOICE_ASSIST` o `KEYCODE_ASSIST`, según el control) y, en vez del Asistente de Google, abre **nuestra escucha** con `SpeechRecognizer` en español usando el micrófono del control.
- Funciona también **dentro de Netflix, YouTube o Claro**: el papá aprieta 🎤 y dice "pausa", "sube el volumen" o "vuelve al inicio".
- En pantalla aparece un aviso grande y bonito: "Te escucho…", luego lo que entendió y lo que va a hacer.
- **Hay que comprobarlo en SU TV:** que se pueda capturar ese botón y que el micrófono del control llegue a nuestra escucha depende del modelo. Es la **primera prueba del plan**. Si su control no lo permite, se usa mantener apretado OK o el botón de Ajustes como botón de "hablar", y siempre queda el celular.

**B) App Mando en el celular:**
- El botón **🎤 Hablar ocupa media pantalla**. Debajo: Canales, Inicio, Volumen + y −, Pausa/Seguir.
- Vibra y suena al empezar y al terminar de escuchar.
- Responde **hablando** ("Listo, poniendo Amor Eterno en Netflix").
- Usa el `SpeechRecognizer` de Android en español.
- **No interpreta nada:** solo manda el texto al TV.

**Intérprete (TV), en dos capas:**
1. **Local:**
   - Quita la cortesía y el relleno ("por favor", "quisiera que me pongas", "háblame", "la serie de", "oye").
   - Detecta la app ("en Netflix") y la intención. Cubre **todos los movimientos**:
     - Navegar: "derecha", "izquierda", "arriba", "abajo", "ok", "atrás", "inicio".
     - Abrir: "abre YouTube", "pon Netflix", "abre la tienda", "abre la configuración", "abre accesibilidad".
     - Ver: "quiero ver Amor Eterno", "pon la misa en YouTube", "sigue con lo que estaba viendo".
     - Canales y entradas: "pon la tele", "cambia a HDMI 1", "pon la laptop", "pon la antena".
     - Reproducción: "pausa", "sigue", "adelanta", "regresa", "siguiente capítulo".
     - Sonido y TV: "sube o baja el volumen", "silencio", "apaga el televisor", "apágalo en una hora".
     - Fotos: "pon las fotos".
   - Busca el resto ("amor eterno") en TMDB con búsqueda difusa.
2. **Respaldo con Claude** (`claude-haiku-5-5`): si la capa local duda, manda la frase y recibe un JSON `{accion, titulo, app}`. Usa la clave de Toro, guardada en el TV como las demás claves (`Secrets`), nunca en el código. Sin internet o sin clave, se queda con la capa local.

**Abridor:**
- Un título se abre directamente con Watchmode (enlace directo a Netflix, Prime o Disney+).
- YouTube se abre con una búsqueda.
- "Canales" o "la tele" abre Claro Video.
- Los movimientos, el volumen, la pausa, atrás, inicio y apagar van con `input keyevent` por el ADB del propio TV (la clase `Adb`), así que funcionan dentro de cualquier app.
- Las entradas se abren igual que desde el panel de Entradas. "Apágalo en una hora" usa el *Sleep Timer* de Free My TV.

**Dudas y errores:** si encuentra dos títulos parecidos, pregunta "¿A o B?". Si no entiende, contesta "No te entendí, ¿qué quieres ver?". Nunca se queda en silencio.

**Pruebas del intérprete:** una lista de 50 o más frases reales de cómo hablan los Gorditos, que tiene que acertar entera antes de instalar.

## 7. Conexión y seguridad

- **Emparejar:** el TV muestra un QR, el móvil lo escanea y recibe una **llave propia por móvil**. Sin esa llave, el TV no obedece. Toro puede quitarle la llave a un móvil.
- El servidor escucha **solo en la red de la casa y en Tailscale**. No hay nada abierto a internet.
- **Toro desde lejos:** Tailscale en el TV y en su móvil, con la misma app Mando. Desde ahí también cambia los favoritos y los ajustes.

## 8. Fuera de alcance (por ahora)

- El TV de la sala y el LG webOS.
- Cambiar los canales dentro de Claro Video por voz ("pon el canal 5"). Se puede estudiar después.
- iPhone.

## 9. Riesgos y lo que falta saber

- **Datos del TV de los papás:** marca, modelo e IP. Además, activar la depuración USB y encontrar el nombre exacto del paquete de Claro Video.
- **Fluidez en su hardware:** se mide antes de dar nada por terminado.
- **Free My TV es un proyecto nuevo** (octubre de 2026, un autor). Mantener el fork al día con el original.
- **Claves:** TMDB, Watchmode y Claude son de Toro y se meten en el TV desde el móvil, nunca en el chat ni en el código.

## 10. Orden de construcción

0. **Prueba rápida en el TV de los papás:** ¿se puede capturar el botón 🎤 del control y escuchar por su micrófono? Esto decide el diseño de la voz A.
1. Fork, compilación e instalación de Free My TV en el **TV de los papás** (este proyecto es solo para su TV; el de la sala no entra).
2. Modo Papás: pantalla, carrusel, "Lo que estabas viendo", voz guía y fondo de fotos. Medir los fps.
3. Salvapantallas de fotos.
4. Servidor del mando, emparejamiento y el Abridor.
5. App Mando: botones y voz.
6. Intérprete local con sus 50 frases, y después el respaldo con Claude.
7. Tailscale para que Toro ayude desde lejos.
8. Instalación en el TV de los papás y prueba con ellos.
