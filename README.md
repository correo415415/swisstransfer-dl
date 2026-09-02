# swisstransfer-dl

Descargador de línea de comandos para [SwissTransfer](https://www.swisstransfer.com) con salida bonita (barras de progreso, árbol de archivos, resumen).

```
╭──────────────────────────────────╮
│ SwissTransfer Downloader  v1.0.0 │
╰──────────────────────────────────╯

        Enlace   01a061be-81b3-739f-a19e-e8853a7638d6
      Archivos   4
  Tamaño total   4 B
        Expira   2026-09-03 11:00  (1 día)
       Destino   /tmp/t3

📁 Carpeta
└── 📁 test1
    ├── 📄 1  (1 B)
    ├── 📄 1 (3.ª copia)  (1 B)
    ├── 📄 1 (copia)  (1 B)
    └── 📄 1 (otra copia)  (1 B)

  Total (4 archivos) ━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 4/4 bytes 3 bytes/s 0:00:00
  ✔ 1 (3.ª copia)    ━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 1/1 bytes ?         0:00:00
  ...

╭────────────────────────────────────────────────────╮
│ ✔ 4/4 archivos descargados  ·  4 B en 1.4s (3 B/s) │
│ 📂 /tmp/t3                                         │
╰────────────────────────────────────────────────────╯
```

## Características

- ✅ Un solo archivo, varios archivos y **carpetas** (se recrea la jerarquía de directorios).
- 🔒 Enlaces protegidos con **contraseña** (`-p` o de forma interactiva).
- ⏯ **Reanudación** de descargas interrumpidas (`.part` + `Range`).
- ↷ Omite archivos ya descargados (verifica tamaño); `--force` para sobrescribir.
- 🔁 Reintentos automáticos con backoff.
- 🛡 Verificación de tamaño y saneado de rutas (sin path traversal).
- 📋 `--list` para inspeccionar el contenido sin descargar.
- Acepta la URL completa **o solo el UUID**.

## Instalación

```bash
pip install -r requirements.txt   # requests + rich
```

Requiere Python 3.9+.

## Uso

```bash
python swisstransfer_dl.py https://www.swisstransfer.com/dl/<uuid>
python swisstransfer_dl.py <uuid> -o ~/Descargas
python swisstransfer_dl.py <enlace> -p "contraseña"
python swisstransfer_dl.py <enlace> --list
python swisstransfer_dl.py <enlace> -s          # crea subcarpeta con el título/UUID
```

| Opción | Descripción |
|---|---|
| `-o, --output DIR` | Directorio de destino (por defecto `.`) |
| `-p, --password PWD` | Contraseña de la transferencia |
| `-l, --list` | Solo listar |
| `-f, --force` | Sobrescribir archivos existentes |
| `-s, --subdir` | Crear subdirectorio con título/UUID |
| `-r, --retries N` | Reintentos por archivo (3) |
| `-t, --timeout S` | Timeout de red (30 s) |
| `-q, --quiet` | Sin banner |

Códigos de salida: `0` OK · `1` error / descargas fallidas · `2` contraseña requerida · `130` cancelado.

## Cómo funciona

1. `GET /dl/{link}` → la página Inertia incluye un `<script data-page="app">` JSON con los metadatos de la transferencia (`transfer.files[]` con `id`, `path`, `size`).
2. Si la transferencia está protegida (`component == link/password`), se envía `POST /dl/{link}` con `{"password": ...}` y cabeceras `X-Inertia` + `X-XSRF-TOKEN` (cookie `SWISSTRANSFER-API-XSRF-TOKEN`).
3. Para cada archivo: `GET /api/1/links/{link}/files/{file}` (`Accept: application/json`) → devuelve una URL S3 prefirmada (válida 1 h).
4. Descarga en streaming con soporte `Range` para reanudar.
