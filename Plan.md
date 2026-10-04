# Plan de implementación · robot-2d-pasarela

Este plan detalla cómo construir la pasarela de API descrita en el [README del repositorio común](https://github.com/ojgarciab/carrera-robots-autonomos). Todavía no hay código: es una propuesta para revisar antes de empezar.

## 1. Decisiones técnicas propuestas

| Tema | Propuesta | Motivo |
|------|-----------|--------|
| Lenguaje | **Python 3.12** con `asyncio` | Coherente con el `.gitignore` del repositorio y con el resto de componentes. El trabajo de la pasarela es de entrada/salida, no de cálculo, así que `asyncio` basta. |
| Servidor HTTP y WebSocket | **Starlette** + **Uvicorn** (`uvloop`, `httptools`) | Ligero, con REST y WebSocket en el mismo servidor y buen soporte de `asyncio`. FastAPI es una alternativa si se quiere OpenAPI automático. |
| Bus | **nats-py** | Cliente oficial de NATS para `asyncio`, con petición-respuesta. |
| Base de datos | **asyncpg** | Rápido y con soporte de `LISTEN`/`NOTIFY`. Solo consultas SQL directas: el esquema es de la interfaz de gestión. |
| Formato interno | **MessagePack** (`msgpack`) | Binario y compacto, como recomienda el README común. Hacia los clientes, JSON. |
| *Auth callout* | **nkeys** + JWT de NATS construidos a mano | La respuesta al *auth callout* es un JWT de usuario firmado con la *nkey* de cuenta. |
| YAML | **PyYAML** (`safe_load`) | Para leer las definiciones de los robots. |
| Pruebas | **pytest** + **pytest-asyncio**, **httpx** | Más un transporte en memoria que sustituye a NATS en las pruebas. |
| Calidad | **ruff** (lint y formato) y **mypy** | |
| Empaquetado | `pyproject.toml` y **uv** o **pip** | |

## 2. Estructura del repositorio

```
robot-2d-pasarela/
├── pyproject.toml
├── Dockerfile
├── src/pasarela/
│   ├── __main__.py          # arranque: configuración, conexiones y servidor
│   ├── config.py            # lectura y validación de las variables de entorno
│   ├── app.py               # aplicación Starlette: rutas, middleware CORS
│   ├── api/
│   │   ├── rest.py          # endpoints REST
│   │   └── websocket.py     # endpoint WebSocket y su protocolo
│   ├── auth/
│   │   ├── tokens.py        # validación de tokens de API y caché en memoria
│   │   ├── accesos.py       # acceso de cada usuario a cada mundo
│   │   └── callout.py       # respuestas al auth callout de NATS
│   ├── bus/
│   │   ├── transporte.py    # interfaz abstracta del bus
│   │   ├── nats.py          # implementación con NATS
│   │   ├── memoria.py       # implementación en memoria para pruebas
│   │   └── mensajes.py      # tipos de mensaje y (de)serialización MessagePack
│   ├── mundos.py            # registro de mundos activos (latidos)
│   ├── sesiones.py          # robots, suscripciones de visores y caché de sensores
│   ├── actividad.py         # agrupación de avisos de actividad (≤ 1/s)
│   ├── robots.py            # carga de robots/ (YAML y SVG)
│   └── bd.py                # consultas SQL y escucha de NOTIFY
└── tests/
```

La clave del diseño es `bus/transporte.py`: la lógica solo conoce mensajes, no NATS. Así las pruebas usan un transporte en memoria y cambiar de bus más adelante no obliga a reescribir nada, como pide el README común.

## 3. Contratos que hay que cerrar antes de programar

La pasarela está en medio de todos los componentes, así que los contratos se definen aquí. Se propone **publicarlos en el repositorio común** (por ejemplo en `contratos/`) en cuanto se aprueben, para que todos los repositorios usen la misma referencia.

### 3.1. API REST (JSON)

Todas las rutas, salvo `/salud` y los SVG, llevan `Authorization: Bearer <token>`. Los errores devuelven `{"error": "<codigo>", "mensaje": "…"}`.

| Método y ruta | Token | Descripción |
|---------------|-------|-------------|
| `GET /salud` | — | Comprobación de vida, para Docker y el balanceador. |
| `GET /ping` | `ro`/`rw` | `{"ts": <ms desde epoch, con decimales>}`. Con `rw`, alarga la vida del robot si se indica `?mundo=<uuid>`. |
| `GET /yo` | `ro`/`rw` | Usuario, rol y tipo del token. Lo usa el visor para saber si puede abrir la vista de administrador. |
| `GET /mundos` | `ro`/`rw` | Mundos a los que el usuario tiene acceso: `uuid`, `nombre`, `activo`, `robots_permitidos`, `robots`, `max_robots` y `circuito`. |
| `GET /robots/<id>` | `ro`/`rw` | Definición del modelo en JSON: sensores, actuadores, capas. |
| `GET /robots/svg/<fichero>` | — | Capa SVG, con cabeceras CORS. |
| `GET /circuitos/<id>` | `ro`/`rw` | Geometría del circuito, para la vista de administrador. |
| `POST /mundos/<uuid>/entrar` | `rw` | `{"modelo": "sigue-lineas-3ir"}`. Respuestas: `200` (dentro, recuperado o nuevo), `409 mundo_lleno`, `403 sin_acceso`, `422 modelo_no_permitido`, `503 mundo_inactivo`. |
| `POST /mundos/<uuid>/salir` | `rw` | Salida voluntaria. Es idempotente. |
| `GET /mundos/<uuid>/robot` | `ro`/`rw` | Si el robot del usuario está en el mundo, y con qué modelo. |
| `GET /mundos/<uuid>/sensores` | `ro`/`rw` | Últimos valores: `{"ts": …, "seq": …, "valores": {"ir_central": 1, …}}`. Con `?esperar=1` hace *long polling* hasta el siguiente muestreo, o como mucho `LONG_POLLING_TIMEOUT_MS`. Con `?desde=<seq>` responde enseguida si ya hay una muestra posterior. Devuelve `404 robot_fuera` si el robot no está en el mundo. |
| `POST /mundos/<uuid>/actuadores` | `rw` | `{"valores": {"motor_izquierdo": 0.5, "motor_derecho": 0.4}}`. Valida los `id` y el rango `[-1, 1]`. |

### 3.2. Protocolo WebSocket (JSON)

Endpoint: `GET /ws`. Cada mensaje es un objeto con un campo `tipo`; si el cliente pone `id`, la respuesta lo repite.

| Cliente → pasarela | Descripción |
|--------------------|-------------|
| `{"tipo": "autenticar", "token": "…"}` | Debe ser el primer mensaje si no se usó cabecera ni cookie. Si no llega en 5 s, se cierra la conexión con el código `4401`. |
| `{"tipo": "entrar", "mundo": "<uuid>", "modelo": "…"}` | Solo con `rw`. Entra al mundo y se suscribe a los sensores de su robot. |
| `{"tipo": "seguir", "mundo": "<uuid>"}` | Se suscribe a los sensores del robot del usuario sin entrar. Es lo que usan el visor y los terceros con `ro`. |
| `{"tipo": "observar", "mundo": "<uuid>"}` | Vista de administrador: estado de todos los robots. Solo para administradores. |
| `{"tipo": "actuadores", "mundo": "<uuid>", "valores": {…}}` | Solo con `rw`. |
| `{"tipo": "salir", "mundo": "<uuid>"}` | Solo con `rw`. |
| `{"tipo": "ping"}` | Respuesta `{"tipo": "pong", "ts": …}`. |

| Pasarela → cliente | Descripción |
|--------------------|-------------|
| `{"tipo": "sensores", "mundo", "ts", "seq", "valores"}` | Cada muestra, en cuanto llega del motor. |
| `{"tipo": "robot", "mundo", "en_mundo": true/false}` | Cambio de presencia del robot. Permite mostrar "El robot no está actualmente en el mundo" y reanudar sin reconectar. |
| `{"tipo": "estado", "mundo", "ts", "robots": [{"usuario", "nombre", "modelo", "x", "y", "theta"}]}` | Vista de administrador, a 30 Hz. |
| `{"tipo": "error", "error": "<codigo>", "mensaje"}` | Errores de comando. |

Una conexión puede seguir varios mundos a la vez, porque un token vale para todos los mundos del usuario.

### 3.3. Mensajes del bus (MessagePack)

El `<id>` de robot en los temas es el **UUID del usuario**, porque cada robot se identifica por el par (mundo, usuario). Todas las marcas `ts` están en **milisegundos desde epoch, con decimales**, según el reloj del motor.

| Tema | Tipo | Contenido |
|------|------|-----------|
| `mundo.<uuid>.robot.<usuario>.sensores` | publicación | `{ts, seq, valores}` |
| `mundo.<uuid>.robot.<usuario>.actuadores` | publicación | `{ts, valores}` |
| `mundo.<uuid>.robot.<usuario>.actividad` | publicación | `{ts, motivo: "polling" \| "ws_abierto" \| "ws_cerrado"}` |
| `mundo.<uuid>.control` | petición-respuesta | `{id_peticion, op: "entrar" \| "salir" \| "listar" \| "expulsar", usuario, nombre, modelo}` → `{id_peticion, resultado: "dentro" \| "recuperado" \| "lleno" \| "fuera" \| "error", robots?}` |
| `mundo.<uuid>.estado` | publicación | `{ts, robots: [{usuario, modelo, x, y, theta}]}` |
| `mundo.<uuid>.eventos` | publicación | `{ts, evento: "entra" \| "sale" \| "motores_off", usuario, motivo}` |
| `mundo.<uuid>.latido` | publicación | `{ts, circuito, robots: [usuario…], max_robots}` |
| `mundo.<uuid>.configuracion` | petición-respuesta | `{circuito}` → `{nombre, robots: {<id>: definición}, circuito: geometría}` |

Detalles que el README común no cubre y hay que tener en cuenta:

- **Buzones de respuesta.** Con permisos limitados a `mundo.<uuid>.>`, el motor no podría recibir las respuestas en `_INBOX.>`. Se propone que el motor use el prefijo de buzón `mundo.<uuid>.buzon` y que el *auth callout* conceda además `allow_responses`, para que pueda contestar a las peticiones de `control` que le hace la pasarela.
- **`expulsar`.** Es una operación nueva de `control`: cuando la interfaz retira el acceso de un usuario a un mundo, la pasarela recibe el `NOTIFY` y pide al motor que saque el robot.
- **Etiquetas del administrador.** El nombre visible del usuario viaja en `entrar`. El motor lo guarda y la pasarela lo añade al `estado`, para que el motor no necesite la base de datos.

### 3.4. Base de datos

El esquema es de la interfaz de gestión (ver su `Plan.md`). La pasarela necesita:

- **Lectura:** `usuarios` (id, nombre visible, rol, activo), `tokens_api` (hash, tipo, caducidad, revocado), `mundos` (uuid, nombre, robots permitidos, hash del testigo) y `accesos` (usuario, mundo).
- **Escritura:** `mundos.ultimo_latido`, como mucho cada 5 s por mundo.
- **Canales de `NOTIFY`:** `token_revocado` (hash), `acceso_retirado` (usuario, mundo), `usuario_desactivado` (usuario) y `testigo_revocado` (mundo).

Hay que acordar los nombres exactos de las tablas con la interfaz y fijarlos en el contrato.

## 4. Fases de implementación

Cada fase termina con pruebas automáticas en verde y se puede revisar por separado.

### Fase 0 · Esqueleto del proyecto
- `pyproject.toml`, estructura de paquetes, `ruff`, `mypy` y `pytest`.
- `config.py`: lee y valida todas las variables de entorno y falla con un mensaje claro si falta una obligatoria.
- `GET /salud` y arranque con Uvicorn.
- `Dockerfile` multietapa (`python:3.12-slim`), usuario sin privilegios y `HEALTHCHECK`.
- GitHub Actions: lint, tipos, pruebas y construcción de la imagen.

**Criterio de aceptación:** `docker compose up pasarela bus bd` (desde el repositorio común) arranca y `/salud` responde 200.

### Fase 1 · Contratos y transporte
- `bus/mensajes.py`: tipos de mensaje (dataclasses) y su serialización MessagePack, con pruebas de ida y vuelta.
- `bus/transporte.py` (interfaz), `bus/memoria.py` y `bus/nats.py`.
- Publicar los contratos de la sección 3 en el repositorio común.

### Fase 2 · Autenticación y accesos
- `auth/tokens.py`: separa el prefijo `crt_rw_`/`crt_ro_`, calcula el hash SHA-256 y lo busca en la base de datos. Comprueba la caducidad, que no esté revocado y que el usuario esté activo. Guarda el resultado en una caché con TTL corto (por ejemplo 10 s).
- `bd.py`: escucha `LISTEN` en los canales de la sección 3.4 e invalida la caché al instante. Al reconectar con la base de datos, vacía la caché entera, porque puede haberse perdido algún aviso.
- `auth/accesos.py`: acceso usuario-mundo, con la misma caché.
- Extracción del token desde la cabecera `Bearer`, la cookie (`HttpOnly`, `Secure`, `SameSite=Strict`) o el primer mensaje del WebSocket. **Nunca** desde la URL.
- `GET /yo` y `GET /mundos` (de momento sin el estado activo).

**Pruebas:** token válido, caducado, revocado (incluido mientras está en caché), de tipo equivocado y ausente; usuario sin acceso al mundo.

### Fase 3 · CORS y recursos estáticos
- Middleware CORS propio con exactamente las reglas del README común: origen concreto y nunca `*`, `Vary: Origin`, *preflight*, `Max-Age` y sin `Allow-Credentials`.
- Comprobación de `Origin` en el WebSocket: se rechaza si viene y no está en la lista, y se acepta si no viene (clientes que no son navegadores).
- `robots.py`: carga y valida los YAML de `ROBOTS_DIR`. Sirve `GET /robots/<id>` y `GET /robots/svg/<fichero>`, protegido contra recorrer rutas fuera del directorio.

### Fase 4 · *Auth callout* de NATS
- `auth/callout.py`: se suscribe a `$SYS.REQ.USER.AUTH`, decodifica la petición (JWT firmado por el servidor) y lee el UUID y el testigo de las credenciales que presenta el motor.
- Comprueba el hash del testigo en `mundos` y responde con un JWT de usuario firmado con `NATS_CALLOUT_SEED`. Los permisos son `pub`/`sub` sobre `mundo.<uuid>.>` más `allow_responses`. Si no es válido, responde con un error.
- Al recibir `testigo_revocado`, desconecta al motor (por ejemplo con la API de sistema de NATS o con un tema de control) y deja de aceptarlo.

**Pruebas:** integración con un `nats-server` real en CI (contenedor de servicio), con un motor falso que se conecta con un testigo bueno y con uno malo, y que intenta publicar fuera de su mundo.

### Fase 5 · Mundos activos y configuración
- `mundos.py`: se suscribe a `mundo.*.latido`, mantiene el registro en memoria y da por parado un mundo sin latidos en 3 × `LATIDO_S`.
- Escribe `ultimo_latido` en la base de datos como mucho cada 5 s por mundo.
- Responde a `mundo.*.configuracion` con los robots permitidos (filtrados de `ROBOTS_DIR`) y la geometría del circuito.
- `GET /mundos` con `activo`, robots presentes y máximo.

### Fase 6 · Ciclo de vida y actuadores
- `POST /entrar` y `POST /salir`, y sus comandos WebSocket, como peticiones `control` con `id_peticion` y reintento idempotente si no llega respuesta.
- `POST /actuadores` y el comando WebSocket, validados contra la definición del modelo.
- `actividad.py`: emite `actividad` solo para tokens `rw`, agrupada (como mucho 1/s por robot), y siempre al abrir o cerrar un WebSocket.
- Sincronización al arrancar: `control` `listar` a cada mundo activo, para saber qué robots hay y de quién son.
- `expulsar` al recibir `acceso_retirado` o `usuario_desactivado`.

### Fase 7 · Telemetría y polling
- `sesiones.py`: se suscribe a los sensores de los robots que tienen algún cliente y guarda el último valor de cada robot.
- Envío por WebSocket a cada suscriptor, con una cola de longitud 1 por conexión: si un cliente va lento, se descartan las muestras viejas en lugar de acumularlas.
- `GET /sensores` inmediato, con `?esperar=1` (un `asyncio.Event` por robot) y con `?desde=<seq>`.
- Mensajes `robot` (`en_mundo`) a partir de `eventos`, para los visores con token `ro`.
- `GET /ping` y el comando `ping`.

### Fase 8 · Vista de administrador
- Comando `observar` (solo administradores): reenvía `estado` a 30 Hz, enriquecido con el nombre visible de cada usuario.
- `GET /circuitos/<id>`.

### Fase 9 · Robustez y observabilidad
- Reconexión a NATS y a PostgreSQL sin perder las conexiones de los clientes.
- Registros estructurados (JSON), sin escribir nunca tokens ni testigos.
- Métricas básicas: conexiones abiertas, latencia del bus y mensajes descartados.
- Límite de conexiones y de tamaño de mensaje por cliente.
- Pruebas de carga sencillas: 4 robots × 10 Hz por WebSocket y por polling, con varios visores.

## 5. Estrategia de pruebas

- **Unitarias:** tokens, caché, CORS, validación de actuadores y serialización.
- **Con transporte en memoria:** un motor simulado responde a `control` y publica sensores, para probar la API completa sin NATS.
- **De integración (CI):** `nats-server` y `postgres` como servicios de GitHub Actions, con un esquema mínimo creado por las pruebas hasta que la interfaz publique el suyo.
- **De extremo a extremo:** con el `compose.yaml` del repositorio común, cuando el motor esté disponible.

## 6. Dependencias con otros repositorios

| Depende de | Qué necesita |
|------------|--------------|
| Repositorio común | Contratos (sección 3), definiciones de robots y, si se aprueba, de circuitos. En `compose.yaml` habría que montar también `circuitos/` en la pasarela. |
| `robot-2d-interfaz-web` | Esquema de la base de datos con nombres de tablas estables y los avisos `NOTIFY`. |
| `robot-2d-motor-fisicas` | Que publique y responda con los mensajes de la sección 3.3. |

## 7. Preguntas abiertas

1. **Valor de los sensores IR:** ¿`0`/`1` (detecta o no la línea) o un valor analógico `0..1` (fracción del punto de medida sobre la línea)? Afecta al motor, al visor y al cliente de referencia.
2. **Circuitos:** ¿se definen en el repositorio común (`circuitos/*.yaml`) como los robots? Es lo que se propone aquí.
3. **Cookie:** ¿quién la emite? La pasarela no tiene inicio de sesión. Una opción es un `POST /sesion` que cambia un token por una cookie del mismo sitio.
4. **Expulsión de motores:** ¿basta con dejar de aceptar el testigo en la siguiente conexión, o hay que cortar la conexión activa al revocarlo?
