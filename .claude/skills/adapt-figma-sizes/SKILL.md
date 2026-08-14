---
name: adapt-figma-sizes
description: Adapta una pieza gráfica seleccionada en Figma a múltiples tamaños (por defecto 1080x1080, 1920x1080, 1200x628), rearmando el layout de forma inteligente en lugar de escalar. Usar cuando el usuario pida "adaptar", "redimensionar" o "generar variantes de tamaño" de una pieza de Figma.
---

# Adaptar piezas gráficas a otros tamaños (Figma)

Este skill toma la pieza gráfica actualmente seleccionada en Figma y genera variantes en otros tamaños, respetando jerarquía visual, legibilidad y elementos de marca — no es un simple "escalar el frame".

## Tamaños por defecto

Si el usuario no pide tamaños específicos, generar estos tres:

| Nombre          | Tamaño (px)  | Uso típico                              |
|-----------------|--------------|------------------------------------------|
| Cuadrado        | 1080 x 1080  | Feed (Instagram/LinkedIn/Facebook)        |
| Horizontal wide | 1920 x 1080  | Presentaciones, pantallas, YouTube        |
| Link preview     | 1200 x 628   | Facebook/LinkedIn link preview, banners   |

Tamaños adicionales frecuentes que se piden ad-hoc (no default, pero comunes):
- **1080 x 1920** — Stories/Reels. Requiere zona segura (ver más abajo).

Si el usuario pide un tamaño puntual, agregarlo a la corrida sin tocar la lista default para futuras corridas, salvo que pida guardarlo como nuevo default.

## Flujo de trabajo

1. **Cargar el skill `/figma-use` primero** — es obligatorio antes de llamar a `use_figma`. Seguir sus instrucciones para autenticación y convenciones del MCP de Figma.

2. **Entender la pieza de origen.** Sobre la selección activa del usuario en Figma:
   - `get_metadata` para dimensiones exactas, jerarquía de capas y nombres.
   - `get_screenshot` para tener referencia visual y poder razonar el layout.
   - `get_design_context` si necesitás además generar código a partir del diseño (no es necesario solo para clonar/reacomodar dentro de Figma).

3. **Clonar, no recrear desde cero.** La forma más fiel de generar una variante es clonar el frame original completo (`node.clone()`), renombrarlo (`{nombre original} — {ancho}x{alto}`) y reposicionarlo lejos del original en el canvas. Esto preserva automáticamente fills, efectos, fuentes y estilos — mucho más seguro que reconstruir el diseño a mano.

4. **Ignorar el auto-resize de Figma y reposicionar todo a mano.** Al hacer `frame.resize(w, h)` sobre el clon, Figma puede aplicar constraints (`SCALE`, `MIN`, `MAX`) que estiran o desplazan hijos de forma no uniforme y rompen la composición (por ejemplo, dos imágenes de un collage que se separan dejando un hueco). Después de clonar y redimensionar el frame contenedor, **reposicioná y re-redimensioná explícitamente cada hijo** (fondo, logo, textos, imágenes) según la estrategia de layout del tamaño destino — no confíes en el resize automático.

5. **Clasificar cada elemento**, porque cada uno se adapta distinto:
   - **Fondo/decorativo (blur, ruido, gradientes)**: debe cubrir el nuevo canvas completo. Si el fondo usa formas decorativas (ellipses, blobs) para lograr un degradé, esas formas **también** hay que reescalarlas y reposicionarlas proporcionalmente al nuevo ancho/alto — si solo agrandás el frame contenedor sin tocarlas, queda un hueco de cobertura visible en el borde.
   - **Logo/isotipo**: tamaño fijo (no lo estires), reposicionar según el layout del nuevo formato.
   - **Texto (título, cuerpo, CTA)**: ver sección "Achicar texto sin romper fuentes" más abajo — es el punto más delicado.
   - **Imagen/foto principal**: recortar ("cover") priorizando el punto focal, nunca deformar. Si es un collage de varias imágenes superpuestas, escalarlas todas por el mismo factor como una unidad para no romper la composición.

6. **Achicar texto sin romper fuentes (importante).** Cualquier operación que dispare un recálculo de línea/tamaño de fuente (`resize()` en ancho de un text node, `setRangeFontSize`, `fontName`, etc.) requiere que la fuente esté cargada vía `figma.loadFontAsync`. Si la pieza usa una fuente que **no existe** en el entorno de Figma (pasa seguido con fuentes de sistema tipo "SF Pro Display" que no están instaladas del lado del plugin), esa carga falla y cualquier `resize()` directo sobre el text node deja el texto roto (la caja cambia de tamaño pero las letras no, y se solapan/desbordan).
   - **Solución:** no toques el text node directamente. Agrupalo (`figma.group([...nodes], parent)`) y aplicá **`group.rescale(factor)`** en vez de `group.resize()`. `rescale()` escala geométricamente el grupo completo — incluyendo el `fontSize` del texto — sin necesitar la fuente cargada, porque no reflowea el texto, solo aplica una transformación proporcional (igual que la herramienta "Scale" de Figma, tecla K). Este es el método a usar siempre que haya que achicar/agrandar un bloque de texto para un formato más chico o más angosto.
   - Si vas a mezclar varios text nodes en un layout compacto (ej. varios niveles de texto en una columna angosta), primero reposicionalos en un stack vertical prolijo (sin resize, solo `x`/`y`) y **recién ahí** agrupalos y aplicá `rescale()` sobre el grupo entero — así se escalan todos juntos manteniendo las proporciones relativas entre ellos.

7. **Definir estrategia de layout por tamaño destino** (regla general, ajustar según el contenido real):
   - **1080x1080 (cuadrado)**: layout centrado, generalmente el más parecido al original.
   - **1920x1080 (horizontal wide)**: texto a la izquierda, imagen a la derecha (o viceversa si el original ya tenía esa dirección) — no centrar todo dejando espacio vacío a los costados.
   - **1200x628 (link preview)**: mismo criterio de **texto a la izquierda, foto siempre a la derecha** para aprovechar el espacio horizontal — no lo apiles verticalmente salvo que no entre ni con `rescale()`. Alinear texto y logo a la **izquierda** (no centrado), salvo que el usuario pida lo contrario.
   - **1080x1920 (stories)**: pedirle al usuario los márgenes de zona segura (top/bottom, suelen rondar 250-400px cada uno) si no los dio, y centrar/posicionar **todo** el contenido de primer plano (texto, logo, imagen) dentro de esa banda segura. El fondo (gradiente/blur) puede seguir a sangre completa cubriendo los 1920px igual.

8. **Verificar visualmente.** Sacar `get_screenshot` de cada variante generada y confirmar que:
   - No hay texto cortado, desbordado o solapado.
   - El fondo cubre 100% del canvas sin huecos (revisar bordes, sobre todo si hubo escalado no uniforme).
   - El logo no quedó pixelado ni fuera de zona segura.
   - La jerarquía visual se mantiene.
   Si algo no cierra, ajustar antes de reportar como terminado.

9. **Reportar al usuario** qué variantes se generaron y preguntar si quiere ajustes puntuales antes de darlo por cerrado.

## Notas

- Nunca asumas que "escalar todo proporcionalmente" es aceptable entre relaciones de aspecto muy distintas (ej. cuadrado → wide): siempre replanteá la composición.
- Si la pieza tiene texto legal/disclaimer chico, verificar que siga siendo legible en el tamaño más chico (1200x628).
- Si el archivo de Figma usa un design system con componentes (botones, badges), reutilizar esos componentes en las variantes en vez de recrear las formas a mano.
- Preferí alineación a la izquierda para texto+logo en layouts lado a lado (texto/imagen), salvo que el usuario pida centrado explícitamente.
