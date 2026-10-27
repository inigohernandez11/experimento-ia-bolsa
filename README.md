# Experimento: IA invirtiendo en bolsa (cuenta demo de Trading 212)

Un agente de Claude se despierta dos veces cada día laborable en la nube de Anthropic
(con tu PC apagado), analiza 35 grandes acciones de EE. UU. y Europa, decide qué comprar o
vender con 1.000 € ficticios, ejecuta las órdenes en la **cuenta demo** de Trading 212 y lo
anota todo en `resultados.xlsx`.

> Dinero ficticio. Esto no es asesoramiento financiero. La API de Trading 212 está en beta y puede cambiar.

## Cómo funciona

```
Rutina en la nube (10:37 y 20:37, lunes a viernes)
  └─ python -m agente.estado      → cartera (Trading 212) + precios, noticias y fechas
                                    de resultados (Yahoo Finance)
  └─ Claude lee las noticias, busca contexto en la web, analiza y escribe su decisión
     (con motivo, base y fuentes) en data/decisiones/AAAAMMDD_HHMM.json
  └─ python -m agente.ejecutar    → comprueba reglas → órdenes en la DEMO → registra → Excel
  └─ git push a main              → data/ y resultados.xlsx quedan guardados en GitHub
```

- **Por qué esas horas:** a las 10:37 está abierta la bolsa europea y a las 20:37 la de EE. UU.
  (también la semana de finales de octubre en que los cambios de hora no coinciden).
- **Precios objetivos:** el precio de compra y de venta es el que Trading 212 ejecuta de verdad.
- **Seguridad:** la dirección de la cuenta demo está fijada en `agente/t212.py`, y una clave
  de la demo no sirve en la cuenta real.
- **Noticias:** en cada ejecución el agente recibe los titulares de los últimos 7 días de las
  acciones que tiene y de las que más se han movido, más la próxima fecha de resultados de cada empresa.
  Puede pedir las noticias de cualquier otra acción y hacer hasta 5 búsquedas web de contexto.
  Lee titulares y resúmenes, no artículos completos: así la red del entorno sigue cerrada a webs desconocidas.
- **Base de cada decisión:** cada orden se etiqueta como basada en `noticias`, `precios` o `ambos`,
  con sus fuentes. La hoja Analisis compara al final qué tipo de decisión acertó más.
- **Sin memoria perdida:** la cartera se reconstruye en cada ejecución a partir del historial de
  órdenes de Trading 212, así que es correcta aunque una ejecución falle a medias.

## Paso 1. Cuenta demo y clave de API en Trading 212

1. En la app o la web de Trading 212, cambia a la cuenta **Práctica/Demo** (cuenta Invest, en **EUR**).
   Usa esa cuenta solo para el experimento.
2. Con la cuenta demo seleccionada, ve a **Ajustes → API (Beta) → Generar clave**.
3. Marca todos los permisos de lectura y el de **ejecutar órdenes**.
   No restrinjas por IP (los servidores en la nube cambian de IP).
4. Copia la **clave** y el **secreto** (el secreto solo se muestra una vez). No los compartas con nadie, tampoco en un chat.

## Paso 2. Repositorio en GitHub

1. Crea un repositorio **privado** en GitHub (por ejemplo `experimento-ia-bolsa`).
2. Sube todos los archivos de esta carpeta (en la web de GitHub: **Add file → Upload files**,
   arrastra el contenido de la carpeta y confirma en la rama `main`).

## Paso 3. Entorno en la nube

1. Entra en [claude.ai/code](https://claude.ai/code). La primera vez te pedirá conectar GitHub: dale acceso a ese repositorio.
2. Pulsa el icono de la nube (encima del cuadro de mensaje) → **Add cloud environment**:
   - **Name:** `bolsa-demo`
   - **Network access:** `Custom`. En **Allowed domains** escribe:
     ```
     *.yahoo.com
     demo.trading212.com
     ```
     y marca **Also include default list of common package managers**.
   - **Setup script:**
     ```bash
     #!/bin/bash
     pip install -q --break-system-packages requests yfinance pandas openpyxl tzdata || pip install -q requests yfinance pandas openpyxl tzdata || true
     ```
   - Crea el entorno.
3. **Credencial de Trading 212** (elige una opción):
   - **Planes Pro o Max (recomendado):** vuelve a abrir el entorno (icono de ajustes) →
     **API credentials → Add credential**:
     - **Name:** `Trading 212 demo`
     - **Allowed websites:** `demo.trading212.com`
     - **Custom headers:** nombre `Authorization`, prefijo `Basic` (cambia `Bearer` por `Basic`)
       y como valor el texto que te da este comando con tu clave y secreto:
       - En el navegador (F12 → Consola): `btoa("TU_CLAVE:TU_SECRETO")`
       - En Windows PowerShell: `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("TU_CLAVE:TU_SECRETO"))`
       - En macOS o Linux: `echo -n "TU_CLAVE:TU_SECRETO" | base64`
     - Pulsa **Connect**. La clave nunca llega a Claude ni a los comandos que ejecuta.
   - **Planes Team o Enterprise:** en **Environment variables** del entorno añade
     ```
     T212_API_KEY=tu_clave
     T212_API_SECRET=tu_secreto
     ```
     (son visibles para quien use ese entorno; al ser personal, solo tú).

## Paso 4. Crear la rutina

1. Ve a [claude.ai/code/routines](https://claude.ai/code/routines) → **New routine**.
2. **Nombre:** `Agente bolsa demo`. **Instrucciones:** pega el texto de `PROMPT_RUTINA.md`.
3. **Modelo:** elige uno y **no lo cambies en todo el mes**, para que el experimento sea comparable.
4. **Repositorio:** el que creaste. **Entorno:** `bolsa-demo`.
5. **Trigger → Schedule → Weekdays → 10:37**. Después **Add another trigger → Schedule → Weekdays → 20:37**.
   Si el formulario no permite dos horarios, crea una segunda rutina idéntica con las 20:37,
   o desde Claude Code en la terminal usa `/schedule update` con la expresión cron `37 10,20 * * 1-5`
   (después comprueba en la página de la rutina que las horas que muestra son las de España).
6. **Connectors:** quita todos; el agente no los necesita.
7. Pulsa **Create**.

Cada ejecución consume uso de tu plan de Claude (unas 44 ejecuciones al mes).

## Paso 5. Prueba

En la página de la rutina pulsa **Run now** y abre la sesión que se crea. Comprueba que:

- `agente.estado` funciona sin errores 401/403 ni `host_not_allowed`.
- En **avisos** no aparecen muchas acciones "No encontradas" (si aparece alguna, simplemente no se operará).
- Al final hay un commit en `main` con `data/evolucion.csv`, `data/mapeo_t212.json` y `resultados.xlsx`.

Si la pruebas con el mercado cerrado, el agente no podrá operar, pero sí registrará la foto de la cartera.

## Ver los resultados

En GitHub abre `resultados.xlsx` → **Download raw file**, y ábrelo en Excel o súbelo a Google Drive.

| Hoja | Contenido |
|---|---|
| Resumen | Valor total, ganancia, rentabilidad frente al índice, acciones que más aportaron, mejores y peores ventas |
| Operaciones | Cada compra y venta: fecha, cantidad, precio, importe en €, comisiones, motivo, base y fuentes |
| Ventas | Por cada venta: (precio de venta − precio medio de compra) × cantidad, y la ganancia en € |
| Posiciones | Lo que la cartera tiene ahora y cuánto gana o pierde |
| Contribucion | Aportación total de cada acción (realizada + no realizada) |
| Evolucion | Valor de la cartera en cada ejecución frente al mismo dinero en el índice, con gráfico |
| Decisiones | Todas las órdenes propuestas, también las rechazadas, con su motivo, base y fuentes |
| Analisis | Si cada compra o venta acertó (el precio subió tras comprar o bajó tras vender), y el porcentaje de aciertos según la decisión se basara en noticias, en precios o en ambos |

En azul, los datos de Trading 212 o Yahoo Finance; en negro, las fórmulas; en verde, los enlaces entre hojas.

## Problemas frecuentes

- **HTTP 401:** la credencial no es correcta. Revisa que el valor sea el base64 de `CLAVE:SECRETO`,
  con prefijo `Basic` y el sitio `demo.trading212.com`, y que la clave sea de la cuenta demo.
- **HTTP 403 "Scope(...) missing":** a la clave le falta un permiso. Genera otra con todos los permisos.
- **`host_not_allowed`:** falta un dominio en **Allowed domains** del entorno.
- **Errores de Yahoo Finance:** ese día no se compra (las ventas sí funcionan) y puede faltar alguna
  noticia o fecha de resultados (aparece en los avisos). Si pasa a menudo, habrá que cambiar de proveedor.
- **Órdenes "NO EJECUTADA":** el mercado estaba cerrado (festivo o media sesión); la orden se cancela sola.
- **No se puede subir a main:** revisa si la rama tiene protección en GitHub (Settings → Branches).

## Cambiar reglas o volver a empezar

- Reglas, universo e índice están en `config.json`. Cámbialos antes de empezar, no a mitad del experimento.
- Para reiniciar: borra `data/*.csv`, `data/ejecutadas.txt`, `data/decisiones/*.json` y `resultados.xlsx`,
  y pon en `config.json` la nueva fecha de inicio, por ejemplo `"fecha_inicio": "2026-11-02"`,
  para que se ignoren las órdenes anteriores.
- Para regenerar solo el Excel: `python -m agente.ejecutar --solo-informe`.

## Prueba local (opcional)

```bash
pip install -r requirements.txt pytest
python -m pytest -q tests
```

La prueba usa un bróker y precios simulados; no se conecta a Trading 212 ni a Yahoo Finance.
