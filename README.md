# Nodo BFA (producción)

Nodo transaccional de la Blockchain Federal Argentina (BFA), basado en la
imagen oficial [`bfaar/nodo`](https://hub.docker.com/r/bfaar/nodo) (geth),
conectado a la red de producción (`network id 47525974938`).

Referencia oficial: https://gitlab.bfa.ar/docker/bfanodo

## Requisitos

- Docker y Docker Compose instalados.
- Al menos 6 GB de RAM libres para el contenedor (`mem_limit: 6g`).
- Espacio en disco suficiente para la blockchain (crece con el tiempo).

## Configuración (`.env`)

Toda la configuración del `docker-compose.yml` (imagen/tag, memoria, puertos,
a qué interfaz se publica el RPC, nombre del contenedor, parámetros del
healthcheck) vive en `.env`, en la misma carpeta. Docker Compose lo carga
automáticamente — para cambiar algo, editá `.env` y volvé a correr
`docker compose up -d`.

Variable | Para qué sirve
--- | ---
`BFANODO_IMAGE` / `BFANODO_TAG` | Imagen y tag (`latest` = producción, `test` = testnet)
`BFANODO_PLATFORM` | Plataforma forzada (`linux/amd64`, requerido por la imagen)
`BFANODO_CONTAINER_NAME` | Nombre del contenedor
`BFANODO_RESTART_POLICY` | Política de reinicio
`BFANODO_MEM_LIMIT` | Límite de memoria del contenedor
`BFANODO_RPC_BIND` | Interfaz donde se publican HTTP/WS (`127.0.0.1` = solo local)
`BFANODO_HTTP_PORT` / `BFANODO_WS_PORT` | Puertos host para JSON-RPC HTTP y WebSocket
`BFANODO_P2P_PORT` | Puerto P2P (TCP+UDP), abierto al mundo
`BFANODO_HEALTHCHECK_*` | Intervalo, timeout, start_period y reintentos del healthcheck

## Levantar el nodo

```bash
docker compose up -d
```

Esto descarga la imagen `bfaar/nodo:latest`, crea un volumen persistente
(`bfanodo_data`) y arranca el nodo con reinicio automático
(`restart: unless-stopped`).

## Verificar estado

```bash
docker compose ps
docker compose logs -f bfanodo
docker stats bfanodo
docker exec bfanodo localstate.pl
```

Para chequear la sincronización vía JSON-RPC (con el RPC expuesto en
127.0.0.1:8545):

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://127.0.0.1:8545
```

`false` como resultado significa que el nodo ya está sincronizado.

## Cuentas

El nodo se levanta **sin cuentas** (es solo la puerta de entrada a la red).
Las cuentas deben crearse y usarse desde las aplicaciones que se conecten al
nodo, no en el nodo mismo. Para crear una cuenta y exportar su keystore:

```bash
docker run --rm -v "$(pwd)":/casa -u "$(id -u)" bfaar/nodo \
  geth --keystore /casa account new --password /dev/null
```

Hacé backup del archivo de la keystore generado antes de usarlo.

## Exponer el RPC a otras máquinas

Por defecto `.env` publica los puertos 8545 (HTTP) y 8546 (WebSocket) solo en
`127.0.0.1` del host, por seguridad. Si necesitás que otras máquinas de tu
red o internet accedan al RPC, cambiá `BFANODO_RPC_BIND` en `.env` a `0.0.0.0`
(o a una IP específica) y luego:

```bash
docker compose up -d
```

## Red de test

Esta configuración apunta a producción. Para levantar en cambio un nodo
contra la red de pruebas (`test2network`, network id 55555000000), cambiá
`BFANODO_TAG=test` en `.env`.

## Detener / eliminar

```bash
docker compose down        # detiene y elimina el contenedor, conserva el volumen
docker compose down -v     # además borra el volumen (pierde la blockchain sincronizada)
```

## Nota técnica: por qué no usamos un `ethereum/client-go` moderno

Existe un setup alternativo de AFIP ([`geth-setup`](https://gitlab.bfa.ar/blockchain/nucleo))
que arma el nodo con la imagen oficial `ethereum/client-go` (geth genérico) en
vez de la imagen `bfaar/nodo`, usando `genesis.json` + `config.toml`
explícitos. Es un approach más transparente y esa imagen es multi-arch (corre
nativo en arm64, sin emulación), así que lo probamos como reemplazo.

**No lo usamos porque no logramos que sincronice con la red real.** Verificado
en vivo (18/09/2026):

- Los nodos BFA de producción que siguen respondiendo corren geth **v1.9.22**
  (2020), y aceptan handshake RLPx únicamente de clientes de esa misma
  generación.
- Probamos `ethereum/client-go:v1.13.15` y `v1.10.26` contra los mismos dos
  peers en vivo (`170.210.45.179:30303` y `200.16.28.124:30303`, enodes
  actuales — los que trae el `config.toml` de `geth-setup` para esas IPs están
  desactualizados, la clave pública ya no corresponde al proceso real): en
  ambos casos el handshake falla con `EOF` inmediato.
- La imagen `bfaar/nodo:latest` (compilada desde el fork `nucleo` de BFA,
  efectivamente geth v1.9.22) conecta con esos mismos peers al instante,
  confirmando que es un problema de compatibilidad de versión y no de red,
  bootnodes, ni IP bloqueada.

Conclusión: hasta que la red BFA actualice sus nodos, un geth genérico
reciente no es viable para producción.
