# borregogfuel/borregogfuel

Repo especial de GitHub (`username/username`) que genera el README de perfil.
Usa el sistema de Andrew Grant (`Andrew6rant/Andrew6rant`) como base: un
GitHub Action corre `today.py`, que llama a la API de GitHub, calcula stats
y escribe los valores directamente dentro de `light_mode.svg` y
`dark_mode.svg` (los SVGs son el README visualmente, referenciados por
`README.md` con `<picture>`/`prefers-color-scheme`).

## Cómo funciona el pipeline

- `.github/workflows/build.yaml` corre en push a `main` y por cron diario
  (4am UTC).
- Instala deps, ejecuta `python today.py` (usa secrets `ACCESS_TOKEN` y
  `USER_NAME`), y si hay cambios hace commit ("Updated README") + push
  automático de vuelta al repo.
- **Importante**: nada de esto corre localmente. Si abres los SVG en el
  editor o en un servidor local, ves el archivo estático tal cual quedó en
  disco — no se recalcula nada ahí. Solo el Action (en los servidores de
  GitHub) ejecuta `today.py`.
- Consecuencia práctica de trabajar en este repo: el Action se dispara casi
  en cada push nuestro, y muchas veces alcanza a correr y pushear su propio
  commit ANTES de que nuestro siguiente push llegue. Es normal que
  `git push` sea rechazado por "fetch first" — hay que `git fetch` +
  `git rebase origin/main`, resolver conflictos (casi siempre en los campos
  con `id=` que el bot recalcula: `age_data`, `commit_data`, `repo_data`,
  `star_data`, `follower_data`, `loc_data`/`loc_add`/`loc_del`), y volver a
  pushear.

## Cómo están estructurados los SVG

- Dos archivos casi idénticos en contenido: `light_mode.svg` y
  `dark_mode.svg`, solo cambian los colores (`.key`, `.value`, `.cc`,
  `.addColor`, `.delColor`) y el fondo.
- El bloque de la izquierda (`x="15"`, clase `ascii`) es el arte ASCII fijo
  (una foto convertida a ASCII), no lo tocamos.
- El bloque de la derecha (`x="332"`) es tipo "neofetch": cada fila es un
  `<tspan>` con label (`class="key"`), puntos de relleno (`class="cc"`,
  muchos con `id="..._dots"`) y valor (`class="value"`, muchos con
  `id="..."`).
- Estilo monoespaciado: **todas las filas de contenido dentro de un mismo
  bloque deben medir el mismo ancho total en caracteres** (actualmente
  target = 59 para la mayoría del bloque, 60 para los encabezados de
  sección tipo `- Stack -----...`) para que el borde derecho quede parejo.
  Los encabezados de sección usan em-dashes (`—`, U+2014) de relleno.
- Las filas con `id` son las que `today.py` sobreescribe automáticamente
  (`svg_overwrite` → `justify_format` → `find_and_replace`). El resto
  (OS, Host, Kernel, Languages, Hobbies, Stack, Community, Email,
  LinkedIn...) es texto estático que solo editamos nosotros a mano.

## `justify_format` (today.py) — cómo funciona el padding dinámico

```python
just_len = max(0, length - len(new_text))
```

El número de puntos de relleno se calcula en base a `length` (ancho
objetivo fijo, elegido por nosotros por campo) menos el largo del texto
real. Esto es clave para el campo `age_data` (Uptime): como el texto
("19 years, 11 months, 11 days") cambia de largo cuando el mes o el día
bajan de 2 a 1 dígito, el número de puntos se recalcula solo — no hace
falta lógica especial para eso, ya es dinámico por diseño.

El valor correcto de `length` para `age_data` es **48** (no 50 — error que
cometí la primera vez y dejaba esa fila 2 caracteres más ancha que el resto
del bloque). Con `length=48` la fila de Uptime siempre mide 59 caracteres
totales, igual que las demás filas del bloque, sin importar cuántos dígitos
tengan mes/día.

## Qué se hizo en esta sesión (2026-07-23)

1. **Fix del Action**: `permissions: contents: write` faltaba en
   `build.yaml` (causaba 403 al hacer push desde el Action) — resuelto
   junto con actualizar versiones deprecadas de `actions/cache`.
2. **Bug real del Uptime**: `svg_overwrite()` en `today.py` recibía
   `age_data` como parámetro pero nunca lo escribía en el SVG (le faltaba
   la llamada a `justify_format`). Corregido.
3. **Ajuste fino del padding de `age_data`**: `length` estaba en 50, debía
   ser 48 para no romper la consistencia de ancho (59 chars) con el resto
   del bloque. Corregido.
4. **Contenido del bloque neofetch**:
   - Se quitó la fila "IDE".
   - Se reemplazó "Skills" (placeholder sin llenar) por una sección
     "Stack" (Cloud, Backend, Frontend, Tools).
   - Se agregó una sección "Community" (HackMTY, AWS Student Builders,
     Class of 2029, RamCpp — luego RamCpp se quitó).
   - Se agregó y luego se quitó una fila de HackMTY en "Contact"
     (decisión final: no incluirla).
   - Se corrigieron typos: "Commmits" → "Commits", "Github" → "GitHub".
   - Se recalcularon las coordenadas `y` en cascada cada vez que se
     agregaron/quitaron filas, y el `height` del `<svg>`/`<rect>` (ahora
     `670px`, antes `530px`).
5. El usuario sigue afinando a mano el estilo específico de
   `dark_mode.svg` (p. ej. el padding de puntos de "Lines of Code", texto
   de Hobbies.Hardware) directamente en el IDE — esos ajustes son
   intencionales, no revertirlos sin que lo pida explícitamente.

## Pendientes / próximos pasos

- **Mañana**: agrandar el arte ASCII para que "llene" mejor el espacio.
  Sí se puede estirar el `<svg>` en el eje X (es solo el atributo
  `width`), pero:
  - El bloque de texto (`x="332"`) tendría que correrse más a la derecha.
  - El `font-size` del arte ASCII se puede escalar independiente del resto
    (son `<text>` separados, aunque ambos heredan el `font-size` del
    `<svg>` raíz salvo que se le ponga uno propio al `<text class="ascii">`).
  - Ojo: GitHub renderiza el README en una columna de ~800-830px; si el
    SVG queda mucho más ancho que eso, GitHub lo escala hacia abajo al
    mostrarlo (no se rompe, pero se pierde parte del efecto de "más
    grande").
- La sección "Skills"/"Stack" no tiene pendientes — ya tiene contenido
  real, no placeholder.
- Revisar si vale la pena aplicar el mismo ancho uniforme (59) a las filas
  que quedaron con anchos inconsistentes desde antes de esta sesión
  (Kernel, Languages.Real, Email, LinkedIn en `light_mode.svg` tienen
  anchos distintos entre sí) — quedó fuera de alcance a propósito, no se
  tocó.
