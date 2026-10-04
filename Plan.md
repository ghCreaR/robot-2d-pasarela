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

## 3. Contratos

La pasarela está en medio de todos los componentes. Sus interfaces están definidas en el directorio [`contratos/`](https://github.com/ojgarciab/carrera-robots-autonomos/tree/master/contratos) del repositorio común, y son la referencia para este plan:

| Contrato | Papel de la pasarela |
|----------|----------------------|
| [API de cliente](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/contratos/api-cliente.md) | La implementa: rutas REST, protocolo WebSocket, códigos de error y de cierre. |
| [Mensajes del bus](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/contratos/bus.md) | Publica `actuadores`, `actividad` y `control`; se suscribe a `sensores`, `estado`, `eventos` y `latido`; responde a `configuracion`. Concede los permisos del *auth callout*. |
| [Base de datos](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/contratos/base-de-datos.md) | Lee `usuarios`, `tokens_api`, `mundos` y `accesos`; escribe `mundos.ultimo_latido`; escucha los canales `NOTIFY`. |
| [Robots](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/robots/README.md) y [circuitos](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/circuitos/README.md) | Los carga de `ROBOTS_DIR` y `CIRCUITOS_DIR`, los sirve al visor y los entrega a los motores. |

Puntos del contrato que afectan especialmente a la implementación:

- **Buzones de respuesta.** El *auth callout* concede al motor `mundo.<uuid>.>` y `allow_responses`. El motor usa el prefijo de buzón `mundo.<uuid>.buzon`, así que la pasarela no necesita darle permisos sobre `_INBOX`.
- **Expulsión.** Al recibir `acceso_retirado` o `usuario_desactivado`, la pasarela manda `control` `expulsar` al motor del mundo afectado.
- **Etiquetas del administrador.** El nombre visible viaja en `control` `entrar`; la pasarela lo añade al `estado` que envía a los visores.
- **Valores de los sensores.** Se reenvían como números sin interpretarlos. Hoy son `0`/`1` (sensor digital), pero el sensor promediado previsto dará valores entre `0` y `1` y no debe requerir cambios en la pasarela.
- **Cookie.** `POST /sesion` cambia un token `Bearer` por la cookie `crt_token`, para clientes del mismo sitio.

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
- Pruebas de que los mensajes cumplen `contratos/bus.md` (mismos nombres de campo y tipos).

### Fase 2 · Autenticación y accesos
- `auth/tokens.py`: separa el prefijo `crt_rw_`/`crt_ro_`, calcula el hash SHA-256 y lo busca en la base de datos. Comprueba la caducidad, que no esté revocado y que el usuario esté activo. Guarda el resultado en una caché con TTL corto (por ejemplo 10 s).
- `bd.py`: escucha `LISTEN` en los canales de `contratos/base-de-datos.md` e invalida la caché al instante. Al reconectar con la base de datos, vacía la caché entera, porque puede haberse perdido algún aviso.
- `auth/accesos.py`: acceso usuario-mundo, con la misma caché.
- Extracción del token desde la cabecera `Bearer`, la cookie (`HttpOnly`, `Secure`, `SameSite=Strict`) o el primer mensaje del WebSocket. **Nunca** desde la URL.
- `GET /yo` y `GET /mundos` (de momento sin el estado activo).

**Pruebas:** token válido, caducado, revocado (incluido mientras está en caché), de tipo equivocado y ausente; usuario sin acceso al mundo.

### Fase 3 · CORS y recursos estáticos
- Middleware CORS propio con exactamente las reglas del README común: origen concreto y nunca `*`, `Vary: Origin`, *preflight*, `Max-Age` y sin `Allow-Credentials`.
- Comprobación de `Origin` en el WebSocket: se rechaza si viene y no está en la lista, y se acepta si no viene (clientes que no son navegadores).
- `robots.py`: carga y valida los YAML de `ROBOTS_DIR` y de `CIRCUITOS_DIR`. Sirve `GET /robots/<id>`, `GET /robots/svg/<fichero>` y `GET /circuitos/<id>`, protegido contra recorrer rutas fuera del directorio.
- `POST /sesion` y `DELETE /sesion` para la cookie.

### Fase 4 · *Auth callout* de NATS
- `auth/callout.py`: se suscribe a `$SYS.REQ.USER.AUTH`, decodifica la petición (JWT firmado por el servidor) y lee el UUID y el testigo de las credenciales que presenta el motor.
- Comprueba el hash del testigo en `mundos` y responde con un JWT de usuario firmado con `NATS_CALLOUT_SEED`. Los permisos son `pub`/`sub` sobre `mundo.<uuid>.>` más `allow_responses`. Si no es válido, responde con un error.
- Guarda, de cada motor aceptado, el identificador del servidor NATS y el `cid` de su conexión, que llegan en la petición del *auth callout*.
- Al recibir `testigo_revocado`, **corta la conexión del motor al instante** con `$SYS.REQ.SERVER.<id_servidor>.KICK` y deja de aceptar el testigo antiguo.
- Para poder hacerlo necesita un segundo usuario de NATS en la **cuenta de sistema**, con permiso solo para publicar en `$SYS.REQ.SERVER.*.KICK`. Hay que añadirlo a `nats/nats.conf`, `compose.yaml` y `.env.example` del repositorio común (`NATS_SISTEMA_PASSWORD`). Como cambia la organización de cuentas de NATS (la cuenta de sistema obliga a declarar las cuentas de forma explícita), el cambio de `nats.conf` se prueba en esta fase con un `nats-server` real antes de publicarlo.
- Como defensa añadida, el JWT que firma la pasarela para cada motor caduca a las 24 h, así que el motor vuelve a pasar por el *auth callout* al reconectar.

**Pruebas:** integración con un `nats-server` real en CI (contenedor de servicio), con un motor falso que se conecta con un testigo bueno y con uno malo, y que intenta publicar fuera de su mundo.

### Fase 5 · Mundos activos y configuración
- `mundos.py`: se suscribe a `mundo.*.latido`, mantiene el registro en memoria y da por parado un mundo sin latidos en 3 × `LATIDO_S`.
- Escribe `ultimo_latido` en la base de datos como mucho cada 5 s por mundo.
- Responde a `mundo.*.configuracion` con los robots permitidos (filtrados de `ROBOTS_DIR`) y la definición del circuito pedido (de `CIRCUITOS_DIR`).
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
- `seguir` con `usuario` y `?usuario=` en `GET /sensores` y `GET /robot` (solo administradores), para ver los sensores de cualquier robot del mundo.
- Pruebas: un usuario normal que manda `usuario` recibe `no_admin`.

### Fase 9 · Robustez y observabilidad
- Reconexión a NATS y a PostgreSQL sin perder las conexiones de los clientes.
- Registros estructurados (JSON), sin escribir nunca tokens ni testigos.
- Métricas básicas: conexiones abiertas, latencia del bus y mensajes descartados.
- Límite de conexiones y de tamaño de mensaje por cliente.
- Pruebas de carga sencillas: 4 robots × 10 Hz por WebSocket y por polling, con varios visores.

## 5. Estrategia de pruebas

- **Unitarias:** tokens, caché, CORS, validación de actuadores y serialización.
- **Con transporte en memoria:** un motor simulado responde a `control` y publica sensores, para probar la API completa sin NATS.
- **De integración (CI):** `nats-server` y `postgres` como servicios de GitHub Actions, con el esquema de `contratos/base-de-datos.md` creado por las pruebas hasta que la interfaz tenga sus migraciones.
- **De extremo a extremo:** con el `compose.yaml` del repositorio común, cuando el motor esté disponible.

## 6. Dependencias con otros repositorios

| Depende de | Qué necesita |
|------------|--------------|
| Repositorio común | Contratos, definiciones de robots y de circuitos. `compose.yaml` ya monta `robots/` y `circuitos/` en la pasarela. |
| `robot-2d-interfaz-web` | Esquema de la base de datos con nombres de tablas estables y los avisos `NOTIFY`. |
| `robot-2d-motor-fisicas` | Que publique y responda con los mensajes de `contratos/bus.md`. |

## 7. Decisiones tomadas

- **Sensores IR:** digitales (`0`/`1`) para empezar. Más adelante habrá un sensor `infrarrojo_promedio` con valores de `0` a `1`.
- **Circuitos:** se definen en el repositorio común (`circuitos/*.yaml`), como los robots.
- **Contratos:** viven en el repositorio común (`contratos/`).
- **Buzón del motor** `mundo.<uuid>.buzon`, operación `expulsar` y cookie con `POST /sesion`: incluidos en los contratos.
- **Testigo revocado:** se corta al instante la conexión activa del motor (`KICK`).
- **Administradores:** pueden seguir los sensores de cualquier robot.

## 8. Preguntas abiertas

Ninguna por ahora.
