# Modo Papás — plan de implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Que los papás de Toro ("los Gorditos") usen su Google TV sin pedir ayuda: un inicio propio "Cine de noche" con fotos de la familia, salvapantallas, y control por voz de todo desde el micrófono del control y desde una app Android "Mando".

**Architecture:** Fork de Free My TV (`ToroHuar/tvs-personalizadas`). La lógica que no necesita Android (textos, orden de fichas, cálculo del carrusel, intérprete, protocolo, emparejamiento) va en `papas/src/io/toro/papas/` y se prueba en la PC con `javac` + `java`, como los tests del original. El código del TV que sí usa Android va en archivos nuevos `app/src/io/freemytv/Papas*.java`; los archivos del original se tocan solo con ganchos de pocas líneas para poder seguir trayendo sus mejoras. La app del celular vive en `mando/` y comparte `papas/src`.

**Tech Stack:** Java 8 (javac `-source 8`), Android SDK 34 sin Gradle (`build.sh`: aapt2 → javac → d8 → zipalign → apksigner), Canvas + `Choreographer` para dibujar, `SpeechRecognizer` y `TextToSpeech` de Android, TMDB, Watchmode, API de Claude (`claude-haiku-5-5`), Tailscale, Python 3 para herramientas de PC.

**Spec:** `docs/superpowers/specs/2026-11-07-modo-papas-design.md` (maqueta visual: `docs/superpowers/specs/modo-papas/maqueta-cine-v4.html` y las capturas `cine-v4-*.png` al lado). Lee el spec antes de cada etapa; donde este plan da un valor, sale de ahí o de la maqueta.

## Global Constraints

- Solo para el **TV de los papás**. El TV de la sala y el LG no entran.
- **Toda animación fluida:** objetivo 60 fps en su TV. Cada tarea con animación termina con `tools/papas_fps.py` en el TV real: **pasa si el percentil 95 ≤ 16 ms y los "Janky frames" ≤ 5 %**.
- Si no llega a 60 fps, se recorta **en este orden**: grano → reflejo → giro 3D → desenfoque en vivo. La fluidez no se recorta nunca.
- Solo se animan cosas baratas: posición, escala, opacidad, color. El desenfoque del fondo se calcula **una vez por foto**, nunca por cuadro.
- Curva de movimiento: `cubic-bezier(.22,1,.36,1)`. Carrusel 620 ms; color del fondo 0,9 s; nombre y descripción 450 ms; fotos del fondo cada 20 s con fundido de 2,4 s; salvapantallas: entra en 1,6 s, foto cada 9 s, fundido 2,4 s, zoom lento que alterna de dirección.
- Saludo: 5–11 h "Buenos días", 12–18 h "Buenas tardes", si no "Buenas noches", seguido de ", **Gorditos**". Fechas en español de Perú (`es-PE`).
- "TV en vivo · Claro Video" es **siempre la primera ficha** y la enfocada al encender o al apretar Inicio.
- En el inicio, **Atrás no hace nada**; Inicio siempre vuelve al Modo Papás. Sin anuncios, sin sugerencias de Google, sin precios.
- Solo los ajustes **propios** del Modo Papás están protegidos (mantener OK 5 s, o desde el celular de Toro). Los ajustes del TV siguen libres.
- Voz guía (el TV dice dónde está el foco): **opcional y APAGADA** por defecto.
- Respaldo de voz: modelo exacto `claude-haiku-5-5`; responde un JSON `{accion, titulo, app}` (más `entrada` y `minutos` cuando hace falta).
- **Nada personal en el repo (es público):** ni fotos, ni claves, ni IPs, ni nombres de la red. Los datos del TV van en `C:\Users\ANTHONY\tasks\free-my-tv\tv-gorditos.md` (fuera del repo). Las claves (TMDB, Watchmode, Claude) entran al TV desde el celular de Toro, nunca por el chat ni en el código.
- El servidor del TV escucha **solo** en la red de la casa y en Tailscale (`10/8`, `172.16/12`, `192.168/16`, `100.64/10`, loopback). Nada abierto a internet. Sin llave de celular, el TV no obedece.
- `papas/src` no importa nada de `android.*` (lo revisa `tools/test-papas.sh`). Java 8: nada de `var`, `record`, `switch` con flechas ni bloques de texto.
- Archivos del original que se pueden tocar (solo ganchos cortos): `build.sh`, `tools/check.sh`, `app/AndroidManifest.xml`, `HomeActivity.java`, `RemoteKeys.java`, `Screensaver.java`, `ScreensaverAlone.java`, `Photos.java`, `Secrets.java`, `HomeReceiver.java`. Todo lo demás es archivo nuevo.
- Se compila **sin** `--generic` (así el actualizador del original queda apagado: `Updates.java:68`).
- Después de cada tarea: commit, `git pull --rebase` y push (regla de Toro). Al empezar cada etapa: `git fetch upstream && git merge upstream/main` y `bash tools/check.sh android`.
- Los pasos marcados **👤 Toro** los hace él en el TV o en el celular; el ejecutor le dice exactamente qué apretar, en palabras simples, y espera su respuesta.

## Review Focus

1. **Mantener ▶ apretado** (el control repite la tecla 10–20 veces por segundo): el carrusel debe seguir suave, retomar la animación desde donde va sin saltos y no quedarse "debiendo" movimientos al soltar. → test `retargetFromCurrent` en la Tarea 2.2 y escena `carrusel-rapido` de `papas_fps.py`.
2. **Sin internet o con TMDB/Watchmode/Claude caídos:** el inicio, los movimientos, el volumen y abrir apps siguen funcionando por voz; buscar un título contesta una frase, nunca silencio. → test `failedLookupSpeaks` en la Tarea 4.4 y `claudeUnavailableKeepsLocal` en la 6.2.
3. **Falta una app** (Claro Video desinstalada o con otro paquete, Netflix sin instalar): no se cae; la ficha no aparece y la voz dice "No encuentro Claro Video en el televisor". → test `missingClaroIsSkipped` en la Tarea 2.2 y `missingAppSpeaks` en la 4.4.
4. **Cero o una sola foto de la familia:** el fondo y el salvapantallas usan el fondo de respaldo o la única foto sin fundirse consigo misma. → test `slidesWithZeroOrOne` en la Tarea 2.4.
5. **Frase vacía, ruido o muletillas solas** ("eh", "oye", "este…"): contesta "No te entendí, ¿qué quieres ver?". → líneas `#ruido` en `tests/papas/frases.tsv` (Tarea 6.1).

---

## Mapa de archivos

**Lógica pura (`papas/src/io/toro/papas/`, probada en la PC):**

| Archivo | Responsabilidad |
|---|---|
| `Words.java` | Saludo, fechas y pies de foto en español |
| `TileOrder.java` | Orden de las fichas del carrusel |
| `Ease.java` | La curva `cubic-bezier(.22,1,.36,1)` |
| `Carousel.java` | Posición, tamaño, giro y opacidad de cada ficha; animación que se puede redirigir |
| `Effects.java` | Qué efectos quedan encendidos según el nivel de recorte |
| `Slides.java` | Qué foto sigue (fondo y salvapantallas) |
| `Json.java` | JSON mínimo (leer y escribir) sin `org.json` |
| `Protocol.java` | Rutas y formas de los mensajes TV ↔ Mando |
| `Pairing.java` | Códigos de 6 dígitos, llaves por celular, roles, intentos fallidos |
| `NetRules.java` | Qué direcciones pueden hablarle al TV |
| `Command.java` | Una orden ya entendida (acción + datos) |
| `Keys.java` | Acción → código de tecla de Android |
| `Titles.java` | Búsqueda difusa de títulos y detección de "¿A o B?" |
| `Replies.java` | Las frases que contesta el TV |
| `TextClean.java` | Quita cortesía y relleno |
| `Interpreter.java` | Frase → `Command` (capa local) |
| `ClaudeFallback.java` | Arma el pedido a Claude y lee su respuesta |

**TV (`app/src/io/freemytv/`, archivos nuevos):** `PapasMode.java` (encendido, ajustes propios), `PapasView.java` (pantalla de inicio), `PapasTiles.java` (fichas desde las apps instaladas), `PapasBackdrop.java` (fondo de fotos), `PapasRow.java` ("Lo que estabas viendo"), `PapasInputs.java` (panel de Entradas), `PapasSettingsActivity.java`, `PapasGuide.java` (voz guía), `PapasCaptions.java`, `PapasDreamView.java`, `PapasPhotosActivity.java`, `PapasServer.java`, `PapasServerService.java`, `PapasOpener.java` (el Abridor), `PapasLookup.java` (TMDB + Watchmode), `PapasNotice.java` (aviso en pantalla), `PapasListenService.java` (micrófono del control), `PapasClaude.java`.

**Celular (`mando/`):** `build.sh`, `AndroidManifest.xml`, `res/`, `src/io/toro/mando/{MainActivity,PairActivity,AdminActivity,MandoClient,MandoVoice}.java`.

**Herramientas y tests:** `tools/test-papas.sh`, `tools/papas_fps.py`, `tools/papas_fotos.py`, `tests/papas/*Test.java`, `tests/papas/frases.tsv`, `tests/papas/test_papas_fps.py`, `tests/papas/gfxinfo-muestra.txt`, `tests/papas/claude-respuesta.json`.

---

## Etapa 0 — Prueba del botón 🎤 en el TV de los papás

Decide el diseño de la voz A (spec §6 A). Se hace en una rama de prueba que **no se fusiona**.

### Tarea 0.1: Preparar la PC y conectar el TV

**Files:**
- Create: `C:\Users\ANTHONY\tasks\free-my-tv\tv-gorditos.md` (fuera del repo)

**Interfaces:**
- Produces: `ANDROID_HOME` apuntando al SDK; `adb connect <IP>:5555` funcionando; en `tv-gorditos.md`: marca, modelo, versión de Android, IP, paquete de Claro Video, servicio de reconocimiento de voz.

- [ ] **Step 1: Instalar el SDK de Android.** Bajar las "command-line tools" de Windows desde developer.android.com (pedirle permiso a Toro antes, diciendo nombre, origen y tamaño), descomprimir en `C:\Users\ANTHONY\Android\Sdk\cmdline-tools\latest\` y correr `sdkmanager "platforms;android-34" "build-tools;34.0.0"`. Poner `ANDROID_HOME=C:\Users\ANTHONY\Android\Sdk`.
  Verificar: `ls "$ANDROID_HOME/build-tools/34.0.0/aapt2.exe" "$ANDROID_HOME/platforms/android-34/android.jar"` → existen los dos.
- [ ] **Step 2: 👤 Toro** activa la depuración en el TV de los papás (Ajustes → Sistema → Información → apretar 7 veces "Compilación del SO"; luego Opciones de desarrollador → Depuración por USB / por red) y dice marca, modelo e IP (Ajustes → Red).
- [ ] **Step 3: Conectar.** `adb connect <IP>:5555`; 👤 Toro acepta "¿Permitir depuración?" con "Permitir siempre".
  Verificar: `adb devices` → la IP aparece como `device`.
- [ ] **Step 4: Anotar datos del TV** en `tv-gorditos.md`: `adb shell getprop ro.product.manufacturer`, `ro.product.model`, `ro.build.version.release`; `adb shell pm list packages | grep -i claro` (paquete de Claro Video); `adb shell settings get secure voice_recognition_service` (quién reconoce la voz; si está vacío o es de Google, anotarlo); `adb shell pm list packages | grep -i katniss`.
- [ ] **Step 5: Ver qué tecla manda el 🎤.** `adb shell getevent -lt` mientras 👤 Toro aprieta el 🎤 una vez; y `adb logcat -d | grep -i "keycode"`. Anotar el `KEYCODE_*` (se espera `KEYCODE_SEARCH` 84, `KEYCODE_VOICE_ASSIST` 231 o `KEYCODE_ASSIST` 219).

### Tarea 0.2: Sonda del micrófono (rama `sonda/microfono`, no se fusiona)

**Files (solo en la rama de prueba):**
- Create: `app/src/io/freemytv/SondaVoz.java` (servicio en primer plano tipo micrófono con `SpeechRecognizer`)
- Modify: `app/src/io/freemytv/RemoteKeys.java:103` (al inicio de `onKey`: anotar cada tecla con `AppLog.w("sonda key " + code)` y, si es la del Step 5 de la 0.1, arrancar `SondaVoz` y devolver `true`)
- Modify: `app/AndroidManifest.xml` (`RECORD_AUDIO` y el servicio con `android:foregroundServiceType="microphone"`)
- Create: `docs/superpowers/notes/2026-11-sonda-microfono.md` (resultado, sin datos personales)

- [ ] **Step 1:** `git switch -c sonda/microfono main`; copiar `config.example.properties` a `config.properties` (Lima: `latitude=-12.0464`, `longitude=-77.0428`, `city=Lima`, `tmdb_region=PE`, `currency=PEN`, `language=es`, `show_crypto=false`, `disable_assistant=false`). `disable_assistant` se queda en `false`: el reconocedor de voz del sistema puede depender del Asistente.
- [ ] **Step 2:** Escribir `SondaVoz`: al arrancar pide `SpeechRecognizer.createSpeechRecognizer(this)` con `RecognizerIntent.EXTRA_LANGUAGE = "es-PE"`, y en `onResults`/`onError` escribe en el log `"sonda voz: <texto>"` o `"sonda voz error <código>"`; después se detiene.
- [ ] **Step 3:** `bash build.sh` → `Built FreeMyTV.apk`. `adb install -r FreeMyTV.apk`. `adb shell pm grant io.freemytv android.permission.RECORD_AUDIO`. Leer `adb shell settings get secure enabled_accessibility_services` y **añadir** (no reemplazar) `io.freemytv/io.freemytv.RemoteKeyService`.
- [ ] **Step 4: 👤 Toro** aprieta 🎤 y dice "hola" en tres lugares: en el inicio, dentro de Netflix y dentro de YouTube. En la PC: `adb logcat -d | grep "sonda"`.
  Esperado por cada lugar: una línea `sonda key <código>` y una `sonda voz: hola`; el Asistente de Google **no** se abre.
- [ ] **Step 5: Si el 🎤 no llega:** repetir el Step 4 con mantener OK y con el botón de Ajustes como botón de "hablar".
- [ ] **Step 6: Decidir y anotar** en `2026-11-sonda-microfono.md` una de estas: `VOZ_A=microfono` (llega la tecla y la voz), `VOZ_A=ok-largo` o `VOZ_A=ajustes` (llega otra tecla y la voz), `VOZ_A=solo-celular` (no llega la voz del control: apretar 🎤 mostrará "Háblale al celular"). Copiar el log relevante sin IPs.
- [ ] **Step 7:** Desinstalar la sonda (`adb uninstall io.freemytv`). En `main`: copiar ahí la nota, commit `docs: resultado de la sonda del micrófono`, `git pull --rebase`, push. La rama `sonda/microfono` se sube pero no se fusiona.

---

## Etapa 1 — El fork compilado e instalado en el TV de los papás

Antes de empezar, crear la rama de trabajo `modo-papas` desde `main` (con este plan ya fusionado). Todas las tareas siguientes van ahí; al cerrar cada etapa que 👤 Toro aprobó en el TV, se fusiona en `main`.

### Tarea 1.1: Instalación base y medidor de fps

**Files:**
- Create: `tools/papas_fps.py`, `tests/papas/test_papas_fps.py`, `tests/papas/gfxinfo-muestra.txt`

**Interfaces:**
- Produces: `python tools/papas_fps.py <escena> [--serial IP:5555]` → imprime `escena=<x> frames=<n> janky=<p>% p95=<ms>ms OK|LENTO` y sale con 0 si pasa, 1 si no. Función `parse(texto: str) -> dict` con claves `frames`, `janky_pct`, `p95_ms`.
  Escenas: `carrusel` (20 × ▶ cada 700 ms y 20 × ◀), `carrusel-rapido` (40 × ▶ cada 60 ms), `fila` (▼▲ 10 veces), `entradas`, `fotos-fondo` (espera 45 s), `salvapantallas` (abre `PapasPhotosActivity`, espera 30 s). Proceso medido: `io.freemytv:home`. Cada escena empieza con `dumpsys gfxinfo io.freemytv:home reset`.

- [ ] **Step 1:** Compilar el original tal cual (`config.properties` de la Tarea 0.2), instalar, y desde Ajustes de Free My TV cambiar al modo Free My TV. Guardar `freemytv.keystore` también fuera del repo (`C:\Users\ANTHONY\tasks\free-my-tv\llaves\`): sin esa llave no se puede actualizar la app.
- [ ] **Step 2:** Revisar en el TV: Inicio abre el inicio de Free My TV; los botones del control (Netflix, YouTube, volumen, Entradas) hacen lo de siempre (Ajustes → Remote Buttons si hace falta); no aparece aviso de actualización.
- [ ] **Step 3:** Guardar la salida real de `adb shell dumpsys gfxinfo io.freemytv:home` después de mover el estante 20 veces en `tests/papas/gfxinfo-muestra.txt`.
- [ ] **Step 4: Test que falla.** `tests/papas/test_papas_fps.py`: `test_parse_muestra` comprueba que `parse(open(muestra).read())` devuelve `frames` > 0, `0 ≤ janky_pct ≤ 100` y `p95_ms` igual al número de la línea `95th percentile` de la muestra; `test_veredicto` comprueba `veredicto({"p95_ms":16,"janky_pct":5.0}) == "OK"` y `veredicto({"p95_ms":17,"janky_pct":1.0}) == "LENTO"`. Correr `python tests/papas/test_papas_fps.py` → falla (no existe `papas_fps`).
- [ ] **Step 5:** Escribir `tools/papas_fps.py` con `parse`, `veredicto` y las escenas. Correr el test → pasa.
- [ ] **Step 6:** Medir la línea base del original: `python tools/papas_fps.py carrusel` (en el estante del original). Anotar el resultado en `tv-gorditos.md`: es el techo del hardware.
- [ ] **Step 7:** Commit `feat(papas): medidor de fps para el TV`, `git pull --rebase`, push.

---

## Etapa 2 — Modo Papás: la pantalla de inicio

### Tarea 2.1: Textos en español y banco de pruebas de `papas/`

**Files:**
- Create: `papas/src/io/toro/papas/Words.java`, `tests/papas/WordsTest.java`, `tools/test-papas.sh`
- Modify: `build.sh:224` (añadir `papas/src` al `find` de javac), `tools/check.sh` (correr `bash tools/test-papas.sh` en la parte `android`)

**Interfaces:**
- Produces: `Words.greeting(int hour) -> String`; `Words.homeDate(java.util.Calendar) -> String` ("SÁBADO · 7 DE NOVIEMBRE"); `Words.saverDate(Calendar) -> String` ("Sábado 7 de noviembre"); `Words.clock(int hour, int minute) -> String[]` (`{"4:45","pm"}`); `Words.captionFromDate(int year, int month1to12) -> String` ("Marzo de 2019"). Los tests de `tests/papas/` siguen el estilo de `tests/VisibleOrderTest.java` (`main` + `check(cond, motivo)`), en el paquete `io.toro.papas`.
- `tools/test-papas.sh`: compila `papas/src` y `tests/papas` con `javac --release 8 -encoding UTF-8`, corre cada `*Test`, corre `python tests/papas/test_papas_fps.py`, y falla si `grep -rl "import android" papas/src` encuentra algo.

- [ ] **Step 1: Test que falla.** `WordsTest`: `greeting(5)=="Buenos días"`, `greeting(11)=="Buenos días"`, `greeting(12)=="Buenas tardes"`, `greeting(18)=="Buenas tardes"`, `greeting(19)=="Buenas noches"`, `greeting(4)=="Buenas noches"`; `homeDate(7 nov 2026)=="SÁBADO · 7 DE NOVIEMBRE"`; `saverDate(7 nov 2026)=="Sábado 7 de noviembre"`; `clock(16,45)` → `{"4:45","pm"}`, `clock(0,5)` → `{"12:05","am"}`, `clock(12,0)` → `{"12:00","pm"}`; `captionFromDate(2019,3)=="Marzo de 2019"`.
- [ ] **Step 2:** Escribir `tools/test-papas.sh`; `bash tools/test-papas.sh` → falla al compilar (`Words` no existe).
- [ ] **Step 3:** Implementar `Words` con nombres de días y meses escritos a mano (no depender del idioma del sistema).
- [ ] **Step 4:** `bash tools/test-papas.sh` → todos pasan. `bash build.sh` → compila con `papas/src`.
- [ ] **Step 5:** Commit `feat(papas): saludo y fechas en español`, `git pull --rebase`, push.

### Tarea 2.2: Orden de las fichas y cálculo del carrusel

**Files:**
- Create: `papas/src/io/toro/papas/{TileOrder,Ease,Carousel}.java`, `tests/papas/{TileOrderTest,EaseTest,CarouselTest}.java`

**Interfaces:**
- Produces:
  - `TileOrder.order(List<String> installedPkgs, String claroPkg, List<String> hidden) -> List<String>`: Claro primero (si está instalado), luego las demás apps en el orden recibido, luego las fijas `"@tienda"`, `"@entradas"`, `"@configuracion"`, `"@accesibilidad"`; quita las de `hidden` (las fijas no se pueden ocultar).
  - `Ease.at(float t) -> float`: la curva `cubic-bezier(.22,1,.36,1)` (resolver x por Newton/bisección, devolver y).
  - `Carousel.Slot { float x, z, rotY, opacity, w, h; boolean focused; boolean jump; }` y `Carousel.layout(int n, int focus, float width, int[] prevOffset) -> Slot[]`. Proporciones de la maqueta sin los topes del navegador: `big = 0.27·W`, `bh = 0.72·big`, `sm = 0.125·W`, `gap = 0.035·W`; desplazamiento `k` envuelto a `[-n/2, n/2]`, `m=|k|`; `x = m==0 ? 0 : sign(k)·(big/2 + gap + sm/2 + (m−1)·(sm + 0.6·gap))`; `z = focused ? 60 : −70·m`; `rotY = −sign(k)·min(16·m, 34)`; `opacity = m≥4 ? 0 : m==3 ? .22 : m==2 ? .6 : 1`; `jump = |prevOffset − k| > 2` (la ficha que da la vuelta se coloca sin animar).
  - `Carousel.Anim`: `void to(Slot[] target, long nowMs)`, `Slot[] at(long nowMs)` (interpola con `Ease.at` en 620 ms desde la posición **actual**), `boolean running(long nowMs)`.

- [ ] **Step 1: Tests que fallan.**
  - `TileOrderTest.claroFirst`: `order([netflix, claro, youtube], claro, [])` = `[claro, netflix, youtube, @tienda, @entradas, @configuracion, @accesibilidad]`.
  - `TileOrderTest.missingClaroIsSkipped`: `order([netflix], "com.claro.x", [])` empieza con `netflix` y no lanza excepción.
  - `TileOrderTest.hiddenAppsGoButFixedStay`: ocultar `netflix` y `@entradas` → sale `netflix`, queda `@entradas`.
  - `EaseTest`: `at(0)==0`, `at(1)==1`, `|at(0.5) − 0.961| < 0.005`, y es creciente en 100 puntos.
  - `CarouselTest.centerAndSides`: con `n=11, focus=0, W=1760`: la ficha 0 tiene `x=0, focused=true, w=475.2`; la 1 tiene `x>0, rotY=−16, opacity=1`; la 10 tiene `x<0, rotY=16`; la 4 tiene `opacity=0`.
  - `CarouselTest.wrapJumps`: de `focus=0` a `focus=1` con `n=11`, la ficha que pasa de `k=−5` a `k=5` tiene `jump=true`; las demás `false`.
  - `CarouselTest.retargetFromCurrent`: `to(A, 0)`, luego a los 100 ms `to(B, 100)`; `at(100)` antes y después del segundo `to` es igual (sin salto); `running(100+620)` es `false` y `at(720)` = `B`.
- [ ] **Step 2:** `bash tools/test-papas.sh` → fallan.
- [ ] **Step 3:** Implementar `TileOrder`, `Ease`, `Carousel`.
- [ ] **Step 4:** `bash tools/test-papas.sh` → pasan.
- [ ] **Step 5:** Commit `feat(papas): orden de fichas y cálculo del carrusel`, `git pull --rebase`, push.

### Tarea 2.3: La pantalla de inicio con el carrusel

**Files:**
- Create: `app/src/io/freemytv/PapasMode.java`, `PapasTiles.java`, `PapasView.java`
- Modify: `app/src/io/freemytv/HomeActivity.java` (en `onCreate`, `setContentView(PapasMode.on(this) ? new PapasView(this) : home)`; los usos de `home` quedan protegidos con `if (home != null)`)

**Interfaces:**
- Consumes: `TileOrder.order`, `Carousel.layout`, `Carousel.Anim`, `Words.*`, `HomeApps.shelf/storePackage/open/tvSettings`, `HomeData.get(c)` (hora y tiempo).
- Produces:
  - `PapasMode.on(Context) -> boolean` (pref `papas_on`, encendida por defecto en este fork); `PapasMode.claroPackage(Context) -> String` (pref `papas_claro`, valor de `tv-gorditos.md`); `PapasMode.hidden(Context) -> List<String>`.
  - `PapasTiles.Tile { String key, label, sub; int brand; Bitmap face; }` y `PapasTiles.load(Context) -> List<Tile>`. La tienda de Google Play no se repite como app suelta: es la ficha `@tienda`. Cada ficha se dibuja **una sola vez** en un `Bitmap` a tamaño grande (degradado de su marca, brillo de cristal, logo y nombre); en cada cuadro solo se transforma. Colores y textos de las fijas y conocidas como en la maqueta (`apps[]` de `maqueta-cine-v4.html`); para el resto, color sacado del banner de la app.
  - `PapasView extends View implements Choreographer.FrameCallback`: dibuja el encabezado (foto redonda, saludo, fecha, hora y tiempo), el carrusel con `android.graphics.Camera` para el giro 3D, la ficha enfocada con aro blanco que late (2,2 s), destello que la cruza (4 s), sombra de su color, reflejo y la etiqueta "▲ ESTÁS AQUÍ" que flota; las fichas de los lados más oscuras (brillo 0,55, saturación 0,7); nombre y descripción debajo con fundido y subida de 450 ms. Pide cuadros solo mientras algo se mueve. `void focusFirst()` (Claro) se llama en `onResume` del Activity y al recibir Inicio.
  - Teclas: ◀ ▶ mueven; OK abre con `HomeApps.open`; Atrás devuelve `true` sin hacer nada.

- [ ] **Step 1:** Implementar `PapasMode`, `PapasTiles`, `PapasView` y el gancho en `HomeActivity`.
- [ ] **Step 2:** `bash tools/check.sh android` → compila y pasan los tests.
- [ ] **Step 3:** Instalar en el TV. Captura: `adb exec-out screencap -p > out/papas-inicio.png`; compararla con `docs/superpowers/specs/modo-papas/cine-v4-inicio.png` (orden, ficha enfocada, etiqueta, saludo con "Gorditos", hora, tiempo de Lima).
- [ ] **Step 4:** Revisar con teclas por adb: `input keyevent 22` ×3 → el foco avanza y se ve deslizar (no saltar); `input keyevent 4` → no pasa nada; abrir Netflix con OK y luego `input keyevent 3` → vuelve al Modo Papás con Claro enfocada.
- [ ] **Step 5:** `python tools/papas_fps.py carrusel` y `python tools/papas_fps.py carrusel-rapido` → los dos `OK`. Si no, recortar en el orden de las Global Constraints (con `Effects`, Tarea 2.4) y volver a medir.
- [ ] **Step 6:** Commit `feat(papas): pantalla de inicio con carrusel`, `git pull --rebase`, push.

### Tarea 2.4: Fondo de fotos de la familia y recorte de efectos

**Files:**
- Create: `papas/src/io/toro/papas/{Effects,Slides}.java`, `tests/papas/{EffectsTest,SlidesTest}.java`, `app/src/io/freemytv/PapasBackdrop.java`
- Modify: `app/src/io/freemytv/PapasView.java` (dibuja el fondo debajo de todo)

**Interfaces:**
- Produces:
  - `Effects.ORDER = {"grano","reflejo","giro3d","desenfoque"}`; `Effects.on(int trimLevel, String effect) -> boolean` (nivel 0 = todo encendido; nivel 1 quita el grano; …; nivel 4 quita también el desenfoque en vivo). Pref `papas_trim`, se cambia en los ajustes propios.
  - `Slides.next(int current, int count) -> int` (`-1` si `count==0`; `0` siempre si `count==1`); `Slides.shouldFade(int from, int to) -> boolean` (`false` si son la misma).
  - `PapasBackdrop`: `void start(Context, Runnable invalidate)`, `void draw(Canvas, int brandColor)`, `void setBrand(int color)` (el tinte pasa al color nuevo en 0,9 s). Cada foto de `Photos.files(c)` se reduce y desenfoca **una vez** al cargarla (en un hilo aparte, con `Backdrop`/`Frost` del original o un desenfoque por reducción); zoom lento de 26 s que alterna de dirección; fundido de 2,4 s cada 20 s; encima el tinte, el brillo central, la sombra, la viñeta y el grano (un `Bitmap` de ruido hecho una vez y repetido). Pie de foto abajo a la derecha con `PapasCaptions.caption(c, file)` (Tarea 3.1; hasta entonces, vacío). Sin fotos: degradado oscuro "Cine de noche" con el brillo del color de la app.

- [ ] **Step 1: Tests que fallan.** `EffectsTest`: nivel 0 → los cuatro encendidos; nivel 1 → `grano` apagado y `reflejo` encendido; nivel 3 → solo `desenfoque` encendido; nivel 4 → ninguno. `SlidesTest.slidesWithZeroOrOne`: `next(0,0)==-1`, `next(0,1)==0`, `shouldFade(0,0)==false`; `next(3,4)==0`.
- [ ] **Step 2:** `bash tools/test-papas.sh` → fallan. Implementar `Effects` y `Slides`. → pasan.
- [ ] **Step 3:** Implementar `PapasBackdrop` y conectarlo en `PapasView` (`setBrand` al mover el foco).
- [ ] **Step 4:** Mandar 3 fotos de prueba con `python tools/photos.py` (las de Toro llegan en la Etapa 3). En el TV: el fondo cambia de foto con fundido, el tinte sigue a la app enfocada.
- [ ] **Step 5:** `python tools/papas_fps.py fotos-fondo` y `python tools/papas_fps.py carrusel` → `OK`. Probar también con 0 fotos (`Photos.removeAll` desde los ajustes) → fondo de respaldo, sin errores en `adb logcat -d | grep -i freemytv`.
- [ ] **Step 6:** Commit `feat(papas): fondo con fotos de la familia`, `git pull --rebase`, push.

### Tarea 2.5: "Lo que estabas viendo"

**Files:**
- Create: `app/src/io/freemytv/PapasRow.java`
- Modify: `app/src/io/freemytv/PapasView.java`

**Interfaces:**
- Consumes: `WatchNext.load(c) -> List<WatchNext.Item>` (pide `READ_TV_LISTINGS`, que el modo Free My TV ya concede).
- Produces: `PapasRow`: hasta 3 tarjetas de cristal con póster, título, capítulo y barra de progreso (`position/duration`); `void draw(Canvas, float y, int focusIndex /* -1 si la fila no tiene el foco */)`; `boolean open(Activity, int index)`. Sin nada guardado: una tarjeta "Aquí aparecerá lo que veas". ▼ baja a la fila (la etiqueta "ESTÁS AQUÍ" se esconde y la ficha central pierde el aro), ▲ sube, OK retoma.

- [ ] **Step 1:** Implementar `PapasRow` y el manejo de ▼ ▲ en `PapasView`.
- [ ] **Step 2:** En el TV: 👤 Toro deja un capítulo a medias en una app que lo anote (Prime Video o Disney+), vuelve al inicio → aparece con su progreso; ▼ y OK → retoma ahí.
- [ ] **Step 3:** `python tools/papas_fps.py fila` → `OK`.
- [ ] **Step 4:** Commit `feat(papas): fila Lo que estabas viendo`, `git pull --rebase`, push.

### Tarea 2.6: Tienda, Entradas, Configuración y Accesibilidad

**Files:**
- Create: `app/src/io/freemytv/PapasInputs.java`
- Modify: `app/src/io/freemytv/PapasView.java`

**Interfaces:**
- Consumes: `HomeApps.allInputs(c)`, `HomeApps.storePackage(c)`, `HomeApps.tvSettings()`, `TvInputManager.getInputState(id)`.
- Produces:
  - `PapasInputs.Entry { String label, device; boolean signal; HomeApps.Item item; }`; `PapasInputs.list(Context) -> List<Entry>`: todas las entradas (HDMI 1/2/3…, AV, antena), con el nombre del aparato por HDMI-CEC y `signal` = `INPUT_STATE_CONNECTED`. Si `allInputs` no trae AV o la antena, añadirlas desde `TvInputManager.getTvInputList()` por tipo (`TYPE_COMPOSITE`, `TYPE_TUNER`).
  - `PapasInputs.Panel`: panel de cristal que entra con fundido y subida (450 ms / 550 ms), punto verde si hay señal, ▲ ▼ elegir, OK cambia, Atrás cierra. Como la maqueta con `?pan=1`.
  - `@tienda` abre `storePackage`; `@configuracion` abre los ajustes del TV; `@accesibilidad` abre `Settings.ACTION_ACCESSIBILITY_SETTINGS`.
  - `PapasInputs.open(Activity, String nameOrDevice) -> boolean` (lo usa también el Abridor: "HDMI 1", "laptop", "antena").

- [ ] **Step 1:** Implementar `PapasInputs` y las cuatro fichas fijas.
- [ ] **Step 2:** En el TV: cada ficha abre lo suyo; en Entradas, la que tiene algo encendido muestra el punto verde y su nombre; OK cambia a esa entrada; Inicio vuelve.
- [ ] **Step 3:** `python tools/papas_fps.py entradas` → `OK`.
- [ ] **Step 4:** Commit `feat(papas): tienda, entradas, configuración y accesibilidad`, `git pull --rebase`, push.

### Tarea 2.7: Ajustes propios protegidos y voz guía

**Files:**
- Create: `app/src/io/freemytv/PapasSettingsActivity.java`, `app/src/io/freemytv/PapasGuide.java`
- Modify: `app/src/io/freemytv/PapasView.java`, `app/AndroidManifest.xml` (la actividad, proceso `:home`, no exportada)

**Interfaces:**
- Produces:
  - `PapasSettingsActivity`: se abre **solo** manteniendo OK 5 s en el inicio (o desde el celular de Toro, Tarea 5.3). Opciones: mostrar u ocultar cada app del carrusel, fondo de fotos sí/no, nivel de recorte (`papas_trim`), voz guía sí/no (**no** por defecto), botón de hablar (`microfono`/`ok-largo`/`ajustes`/`solo-celular`, por defecto el resultado de la Etapa 0), respaldo con Claude sí/no, "Emparejar un celular" (Tarea 4.3), "Modo Papás encendido" (apagarlo vuelve al inicio del original).
  - `PapasGuide.say(Context, String)`: `TextToSpeech` en español; no hace nada si la voz guía está apagada. `PapasView` la llama al mover el foco ("Netflix. Series y películas") y al abrir ("Abriendo Netflix").

- [ ] **Step 1:** Implementar las dos clases y la detección de OK mantenido 5 s (OK corto sigue abriendo la ficha al soltar).
- [ ] **Step 2:** En el TV: OK 4 s → no abre los ajustes; OK 5 s → los abre. Ocultar una app → desaparece del carrusel. Encender la voz guía → dice el nombre al moverse; apagarla → silencio.
- [ ] **Step 3:** `python tools/papas_fps.py carrusel` con la voz guía encendida → `OK`.
- [ ] **Step 4:** Commit `feat(papas): ajustes propios y voz guía opcional`, `git pull --rebase`, push. **👤 Toro** revisa el inicio en el TV antes de pasar a la Etapa 3.

---

## Etapa 3 — Salvapantallas de fotos

### Tarea 3.1: Pies de foto y envío de fotos desde la PC

**Files:**
- Create: `app/src/io/freemytv/PapasCaptions.java`, `tools/papas_fotos.py`
- Modify: `app/src/io/freemytv/Photos.java` (en la importación, antes de borrar el original: `PapasCaptions.imported(c, src, originalName, out)`, una línea)

**Interfaces:**
- Produces:
  - `PapasCaptions.imported(Context, File src, String originalName, File out)`: guarda en `files/papas/captions.json` el texto que Toro puso para `originalName` (de `files/papas/pending.json`) o, si no hay, `Words.captionFromDate` con la fecha EXIF de `src` (se lee **antes** de que `Photos` la borre al recodificar).
  - `PapasCaptions.caption(Context, File photo) -> String` ("" si no hay nada).
  - `python tools/papas_fotos.py <carpeta> [--serial IP:5555]`: manda las fotos con `tools/photos.py` y antes sube `pending.json` con los textos de un `pies.txt` opcional en esa carpeta (`archivo.jpg = Paseo a la playa · 2019`).

- [ ] **Step 1:** Implementar las dos piezas y el gancho.
- [ ] **Step 2:** 👤 Toro junta las fotos en una carpeta de su PC (fuera del repo) y escribe `pies.txt` para algunas. Mandarlas con `python tools/papas_fotos.py <carpeta>`.
- [ ] **Step 3:** En el TV: las fotos con texto muestran su texto; las demás, "Mes de año"; las que no tienen fecha, nada.
- [ ] **Step 4:** Commit `feat(papas): pies de foto`, `git pull --rebase`, push.

### Tarea 3.2: El salvapantallas

**Files:**
- Create: `app/src/io/freemytv/PapasDreamView.java`, `app/src/io/freemytv/PapasPhotosActivity.java`
- Modify: `app/src/io/freemytv/Screensaver.java:25` y `ScreensaverAlone.java` (`PapasMode.on(this) ? new PapasDreamView(this) : new DreamView(this)`), `app/AndroidManifest.xml` (la actividad, proceso `:home`)

**Interfaces:**
- Consumes: `Photos.files`, `Slides`, `PapasCaptions.caption`, `Words.clock/saverDate`, `HomeData` (tiempo).
- Produces: `PapasDreamView extends View`: entra con fundido de 1,6 s; fotos nítidas a pantalla completa, cada una se funde con la siguiente en 2,4 s, cambia cada 9 s, zoom lento que alterna de dirección; hora gigante, fecha, tiempo y pie de foto; arriba "Aprieta cualquier botón para volver". `void exit(Runnable done)` sale con fundido. `PapasPhotosActivity` muestra la misma vista a pedido ("pon las fotos", botón del celular) y se cierra con cualquier tecla.

- [ ] **Step 1:** Implementar las dos clases y los ganchos.
- [ ] **Step 2:** En el TV: `adb shell am start -n io.freemytv/.PapasPhotosActivity` → se ve como `cine-v4-salvapantallas.png`; cualquier tecla sale con fundido. Dejar el TV sin tocar el tiempo que tenga configurado Android → entra solo.
- [ ] **Step 3:** `python tools/papas_fps.py salvapantallas` → `OK`.
- [ ] **Step 4:** Commit `feat(papas): salvapantallas de fotos`, `git pull --rebase`, push. Fusionar `modo-papas` en `main` (PR) cuando 👤 Toro lo apruebe en el TV.

---

## Etapa 4 — Servidor del mando, emparejamiento y Abridor

### Tarea 4.1: JSON y protocolo compartidos

**Files:**
- Create: `papas/src/io/toro/papas/{Json,Protocol}.java`, `tests/papas/{JsonTest,ProtocolTest}.java`

**Interfaces:**
- Produces:
  - `Json.parse(String) -> Object` (`Map<String,Object>`, `List<Object>`, `String`, `Double`, `Boolean`, `null`); `Json.write(Object) -> String`. Lanza `IllegalArgumentException` con texto mal formado.
  - `Protocol`: puerto `PORT = 8787`; rutas `HELLO="/hola"` (GET, sin llave), `PAIR="/emparejar"`, `SAY="/decir"`, `KEY="/orden"`, `SETTINGS="/ajustes"`, `SECRET="/clave"`, `PHONES="/moviles"`, `UNPAIR="/moviles/quitar"`. Cabecera `Authorization: Bearer <llave>`. Cuerpos: `/emparejar {codigo, nombre} → {llave, rol, lan, tailscale}`; `/decir {texto} → {decir, opciones:[…]}`; `/orden {accion, veces?} → {decir}`; `/ajustes` GET → mapa, POST `{clave, valor}`; `/clave {nombre: "tmdb"|"watchmode"|"claude", valor} → {ok, error?}`; `/moviles` GET → `[{id, nombre, rol}]`, `/moviles/quitar {id}`. Errores: `{error}` con 400, 401, 403, 413 (más de 4 KB), 429.
  - Ayudantes: `Protocol.sayRequest(String texto) -> String`, `Protocol.reply(String json) -> Protocol.Reply { String say; List<String> choices; String error; }`.

- [ ] **Step 1: Tests que fallan.** `JsonTest`: ida y vuelta de `{"decir":"Listo, poniendo Amor Eterno en Netflix","opciones":["A","B"],"n":3,"ok":true,"x":null}` con tildes y comillas escapadas; `parse("{")` lanza. `ProtocolTest`: `reply("{\"decir\":\"Listo\",\"opciones\":[]}").say == "Listo"`; `reply("{\"error\":\"sin llave\"}").error == "sin llave"`; `sayRequest("pon \"la\" misa")` se puede leer de vuelta.
- [ ] **Step 2:** Fallan → implementar → `bash tools/test-papas.sh` pasa.
- [ ] **Step 3:** Commit `feat(papas): protocolo TV-Mando`, `git pull --rebase`, push.

### Tarea 4.2: Emparejamiento y reglas de red

**Files:**
- Create: `papas/src/io/toro/papas/{Pairing,NetRules}.java`, `tests/papas/{PairingTest,NetRulesTest}.java`

**Interfaces:**
- Produces:
  - `Pairing(Clock clock, java.util.Random random)` con `interface Clock { long now(); }`.
  - `String newCode(String role)`: 6 dígitos, vale 5 minutos, un solo uso; `role` = `"admin"` (celular de Toro) o `"familia"`.
  - `Pairing.Result pair(String code, String phoneName)`: `{String key; String role; String id;}` o `null`. La llave son 32 bytes en hex; se guarda solo su SHA-256. Tras 5 códigos errados en 5 minutos, bloquea 5 minutos (`locked() -> boolean`).
  - `String roleOf(String key)` (`null` si no existe); `List<String[]> phones()` (`{id, nombre, rol}`); `boolean remove(String id)`.
  - `String save()` / `static Pairing load(String saved, Clock, Random)` (persistencia en una pref).
  - `NetRules.allowed(String ip) -> boolean`: `true` para `10/8`, `172.16/12`, `192.168/16`, `100.64/10`, `127/8`; `false` para el resto (incluido IPv6 público).

- [ ] **Step 1: Tests que fallan.** `PairingTest`: código válido empareja y devuelve el rol pedido; el mismo código dos veces → `null` la segunda; a los 5 min 1 s → `null`; 5 códigos errados → `locked()` y ni el bueno sirve; a los 5 min se desbloquea; `roleOf(llave)` correcto y `roleOf("x") == null`; `remove(id)` → `roleOf` de esa llave `null`; `load(save())` conserva llaves y roles; `save()` no contiene la llave en texto. `NetRulesTest`: `192.168.1.20`, `10.0.0.5`, `172.20.1.1`, `100.101.102.103`, `127.0.0.1` → `true`; `8.8.8.8`, `172.32.0.1`, `100.128.0.1`, `2001:4860::8888` → `false`.
- [ ] **Step 2:** Fallan → implementar → pasan.
- [ ] **Step 3:** Commit `feat(papas): emparejamiento con llave por celular`, `git pull --rebase`, push.

### Tarea 4.3: El servidor en el TV

**Files:**
- Create: `app/src/io/freemytv/PapasServer.java`, `app/src/io/freemytv/PapasServerService.java`
- Modify: `app/src/io/freemytv/HomeReceiver.java` (al arrancar el TV, si `PapasMode.on`, iniciar el servicio), `app/src/io/freemytv/Secrets.java:32-33` (nombre `claude_api_key`), `app/AndroidManifest.xml` (servicio en primer plano, proceso principal), `app/src/io/freemytv/PapasSettingsActivity.java` ("Emparejar un celular")

**Interfaces:**
- Consumes: `Protocol`, `Json`, `Pairing`, `NetRules`, `KeyPage.lanAddress(c)`, `Qr.encode`.
- Produces:
  - `PapasServerService`: servicio en primer plano que vive siempre; abre `PapasServer` en `Protocol.PORT` sobre la dirección de la red de la casa y, si existe, la de Tailscale (una interfaz con dirección en `100.64/10`). Rechaza (cierra la conexión) cualquier cliente que no cumpla `NetRules.allowed`.
  - `PapasServer.Handler` (interfaz): `Protocol.Reply say(String text, String phoneId)`, `Protocol.Reply key(String action, int times)`; en esta tarea un manejador de prueba devuelve `{decir:"Recibido: <texto>"}`; la Tarea 4.4 pone el Abridor.
  - "Emparejar un celular" muestra un QR grande (`http://<lan>:8787/emparejar/<código>`) y el código de 6 dígitos, y pregunta si es el celular de Toro (rol `admin`). Esa página web del TV tiene un botón "Abrir en Mando" con el enlace `intent://emparejar?h=<lan>&t=<tailscale>&c=<código>#Intent;scheme=mando;package=io.toro.mando;end`.
  - `/clave` guarda TMDB y Watchmode donde ya los guarda `KeyPage` (con la misma revisión que hace `KeyPage` al recibir una clave) y Claude en los archivos privados de `Secrets`; nunca se escriben en el log.

- [ ] **Step 1:** Implementar las dos clases y los ganchos.
- [ ] **Step 2:** Desde la PC (en la red de la casa): `curl http://<IP>:8787/hola` → `{"tv":"Gorditos",…}`; `curl -X POST .../orden -d '{"accion":"inicio"}'` → 401; emparejar con el código de la pantalla (`curl -X POST .../emparejar -d '{"codigo":"…","nombre":"pc"}'`) → llave; repetir `/orden` con la llave → 200; con rol `familia`, `/ajustes` → 403; un cuerpo de 5 KB → 413.
- [ ] **Step 3:** Reiniciar el TV (`adb reboot`) → `/hola` responde solo y la llave sigue sirviendo.
- [ ] **Step 4:** Commit `feat(papas): servidor del mando en el TV`, `git pull --rebase`, push.

### Tarea 4.4: El Abridor y el aviso en pantalla

**Files:**
- Create: `papas/src/io/toro/papas/{Command,Keys,Titles,Replies}.java`, `tests/papas/{KeysTest,TitlesTest,RepliesTest}.java`, `app/src/io/freemytv/{PapasOpener,PapasLookup,PapasNotice}.java`
- Modify: `app/src/io/freemytv/PapasServerService.java` (usa `PapasOpener` como manejador de `/orden`)

**Interfaces:**
- Produces:
  - `Command { String action; String title; String app; String input; int minutes; int times; boolean sure; }`. Acciones: `derecha izquierda arriba abajo ok atras inicio abrir tienda configuracion accesibilidad ver youtube seguir_viendo tele entrada pausa seguir adelantar regresar siguiente_capitulo volumen_mas volumen_menos silencio apagar apagar_en fotos no_entendi`.
  - `Keys.code(String action) -> int` (`-1` si no es una tecla): derecha 22, izquierda 21, arriba 19, abajo 20, ok 23, atras 4, inicio 3, pausa 127, seguir 126, adelantar 90, regresar 89, siguiente_capitulo 87, volumen_mas 24, volumen_menos 25, silencio 164, apagar 223.
  - `Titles.Candidate { String title; String app; String link; double popularity; }`; `Titles.rank(String query, List<Candidate>) -> List<Candidate>` (mejor primero; se descartan los de parecido < 0,5, comparando textos normalizados sin tildes ni mayúsculas); `Titles.ambiguous(List<Candidate> ranked) -> boolean` (los dos primeros están a menos de 0,08 de parecido y los dos ≥ 0,7).
  - `Replies`: `playing(title, app)` → "Listo, poniendo Amor Eterno en Netflix"; `opening(app)` → "Listo, abriendo YouTube"; `done()` → "Listo"; `choose(a, b)` → "¿Amor Eterno o Amores Perros?"; `notUnderstood()` → "No te entendí, ¿qué quieres ver?"; `notInstalled(app)` → "No encuentro Claro Video en el televisor"; `searchFailed(title)` → "No pude buscar Amor Eterno ahora. Prueba en un ratito."; `sleepIn(minutes)` → "Listo, el televisor se apaga en una hora" (60 → "una hora", 30 → "media hora", otro → "N minutos").
  - `PapasOpener.run(Context, Command) -> Protocol.Reply`: teclas con `Adb.shell("input keyevent <code>")` tantas veces como `times` (volumen: 3 por orden); `abrir`/`tele`/`tienda`/`configuracion`/`accesibilidad` como las fichas; `entrada` con `PapasInputs.open`; `seguir_viendo` abre el primero de `WatchNext.load`; `fotos` abre `PapasPhotosActivity`; `apagar_en` con `SleepTimer.set(c, minutes)`; `youtube` con `Intent.ACTION_SEARCH` a la app de YouTube con `query`; `ver` con `PapasLookup`. Nunca devuelve `say` vacío.
  - `PapasLookup.find(Context, String title, String app) -> List<Titles.Candidate>`: TMDB `search/multi` (`language=es-PE`, `region=PE`) y Watchmode (fuentes en `PE`; `android_url` si hay, si no `web_url`), con la clave que ya usa `Recs`. Lanza `IOException` si falla la red.
  - `PapasNotice.show(Context, String heard, String doing)`: aviso grande de cristal por encima de cualquier app (`TYPE_APPLICATION_OVERLAY`, sin tomar el foco), entra y sale con fundido, se va solo a los 3 s. Estados: "Te escucho…", lo que entendió, lo que va a hacer.

- [ ] **Step 1: Tests que fallan.** `KeysTest`: cada acción de la lista de teclas da su código; `ver` → `-1`. `TitlesTest`: "amor eterno" entre `["Amor Eterno","Amores perros","El eterno resplandor"]` → primero "Amor Eterno" y `ambiguous == false`; "la casa" entre `["La casa de papel","La casa de las flores"]` → `ambiguous == true`; "xyz" → lista vacía. `RepliesTest`: cada frase exacta de arriba, y `failedLookupSpeaks`: `searchFailed("Amor Eterno")` no está vacío; `missingAppSpeaks`: `notInstalled("Claro Video")` es la frase exacta.
- [ ] **Step 2:** Fallan → implementar las cuatro clases puras → pasan.
- [ ] **Step 3:** Implementar `PapasOpener`, `PapasLookup`, `PapasNotice` y conectarlos.
- [ ] **Step 4:** En el TV, con la llave de la 4.3: `/orden` con `derecha`, `ok`, `atras`, `inicio`, `volumen_mas`, `pausa` (dentro de Netflix), `entrada` "HDMI 1", `fotos`, `apagar_en` 60 → cada una hace lo suyo y contesta su frase. Con el wifi del TV apagado un momento, `ver` "Amor Eterno" → contesta `searchFailed`, sin colgarse.
- [ ] **Step 5:** Commit `feat(papas): el Abridor y el aviso en pantalla`, `git pull --rebase`, push.

---

## Etapa 5 — App Mando en el celular

### Tarea 5.1: Botones gigantes y emparejamiento

**Files:**
- Create: `mando/build.sh`, `mando/AndroidManifest.xml`, `mando/res/…`, `mando/src/io/toro/mando/{MainActivity,PairActivity,MandoClient}.java`
- Modify: `tools/check.sh` (compila también `mando/`), `.gitignore` (`mando.keystore`)

**Interfaces:**
- Consumes: `Protocol`, `Json`.
- Produces:
  - `mando/build.sh` → `Mando.apk` (paquete `io.toro.mando`, minSdk 26, mismas herramientas que `build.sh`, compila `mando/src` + `papas/src`, firma con `mando.keystore`).
  - `MainActivity`: arriba el botón **🎤 Hablar** que ocupa media pantalla; debajo, en letras grandes: Canales (`tele`), Inicio, Volumen +, Volumen −, Pausa/Seguir y Fotos (el salvapantallas a pedido, spec §4). Cada botón vibra y manda `/orden`.
  - `PairActivity`: abre con el enlace `mando://emparejar?h=&t=&c=` (del QR) o pide la IP y el código de 6 dígitos a mano.
  - `MandoClient(Context)`: guarda llave, rol, IP de la casa y de Tailscale en `SharedPreferences`; `Protocol.Reply send(String path, String body)` prueba la casa (1,5 s) y si no, Tailscale.

- [ ] **Step 1:** Implementar y compilar: `bash mando/build.sh` → `Built Mando.apk`.
- [ ] **Step 2:** 👤 Toro conecta su celular por USB con la depuración activada; `adb -s <celular> install -r Mando.apk`.
- [ ] **Step 3:** En el TV, ajustes propios → "Emparejar un celular" (rol admin); 👤 Toro escanea el QR con la cámara y toca "Abrir en Mando" → queda emparejado. Cada botón hace lo suyo en el TV.
- [ ] **Step 4:** Commit `feat(mando): app con botones gigantes`, `git pull --rebase`, push.

### Tarea 5.2: Hablar desde el celular

**Files:**
- Create: `mando/src/io/toro/mando/MandoVoice.java`
- Modify: `mando/src/io/toro/mando/MainActivity.java`, `mando/AndroidManifest.xml` (`RECORD_AUDIO`, `VIBRATE`)

**Interfaces:**
- Produces: `MandoVoice.listen(Activity, Callback)` con `SpeechRecognizer` en `es-PE`; vibra y suena (`ToneGenerator`) al empezar y al terminar; manda el texto a `/decir` **sin interpretarlo**; dice la respuesta con `TextToSpeech` en español y la muestra grande. Si llegan `opciones`, las muestra como dos botones grandes y la siguiente frase o el toque se manda como respuesta.

- [ ] **Step 1:** Implementar. Hasta la Etapa 6, el TV contesta con el manejador de prueba.
- [ ] **Step 2:** 👤 Toro aprieta 🎤 y dice "hola" → vibra al empezar y al terminar; el TV muestra el aviso; el celular dice "Recibido: hola".
- [ ] **Step 3:** Commit `feat(mando): hablarle al TV desde el celular`, `git pull --rebase`, push.

### Tarea 5.3: Pantalla de Toro (solo rol admin)

**Files:**
- Create: `mando/src/io/toro/mando/AdminActivity.java`

**Interfaces:**
- Consumes: `/ajustes`, `/clave`, `/moviles`, `/moviles/quitar`.
- Produces: pantalla visible solo con rol `admin`: los mismos ajustes que `PapasSettingsActivity` (fichas, fotos, voz, botón de hablar, Claude), las claves (TMDB, Watchmode, Claude: se escriben, se mandan y se borran del celular), los celulares emparejados con "Quitar", y "Poner las fotos".

- [ ] **Step 1:** Implementar.
- [ ] **Step 2:** 👤 Toro mete sus claves de TMDB, Watchmode y Claude desde el celular → el TV responde `ok`. Ocultar una app desde el celular → desaparece del carrusel. Emparejar un segundo celular como `familia` → no ve la pantalla de Toro; quitarlo → su llave deja de servir.
- [ ] **Step 3:** Commit `feat(mando): ajustes y claves desde el celular de Toro`, `git pull --rebase`, push.

---

## Etapa 6 — El intérprete

### Tarea 6.1: Intérprete local con las frases de los Gorditos

**Files:**
- Create: `papas/src/io/toro/papas/{TextClean,Interpreter}.java`, `tests/papas/InterpreterTest.java`, `tests/papas/frases.tsv`
- Modify: `app/src/io/freemytv/PapasServerService.java` (`/decir` → `Interpreter` → `PapasOpener`; guarda las opciones pendientes de cada celular 60 s)

**Interfaces:**
- Produces:
  - `TextClean.clean(String) -> String`: minúsculas, sin tildes, sin signos; quita cortesía y relleno ("por favor", "quisiera que me pongas", "háblame", "la serie de", "la película", "oye", "este", "eh", "ya", "pues").
  - `Interpreter(List<String> appNames, List<String> inputNames)`; `Command parse(String phrase)`; `Command choose(String phrase, List<String> options)` (entiende "la primera", "la segunda", "A", "B" o el título).
  - Formato de `frases.tsv`: `frase<TAB>accion<TAB>titulo<TAB>app<TAB>entrada<TAB>minutos<TAB>nota` (columna vacía = no importa; la frase vacía se escribe `<vacío>`); las líneas que empiezan con `#` son comentarios; las de ruido llevan `#ruido` en `nota`. `InterpreterTest` lee el archivo y exige **100 %** de aciertos.
- Contenido mínimo de `frases.tsv` (60 líneas, ampliadas con las de 👤 Toro): todas las frases de ejemplo del spec §6 — "derecha", "izquierda", "arriba", "abajo", "ok", "atrás", "inicio"; "abre YouTube", "pon Netflix", "abre la tienda", "abre la configuración", "abre accesibilidad"; "quiero ver Amor Eterno", "pon la misa en YouTube" (`youtube`, "misa"), "sigue con lo que estaba viendo" (`seguir_viendo`); "pon la tele" (`tele`), "cambia a HDMI 1", "pon la laptop", "pon la antena"; "pausa", "sigue" (`seguir`), "adelanta", "regresa", "siguiente capítulo"; "sube el volumen" (`volumen_mas`, 3), "baja el volumen", "silencio", "apaga el televisor", "apágalo en una hora" (`apagar_en`, 60); "pon las fotos"; la frase larga "por favor quisiera que me pongas la serie de amor eterno en netflix" (`ver`, "amor eterno", "netflix"); variantes con "oye", "porfa", "quiero que", "ponme"; y ruido: "", "eh", "oye", "este…" → `no_entendi`.

- [ ] **Step 1:** Escribir `frases.tsv` y `InterpreterTest`. **👤 Toro** lee la lista y añade cómo hablan de verdad sus papás (mínimo 50 en total). `bash tools/test-papas.sh` → falla.
- [ ] **Step 2:** Implementar `TextClean` e `Interpreter` (reglas por palabras clave y patrones; `sure=false` cuando queda texto sin explicar o hay dos lecturas). → 100 % de aciertos.
- [ ] **Step 3:** Conectar `/decir`. Desde el celular: "quiero ver Amor Eterno" → abre el título en Netflix y dice "Listo, poniendo Amor Eterno en Netflix"; "la casa" → "¿La casa de papel o La casa de las flores?" y "la primera" lo abre.
- [ ] **Step 4:** Commit `feat(papas): intérprete local con 50+ frases`, `git pull --rebase`, push.

### Tarea 6.2: Respaldo con Claude

**Files:**
- Create: `papas/src/io/toro/papas/ClaudeFallback.java`, `tests/papas/ClaudeFallbackTest.java`, `tests/papas/claude-respuesta.json`, `app/src/io/freemytv/PapasClaude.java`
- Modify: `app/src/io/freemytv/PapasServerService.java`

**Interfaces:**
- Produces:
  - `ClaudeFallback.requestBody(String phrase, List<String> apps, List<String> inputs) -> String`: cuerpo para `POST https://api.anthropic.com/v1/messages` con `model: "claude-haiku-5-5"`, `max_tokens: 200`, un `system` en español que explica que es el control de un TV de dos personas mayores y lista las acciones de `Command`, apps y entradas; una herramienta `orden` con `input_schema` `{accion (enum de Command), titulo, app, entrada, minutos}` y `tool_choice: {"type":"tool","name":"orden"}`; `messages: [{role:"user", content: phrase}]`.
  - `ClaudeFallback.parse(String responseJson) -> Command` (del bloque `tool_use` de `content`; `no_entendi` si falta o la acción no existe).
  - `ClaudeFallback.pick(Command local, Command claudeOrNull) -> Command`: el de Claude si llegó y no es `no_entendi`; si no, el local.
  - `PapasClaude.ask(Context, String phrase) -> Command` (cabeceras `x-api-key` con `Secrets.get(c, "claude_api_key")`, `anthropic-version: 2023-06-01`, `content-type: application/json`; espera máximo 5 s). Se usa solo si la capa local da `sure=false`, el respaldo está encendido y hay clave.

- [ ] **Step 1: Tests que fallan.** `ClaudeFallbackTest`: `requestBody` contiene `"claude-haiku-5-5"`, la herramienta `orden` y la frase; `parse(claude-respuesta.json)` (una respuesta real guardada con `tool_use` `{accion:"ver", titulo:"Amor eterno", app:"Netflix"}`) da esa orden; `parse("{}")` → `no_entendi`; `parse` con acción `"bailar"` → `no_entendi`; `claudeUnavailableKeepsLocal`: con un `Command` local `sure=false` y sin clave, el elegido es el local (`pick(local, null)` devuelve `local`).
- [ ] **Step 2:** Fallan → implementar → pasan.
- [ ] **Step 3:** En el TV con la clave puesta: una frase que la capa local no entiende (👤 Toro la elige) → la resuelve Claude. Sin internet → se queda con la local y nunca calla.
- [ ] **Step 4:** Commit `feat(papas): respaldo con Claude`, `git pull --rebase`, push.

### Tarea 6.3: Hablarle al micrófono del control

Según la decisión `VOZ_A` de la Etapa 0.

**Files:**
- Create: `app/src/io/freemytv/PapasListenService.java`
- Modify: `app/src/io/freemytv/RemoteKeys.java:103` (primera línea de `onKey`: `if (PapasMode.talkKey(c, e)) return true;`), `app/src/io/freemytv/PapasMode.java`, `app/AndroidManifest.xml` (`RECORD_AUDIO`; servicio con `foregroundServiceType="microphone"`)

**Interfaces:**
- Produces:
  - `PapasMode.talkKey(Context, KeyEvent) -> boolean`: según el botón de hablar elegido (`microfono`: códigos 84, 231, 219; `ok-largo`; `ajustes`), consume la tecla (bajada y subida) y arranca `PapasListenService`; con `solo-celular`, muestra `PapasNotice` "Háblale al celular". No hace nada más (este proceso tiene que ser rápido).
  - `PapasListenService`: `PapasNotice` "Te escucho…", `SpeechRecognizer` en `es-PE`, luego lo entendido → `Interpreter` (y Claude si hace falta) → `PapasOpener` → `PapasNotice` con lo que va a hacer. Funciona dentro de Netflix, YouTube y Claro. Al activar el Modo Papás, `pm grant io.freemytv android.permission.RECORD_AUDIO` por `Adb`.

- [ ] **Step 1:** Implementar.
- [ ] **Step 2:** 👤 Toro, con el botón de hablar: en el inicio "abre YouTube"; dentro de Netflix "pausa", "sigue", "sube el volumen", "vuelve al inicio" → cada una funciona y el aviso se ve. El Asistente de Google no aparece, y los demás botones del control (Netflix, YouTube, volumen, Entradas) siguen haciendo lo suyo.
- [ ] **Step 3:** `python tools/papas_fps.py carrusel` con el aviso apareciendo → `OK`.
- [ ] **Step 4:** Commit `feat(papas): hablarle al micrófono del control`, `git pull --rebase`, push.

---

## Etapa 7 — Tailscale para ayudar desde lejos

### Tarea 7.1: Mando por Tailscale

**Files:**
- Modify: `app/src/io/freemytv/PapasServerService.java` (vuelve a buscar la dirección de Tailscale cuando cambia la red), `mando/src/io/toro/mando/MandoClient.java`

- [ ] **Step 1: 👤 Toro** instala Tailscale desde la tienda en el TV de los papás y en su celular, con la misma cuenta. Anotar la dirección `100.x` del TV en `tv-gorditos.md`.
- [ ] **Step 2:** Reabrir "Emparejar un celular" → el QR lleva también la dirección de Tailscale; volver a emparejar el celular de Toro.
- [ ] **Step 3:** 👤 Toro, con el wifi del celular apagado (solo datos y Tailscale): botones, voz y su pantalla de ajustes funcionan. Con Tailscale apagado en el celular: no conecta (el TV no está abierto a internet).
- [ ] **Step 4:** Commit `feat(papas): mando desde lejos por Tailscale`, `git pull --rebase`, push.

---

## Etapa 8 — Instalación final y prueba con los Gorditos

### Tarea 8.1: Entrega

**Files:**
- Create: `docs/superpowers/notes/2026-xx-prueba-con-los-gorditos.md` (sin datos personales)

- [ ] **Step 1:** `git fetch upstream && git merge upstream/main`; `bash tools/check.sh android` y `bash tools/test-papas.sh` → todo pasa.
- [ ] **Step 2:** Compilar e instalar `FreeMyTV.apk` en el TV y `Mando.apk` en los celulares de los papás y de Toro.
- [ ] **Step 3:** Medir todas las escenas: `carrusel`, `carrusel-rapido`, `fila`, `entradas`, `fotos-fondo`, `salvapantallas` → todas `OK`.
- [ ] **Step 4:** Desenchufar el TV y volver a enchufarlo → aparece el Modo Papás con Claro enfocada; el servidor responde; los celulares siguen emparejados; no aparece aviso de actualización.
- [ ] **Step 5: 👤 Toro con sus papás**, sin ayudarles: (1) ver la tele en Claro, (2) poner Netflix y un título por voz desde el celular, (3) igual desde el control, (4) volver al inicio, (5) subir el volumen por voz, (6) cambiar a la laptop por HDMI, (7) seguir lo que estaban viendo, (8) poner las fotos, (9) apagar en una hora. Anotar cuáles salieron solos y qué frases dijeron que no se entendieron.
- [ ] **Step 6:** Las frases que fallaron se añaden a `tests/papas/frases.tsv` y se arreglan (Tarea 6.1, Steps 2–4) antes de cerrar.
- [ ] **Step 7:** Commit `docs: prueba con los Gorditos`, `git pull --rebase`, push; fusionar en `main`.
