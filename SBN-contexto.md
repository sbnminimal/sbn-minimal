# Proyecto SBN — Contexto

Tienda online de joyería "SBN" (joyas de todos los días), orientada a Buenos Aires, Argentina. Sitio de una sola página (`index.html`), sin build ni framework — HTML + CSS + JS vanilla en un único archivo (~1687 líneas).

## Stack

- **Frontend**: HTML/CSS/JS vanilla, un solo archivo `index.html`.
- **Backend/datos**: Supabase (Postgres + REST API vía PostgREST). El front pega directo a la REST API con una API key `anon` embebida en el código.
- **Notificación por email**: EmailJS (`service_4dyfunc` / `template_nhwq4bq`).
- **Checkout**: no hay pasarela de pago real. El pedido se cierra por WhatsApp (`wa.me`) con un mensaje prearmado.
- **Imágenes**: se cargan desde links externos (típicamente Google Drive), proxeadas vía `wsrv.nl` para evitar problemas de CORS/hotlinking de Drive.

## Credenciales / config (ya públicas en el código, key `anon`)

```
SUPABASE_URL   = https://mpdrpzhziibnafdjdqdb.supabase.co
SUPABASE_KEY   (anon) = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Im1wZHJwemh6aWlibmFmZGpkcWRiIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzUyNTgxOTgsImV4cCI6MjA5MDgzNDE5OH0.Dvi_xq6ZV4MBQ1u3KbDNKxngaVeIWeCs7Ih7KbC39lA
WhatsApp destino = 541126647783 (CONFIG.whatsapp)
EmailJS public key = tKAlTZkUtszFTjeO6
```

⚠️ La key `anon` y toda la lógica de escritura (crear/editar/borrar productos y pedidos) están expuestas en el cliente. Si las políticas de Row Level Security (RLS) de Supabase no están bien configuradas, cualquiera con la key podría escribir directo en las tablas sin pasar por el panel admin. **Pendiente de revisar/confirmar RLS.**

## Esquema de datos en Supabase (tablas)

**`producto`**
- `id_producto` (texto, PK/ID único, ej: "ARI-001")
- `nombre_variante`
- `categoria`
- `subtitulo` (texto que aparece en el hero para esa categoría)
- `color`
- `precio_venta` (número, ARS)
- `stock_disponible` (número o `null` = ilimitado)
- `stock_minimo`
- `etiqueta` (texto libre: SALE/OFERTA/DESCUENTO → badge rojo; NUEVO/NEW/NOVEDAD → badge rosa; "SIN STOCK" → badge blanco)
- `link_imagenes` (URLs separadas por `|`, la primera es la foto principal)
- `descripcion_web`
- `activo` (booleano, visible/oculto en la tienda)

**`pedidos`**
- `id` (PK autoincremental)
- `nro_orden` (formato `SBN-YYYYMMDD-NNNN`, autoincremental por día)
- `fecha` (ISO)
- `detalle` (texto con el listado de productos del pedido)
- `total`
- `metodo_pago` ("Transferencia" o "Tarjeta")
- `nota` (nombre/teléfono que deja el cliente)
- `estado` (Pendiente / Confirmado / En preparación / Enviado / Entregado / Cancelado)
- `direccion_entrega`

**`config`** (clave/valor)
- `interes_tarjeta` — % de recargo por pago con tarjeta/link de pago
- `iva` — % de IVA aplicado sobre el recargo
- `admin_password_hash` — SHA-256 de la contraseña del panel admin (se verifica client-side contra este hash)

## Flujo de compra

1. Usuario navega catálogo (filtros por categoría, grid de productos, vista de detalle con galería y zoom).
2. Agrega productos al carrito (persistido en `localStorage`, key `sbn_cart`).
3. En el carrito elige método de pago: **Transferencia** (precio de lista) o **Link de pago/Tarjeta** (aplica recargo `interes_tarjeta` + `iva` sobre ese recargo, ver `calcFactor()`).
4. Deja nota con nombre y teléfono (obligatorio para poder enviar).
5. Al confirmar (`sendToWhatsApp()`):
   - Revalida stock en tiempo real contra Supabase antes de enviar.
   - Genera número de orden (`generarNroOrden()`).
   - Arma mensaje de WhatsApp con el detalle del pedido y lo abre en `wa.me`.
   - Guarda el pedido en la tabla `pedidos`.
   - Descuenta stock de cada producto comprado (y marca `etiqueta = 'SIN STOCK'` si llega a 0).
   - Envía notificación por email vía EmailJS.
   - Limpia el carrito.

No hay confirmación automática de pago — el pedido se coordina manualmente por WhatsApp después de este paso.

## Panel de administración

- Trigger oculto: ícono de engranaje casi invisible en el footer.
- Login con contraseña, verificada por SHA-256 contra `config.admin_password_hash`. Sesión guardada en `sessionStorage` (`sbn_admin`).
- **Tab Productos**: tabla editable inline (todos los campos), toggle de visibilidad, alta/baja, búsqueda, export/import CSV (columnas: `id_producto, nombre_variante, categoria, color, precio_venta, stock_disponible, descripcion_web, link_imagenes, activo, stock_minimo, etiqueta`).
- **Tab Pedidos**: tabla editable (estado, dirección de entrega), export CSV, búsqueda.
- Configuración de `interes_tarjeta` e `iva` editable desde el panel.
- Cambio de contraseña admin desde el panel (verifica la actual contra Supabase antes de guardar la nueva).

## Infraestructura / mantenimiento

**Problema resuelto (en curso de debug):** Supabase Free Plan pausa proyectos con poca actividad de base de datos durante 7 días. Se armó un workflow de GitHub Actions (`.github/workflows/supabase-keepalive.yml`) que hace un GET liviano a `producto` todos los días a las 12:00 UTC para mantener el proyecto activo. Usa los secrets `SUPABASE_URL` y `SUPABASE_ANON_KEY` del repo.

Estado actual: el workflow tiró error (`curl` exit code 22 = respuesta HTTP de error del servidor, oculta por el flag `-f`). Sospecha principal: secret mal copiado (comillas de más, espacios, o key/URL incorrecta). Se le pasó una versión de debug del workflow con `-w "\nHTTP status: %{http_code}\n"` sin `-f` para ver el código de estado y el mensaje real de error — **falta correr esa versión y revisar el log para diagnosticar (401 = key mal copiada, 404 = URL mal, otro = posible tema de RLS).**

```yaml
name: Supabase Keep-Alive

on:
  schedule:
    - cron: '0 12 * * *'   # todos los días a las 12:00 UTC (~9am Argentina)
  workflow_dispatch: {}

jobs:
  ping:
    runs-on: ubuntu-latest
    steps:
      - name: Ping Supabase REST API
        run: |
          curl -s -w "\nHTTP status: %{http_code}\n" "$SUPABASE_URL/rest/v1/producto?select=id_producto&limit=1" \
            -H "apikey: $SUPABASE_ANON_KEY" \
            -H "Authorization: Bearer $SUPABASE_ANON_KEY"
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_ANON_KEY: ${{ secrets.SUPABASE_ANON_KEY }}
```

## Pendientes / próximos pasos conocidos

1. Diagnosticar y resolver el error del workflow de keep-alive (ver arriba).
2. Revisar políticas RLS en Supabase para las tablas `producto`, `pedidos` y `config` (la key anon tiene permisos de escritura directa desde el cliente).
3. Confirmar que el keep-alive efectivamente previene la pausa (monitorear durante al menos una semana).
