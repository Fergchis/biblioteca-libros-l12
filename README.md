# biblioteca-libros

Microservicio de libros del proyecto guía de DSY1107, **tal como queda al terminar L8**, ya separado
en su propio repositorio, más lo que agrega L10. Node plano: `libros.mjs` sirve `datos/catalogo.json` en el
puerto **3001**, con 300 ms de latencia simulada. Cualquier `GET` responde **200** con la lista de libros. Desde
L10, si recibe `KAFKA_BROKERS`, además publica el catálogo en Kafka (ver «Kafka (L10)»).

| Archivo | Qué es |
|---|---|
| `libros.mjs` | El servidor HTTP. Relee `datos/catalogo.json` en cada petición. Con `KAFKA_BROKERS`, arranca `catalogo-kafka.mjs` |
| `catalogo-kafka.mjs` | L10: publica `libro.publicado` y `libro.actualizado` en `biblioteca.catalogo` |
| `package.json` | Su única dependencia es `@platformatic/kafka` 2.14.0, el cliente del curso (`npm ci` antes de `node libros.mjs`) |
| `datos/catalogo.json` | El catálogo que se sirve |
| `datos/prestamos.json` | Lo deja `sembrar.mts` junto al catálogo, con `sub` de ejemplo. Ningún servicio lo lee: los préstamos viven en Postgres, en `biblioteca-prestamos` |
| `sembrar.mts` | El seed de L4: arma el catálogo con Google Books y las sinopsis con un modelo (o `--respaldo`, sin conexión) |
| `Dockerfile` | La imagen que construye `biblioteca-plataforma/compose.yml` |
| `.env.example` | Los nombres que lee `sembrar.mts`. Solo hace falta para sembrar |

## Cómo usarlo

Necesitas **Node 24.15.0 o superior**. Los mismos comandos sirven en Windows (PowerShell), macOS y Linux:

```bash
cd $HOME/DSY1107/l12/biblioteca-libros
npm ci
node libros.mjs
```

En otra terminal:

```bash
curl.exe -i http://localhost:3001/libros
```

En macOS y Linux, `curl` sin el `.exe`. Tiene que responder **200** con un arreglo JSON de libros.

Con todo el sistema, desde `biblioteca-plataforma`: `docker compose up -d --build --wait`. Mira el README de
ese repositorio.

## Kafka (L10): el catálogo en `biblioteca.catalogo`

`biblioteca.catalogo` es un topic **compactado** (key `libroId`, `cleanup.policy=compact`, lo crea
`biblioteca-plataforma/kafka/topics.sh`): Kafka puede borrar las versiones viejas de un libro, pero nunca la última.
Por eso cada evento lleva el libro completo (`titulo`, `autor`, `anio`, `paginas`, `categoria`, `isbn13`,
`ejemplares`; la sinopsis no), y quien lo lea desde el offset 0 tiene el catálogo vigente.

| Evento | Cuándo |
|---|---|
| `libro.publicado` | Un libro de `datos/catalogo.json` que el topic todavía no tiene |
| `libro.actualizado` | Un libro cuyo estado en el topic es distinto del de `datos/catalogo.json` |

Al arrancar lee su propio topic hasta el final (grupo `biblioteca-libros`, sin confirmar offsets) para saber qué
ya publicó: la primera vez publica los 24 libros; al reiniciar, nada. Después revisa `datos/catalogo.json` cada 5
segundos y publica solo lo que cambió. Sin `KAFKA_BROKERS` no publica y sirve igual. El productor es el de
`biblioteca-prestamos`: idempotente, `acks=all`, `retries: 10` y `compatibilityPartitioner`. Lo que se ve en el log:

```
[libros] kafka: 0 libros ya estaban en biblioteca.catalogo
[libros] kafka: 24 libro.publicado y 0 libro.actualizado (libros 1, 2, 3, ..., 24)
[libros] kafka: 0 libro.publicado y 1 libro.actualizado (libros 3)
```

Lo lee `biblioteca-eventos`, grupo `biblioteca-catalogo`, que reconstruye la tabla `catalogo_libros`. Un libro que
se borra del JSON no se retira del topic: eso sería un *tombstone* (value nulo), que este laboratorio no usa.

## Volver a sembrar

Copia `.env.example` a `.env` (Windows: `Copy-Item .env.example .env`; macOS y Linux: `cp .env.example .env`),
pon tus dos `sub` y corre:

```bash
node --env-file=.env sembrar.mts --respaldo
```

Sin `--respaldo`, y con las dos API keys puestas, arma el catálogo de verdad. Después de sembrar, `git status`
marca los dos JSON de `datos/` como modificados: es lo esperado.
