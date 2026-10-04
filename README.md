# robot-2d-pasarela

Pasarela de API en tiempo real de la [carrera de robots autónomos](https://github.com/ojgarciab/carrera-robots-autonomos).

> **Estado:** en diseño. Todavía no hay código; el plan de implementación está en [`Plan.md`](Plan.md).

## Qué es

La pasarela es la **única puerta de entrada al servidor** para los clientes de control y para el visor web. Recibe sus conexiones por **REST** y **WebSocket**, comprueba sus **tokens de API** y reenvía los datos por el **bus de mensajes (NATS)** a los motores de simulación, que son los que calculan la física.

No simula nada. Se encarga de todo lo que depende de la red y de los clientes, para que el bucle de física de los motores no se vea afectado por clientes lentos o por picos de conexiones. Por eso se pueden levantar **varias réplicas** detrás de un balanceador.

```
  clientes de control ─┐                        ┌─► base de datos (PostgreSQL)
  (REST / WebSocket)   │   ┌────────────────┐   │   tokens, accesos, mundos
                       ├──►│    PASARELA    │◄──┤
  visor web ───────────┘   │  (este repo)   │   └── LISTEN/NOTIFY: revocaciones
  (WebSocket / REST)       └───────┬────────┘
                                   │ NATS: mundo.<uuid>.…
                                   ▼
                     motores de simulación (uno por mundo)
```

## Responsabilidades

| Área | Qué hace |
|------|----------|
| **API de cliente** | Endpoints REST (`GET /mundos`, entrada y salida del mundo, lectura de sensores, envío de actuadores, `ping`) y un endpoint WebSocket con los mismos comandos. |
| **Autenticación** | Valida los tokens de API (`crt_rw_…` de lectura-escritura y `crt_ro_…` de solo lectura). Los acepta por cabecera `Authorization: Bearer`, en el primer mensaje del WebSocket o en una cookie. Guarda en memoria durante unos segundos los tokens ya validados y los olvida al instante cuando la base de datos avisa de una revocación (`NOTIFY`). |
| **Control de acceso** | Comprueba que el usuario tiene acceso al mundo y que su token permite la operación: un token de solo lectura no mueve el robot ni alarga su vida. La vista de administrador solo la pueden usar los administradores. |
| **Datos en tiempo real** | Reenvía al motor las consignas de los actuadores y empuja la telemetría de los sensores a los WebSocket. Guarda el último valor de cada sensor para el polling, incluido el *long polling* (esperar al siguiente muestreo). |
| **Actividad del cliente** | Avisa al motor de que el cliente sigue vivo, agrupando los avisos (como mucho uno por segundo). Con ellos el motor lleva los temporizadores de parada de motores (5 s) y de salida del mundo (5 min). |
| **Mundos activos** | Escucha los latidos de los motores, ofrece el listado de mundos activos y guarda en la base de datos la hora del último latido. |
| **Autenticación de motores** | Responde al *auth callout* de NATS: comprueba el UUID y el testigo de cada motor contra la base de datos y le da permisos solo sobre los temas de su mundo (`mundo.<uuid>.>`). |
| **Configuración de mundos** | Entrega a cada motor, al arrancar, las definiciones de sus robots permitidos y de su circuito. |
| **Recursos estáticos** | Sirve las capas SVG de los robots al visor web. |
| **CORS** | Aplica la lista de orígenes permitidos (`CORS_ORIGENES`) a las peticiones REST y comprueba la cabecera `Origin` al abrir WebSockets desde un navegador. |

## Qué no hace

- **No simula.** La física, los sensores, los temporizadores del ciclo de vida y el límite de robots son cosa del motor de simulación.
- **No es dueña del esquema de la base de datos.** Lo crea y lo migra la [interfaz de gestión](https://github.com/ghCreaR/robot-2d-interfaz-web). La pasarela solo lee, salvo la hora del último latido de cada mundo, que sí escribe.
- **No gestiona usuarios ni tokens.** Se generan y revocan en la interfaz de gestión.
- **No guarda la telemetría.** Los sensores y los actuadores nunca pasan por la base de datos.

## Configuración

Se configura con variables de entorno (ver el [`compose.yaml`](https://github.com/ojgarciab/carrera-robots-autonomos/blob/main/compose.yaml) del repositorio común):

| Variable | Descripción |
|----------|-------------|
| `NATS_URL` | Dirección del bus, por ejemplo `nats://bus:4222`. |
| `NATS_USUARIO`, `NATS_PASSWORD` | Credenciales de la pasarela en NATS. |
| `NATS_CALLOUT_SEED` | Clave privada (*nkey* de cuenta) con la que firma las respuestas del *auth callout*. |
| `DATABASE_URL` | Conexión a PostgreSQL. |
| `LONG_POLLING_TIMEOUT_MS` | Espera máxima de una petición de long polling (1000 ms por defecto). |
| `CORS_ORIGENES` | Orígenes permitidos, separados por comas. |
| `ROBOTS_DIR` | Directorio con las definiciones YAML y las capas SVG de los robots. |

Escucha en el puerto `8080` del contenedor.

## Documentación relacionada

- [Arquitectura del servidor](https://github.com/ojgarciab/carrera-robots-autonomos#arquitectura-del-servidor)
- [API de cliente](https://github.com/ojgarciab/carrera-robots-autonomos#api-de-cliente) y [tokens de API](https://github.com/ojgarciab/carrera-robots-autonomos#tokens-de-api)
- [Mensajes entre componentes](https://github.com/ojgarciab/carrera-robots-autonomos#mensajes-entre-componentes)
- [Plan de implementación](Plan.md)

## Licencia

[GPL-3.0](LICENSE).
