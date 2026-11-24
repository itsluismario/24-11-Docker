# 24-11-Docker

Notas de aprendizaje sobre Docker: imágenes, contenedores, Dockerfile, volúmenes y Docker Compose, con R (`rocker/r-ver`) y Python (`uv`).

---

## 1. Conceptos básicos

| | Qué es | Cuándo se define |
|---|---|---|
| **Dockerfile** | La receta | Lo escribes tú |
| **Imagen** | Dependencias y versiones empaquetadas | `docker build` |
| **Contenedor** | La imagen corriendo | `docker run` |
| **Puertos** | Conexión entre tu máquina y el contenedor | `docker run -p` |
| **Volumen** | Carpeta de tu máquina conectada al contenedor | `docker run -v` |

- La **imagen** se parece a una *clase* y el **contenedor** a un *objeto*: de una imagen puedes lanzar muchos contenedores.
- Cada instrucción del Dockerfile crea una **capa**. Docker guarda en caché las capas que no cambian, así que **el orden importa**: lo que menos cambia va primero.
- Un contenedor **vive mientras vive su proceso principal**, el comando de `CMD`. Cuando ese proceso termina, el contenedor se detiene.
- Lo que se crea dentro de un contenedor **se pierde al borrarlo**. Lo permanente va en el Dockerfile, y los datos en volúmenes.
- **Cliente y daemon**: el comando `docker` solo envía órdenes. El motor que construye y ejecuta es el daemon, que en Mac arranca **Docker Desktop**.

---

## 2. Dockerfile con R (`rocker/r-ver`)

```dockerfile
FROM rocker/r-ver:4.6.2

CMD ["R"]
```

- `rocker/r-ver` fija la versión de R, y el repositorio de CRAN queda fijado a una fecha. Esto lo hace reproducible.
- Usa siempre una versión exacta, nunca `latest`.
- Ejemplo más completo:

```dockerfile
FROM rocker/r-ver:4.6.2

RUN apt-get update && apt-get install -y --no-install-recommends \
        libcurl4-openssl-dev libssl-dev libxml2-dev \
    && rm -rf /var/lib/apt/lists/*

RUN install2.r --error --skipinstalled dplyr readr ggplot2

WORKDIR /app
COPY script.R .
CMD ["Rscript", "script.R"]
```

### Instrucciones principales

| Instrucción | Para qué sirve |
|---|---|
| `FROM` | Imagen base (siempre va primero) |
| `RUN` | Ejecuta un comando **al construir** |
| `WORKDIR` | Carpeta de trabajo dentro de la imagen |
| `COPY` | Copia archivos de tu máquina a la imagen |
| `ENV` | Define variables de entorno |
| `EXPOSE` | Solo **documenta** un puerto; no lo abre |
| `CMD` | Comando por defecto **al ejecutar** |

---

## 3. Dockerfile con Python (`uv`)

```dockerfile
FROM ubuntu:24.04

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Entorno fuera de /workspace, para que el volumen no lo tape
ENV UV_PROJECT_ENVIRONMENT=/opt/venv

WORKDIR /workspace

# 1. Primero solo las dependencias (aprovecha la caché)
COPY pyproject.toml uv.lock .python-version ./
RUN uv sync --locked

# 2. Después el código
COPY . .

CMD ["uv", "run", "python", "hello.py"]
```

### `pyproject.toml` y `uv.lock`

- **`pyproject.toml`**: lo editas tú. Dice *qué* paquetes necesitas.
- **`uv.lock`**: lo genera uv. Fija la **versión exacta** de todo, incluidas las dependencias de tus dependencias.
- El lockfile se crea **en tu Mac**, no dentro de Docker:

```bash
uv init            # crea pyproject.toml y .python-version
uv add cowsay==6.1 # añade la dependencia y crea/actualiza uv.lock
uv lock            # regenera uv.lock si editaste pyproject.toml a mano
```

- `uv sync --locked` instala exactamente lo del lockfile, y falla si no coincide con `pyproject.toml`.
- El `.venv` de tu Mac **no sirve** en Linux, porque algunos paquetes tienen partes compiladas para macOS. Por eso se crea dentro de la imagen.

### `.dockerignore`

```
.git/
.venv/
__pycache__/
*.pyc
.DS_Store
```

---

## 4. `COPY` y rutas

```
COPY  hello.py   ./
      └ origen ┘ └ destino ┘
```

- **Origen**: ruta en tu Mac, relativa al contexto de construcción (el `.` de `docker build`).
- **Destino**: ruta dentro de la imagen, relativa al `WORKDIR`.
- `./` significa "la carpeta actual" = el `WORKDIR`. Con `WORKDIR /workspace`, `./` equivale a `/workspace/`.
- La `/` final indica que el destino es una carpeta. Es obligatoria cuando copias varios archivos.
- Usa **`COPY . .`** en vez de `COPY * ./`: con `*`, Docker copia el **contenido** de las carpetas y pierde la estructura.

---

## 5. Volúmenes (`--volume` / `-v`)

```bash
docker run --rm -it --volume "$(pwd)":/workspace mi-imagen
#                            └─ tu Mac ─┘ └ contenedor ┘
```

- Conectan una carpeta de tu Mac con una del contenedor, **en vivo y en las dos direcciones**.
- `COPY` actúa al construir y hace una copia fija. `--volume` actúa al ejecutar y conecta la carpeta en vivo.
- En Mac, la ruta debe ser absoluta: por eso se usa `"$(pwd)"`.

### Un volumen **tapa** la carpeta donde se monta

Analogía de los **portarretratos**: el volumen pone la foto de tu Mac *delante* de lo que había en esa carpeta. Lo de atrás no se borra, pero deja de verse.

```
❌ Mal                         ✅ Bien
/                              /
├── workspace/  ← volumen      ├── workspace/  ← volumen
│   └── .venv/  ← tapado       │   └── hello.py
└── opt/                       └── opt/
                                   └── venv/   ← intacto
```

**Regla:** todo lo que está *dentro* de la carpeta que montas queda tapado; lo que está *fuera*, no. Por eso se usa `ENV UV_PROJECT_ENVIRONMENT=/opt/venv`. Puede ser cualquier ruta fuera de `/workspace`; `/opt` es solo la convención para software adicional.

---

## 6. Docker Compose

- **El Dockerfile** dice **cómo construir** la imagen ("¿qué hay dentro?").
- **Compose** dice **cómo ejecutar** los contenedores ("¿cómo lo arranco?"): puertos, volúmenes y nombres.
- Compose no sustituye al Dockerfile: `build: .` lo usa. Es tu `docker run` escrito en un archivo, y permite arrancar varios servicios juntos.

```yaml
services:
  analysis-env:
    build: .                    # docker build .
    image: 2026-11-24-docker    # --tag
    ports:
      - "127.0.0.1:2222:22"     # -p (solo accesible desde tu computadora)
    volumes:
      - .:/workspace            # -v  tu_Mac:contenedor
```

- Los volúmenes necesitan **dos rutas** (`origen:destino`).
- Pon los puertos **entre comillas**.
- El puerto 22 es SSH: solo sirve si la imagen tiene un servidor SSH.

```bash
docker compose up --build                 # construir y arrancar
docker compose up -d                      # en segundo plano
docker compose run --rm analysis-env bash # terminal dentro
docker compose down                       # detener y borrar
```

---

## 7. Comandos esenciales

```bash
# Construir (desde la carpeta del proyecto; el "." es obligatorio)
cd /Users/itsluismario/Downloads/UBC/522/Lecture3
docker build --tag lecture3-r:2026-11-14 .

# Ejecutar
docker run --rm -it lecture3-r:2026-11-14        # interactivo; se borra al salir
docker run -dit --name mi-r lecture3-r:2026-11-14 # en segundo plano

# Conectarse a un contenedor que está corriendo
docker attach mi-r          # salir sin matarlo: Ctrl+P y luego Ctrl+Q
docker exec -it mi-r bash   # nueva terminal dentro

# Ver y gestionar
docker images               # imágenes
docker ps / docker ps -a    # contenedores (corriendo / todos)
docker stop | start | rm mi-r
docker logs mi-r
docker rmi lecture3-r:2026-11-14
docker version              # comprueba que el daemon responde (sección Server)
```

### Nombres y etiquetas: `nombre:etiqueta`

```
lecture3-r:2026-11-14
└ nombre ┘ └ etiqueta ┘
```

- El **nombre** dice *qué* es; la **etiqueta**, *qué versión*. Sin etiqueta, Docker usa `latest`.
- `lecture3-r` es un nombre inventado: puede ser cualquiera.
- Reglas: **solo minúsculas**; puedes usar letras, números, `-`, `_` y `.`; no puede empezar ni terminar con separador.

### Opciones (flags)

- **Un guion** para opciones de una letra: `-i`, `-t`, `-p`, `-v`. Se pueden juntar: `-it`.
- **Dos guiones** para palabras completas: `--rm`, `--tag`, `--name`, `--volume`.
- `-it` es necesario para programas interactivos como R o Python. Sin él, el proceso termina al instante.

---

## 8. Errores que encontré y su solución

| Error / síntoma | Causa | Solución |
|---|---|---|
| `docker build` no hace nada | Faltaba el contexto | `docker build --tag nombre .` (con el `.`) |
| `--tag 2026-11-14-docker.` | El punto estaba pegado al nombre | Separar con un espacio: `... 2026-11-14-docker .` |
| `failed to connect to the docker API ... docker.sock` | Docker Desktop no estaba corriendo | `open -a Docker` y esperar a que arranque |
| `unknown instruction: CMD["R"]` | Faltaba un espacio | `CMD ["R"]` |
| `-rm` no funciona | Opción larga con un solo guion | `--rm` |
| La imagen no aparece | El nombre no coincidía (día 14 vs 24) | Revisar con `docker images` y copiar `REPOSITORY:TAG` |
| `coway was not found in the package registry` | Errata en el nombre del paquete | `cowsay==6.1`, o usar `uv add` |
| Python no encuentra los paquetes al usar `-v` | El volumen tapaba `/workspace/.venv` | `ENV UV_PROJECT_ENVIRONMENT=/opt/venv` |
| Volumen en Compose `- ./workspace` | Solo tenía una ruta | `- .:/workspace` |

---

## 9. Mis preguntas (y respuestas cortas)

1. **¿La imagen es donde defino dependencias, versiones y puertos? ¿El contenedor es la imagen corriendo?**
   Sí, el contenedor es la imagen corriendo. Las dependencias se definen en el Dockerfile y se empaquetan en la imagen, pero **los puertos se conectan al ejecutar** (`-p`). `EXPOSE` solo los documenta.
2. **¿Por qué `docker build` no hizo nada?**
   Hay que entrar a la carpeta (`cd`) y pasar el contexto (`.`).
3. **¿Es correcto `docker build --tag 2026-11-14-docker.`?**
   Falta un espacio antes del punto.
4. **¿Cómo veo la imagen creada?**
   `docker images`, o la pestaña *Images* de Docker Desktop.
5. **¿Qué es `lecture3` en `lecture3-r:2026-11-14`?**
   Un nombre inventado para la imagen. Puede ser cualquiera, en minúsculas.
6. **¿Cómo dejo corriendo el contenedor?**
   Con `-d`: `docker run -dit --name mi-r imagen`. Luego `docker attach` o `docker exec`.
7. **¿Cómo se crea el `uv.lock`?**
   En tu Mac, con `uv add` o `uv lock`.
8. **¿Qué significa `./` en `COPY hello.py ./`?**
   La carpeta actual dentro de la imagen, es decir, el `WORKDIR`.
9. **¿Y si pongo `COPY * ./`?**
   Copia los archivos, pero aplana las carpetas. Mejor `COPY . .` con un `.dockerignore`.
10. **¿Qué hace `--volume`?**
    Conecta en vivo una carpeta de tu Mac con una del contenedor.
11. **¿En Linux el venv tiene que estar en `/opt/venv`?**
    No. Puede estar en cualquier ruta **fuera** de la carpeta que montas.
12. **¿El venv debe estar al mismo nivel que `/workspace` y no dentro?**
    Sí. Debe estar fuera de `/workspace`, por ejemplo en `/opt/venv`, porque `/opt` y `/workspace` cuelgan los dos de `/`.
13. **¿Diferencia entre Docker Compose y el Dockerfile?**
    El Dockerfile **construye** la imagen; Compose **ejecuta** los contenedores con su configuración (puertos, volúmenes), y usa el Dockerfile mediante `build: .`.
