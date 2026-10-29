# Centro de contenido (theme CMS)

Theme de HubSpot CMS que agrupa los blogs de cada país (Colombia, México) en un
centro de contenido con navegación por **país → industria → solución**.

Son assets de **CMS (Design Manager)**, independientes del proyecto de UI
Extension que vive en `src/`. Se despliegan con `hs cms upload`, no con
`hs project upload`.

---

## Arquitectura

```
                 /home-hub                         Plantilla: home.html
        ┌──────────────────────────────┐          Módulo:    blog-list
        │  Hero + "Nuestros blogs"     │          Datos:     blog_by_id()
        │  [Colombia]   [México]       │
        └───────┬──────────────┬───────┘
                │              │
          /colombia        /mexico                Plantilla: blog-listing.html
        ┌──────────────────────────────┐          Módulo:    article-list
        │ Inicio › Colombia            │          Datos:     blog_tags()
        │ Industria: [Alimentos] [...] │                     blog_recent_posts()
        │ Solución:  [WMS] [TMS]       │          Filtro:    client-side (JS)
        │ ┌────┐ ┌────┐ ┌────┐         │
        │ │post│ │post│ │post│         │
        └─┴─┬──┴─┴────┴─┴────┴─────────┘
            │
   /colombia/<slug-del-post>                      Plantilla: blog-post.html
        ┌──────────────────────────────┐          Datos:     content.*, group.*
        │ Inicio › Colombia › Post     │                     blog_tag_url()
        │ (Alimentos) (WMS)            │
        │ Título · Autor · Fecha       │
        │ content.post_body            │  ← lienzo drag-and-drop del editor
        └──────────────────────────────┘
```

Tres capas:

1. **Contenido:** un blog nativo de HubSpot por país. El país lo define el blog
   (`group`), así que no se necesita un tag de país.
2. **Taxonomía:** industria y solución son **tags** del blog con una convención
   de prefijo (ver [Convención de tags](#convención-de-tags)).
3. **Presentación:** este theme (plantillas + módulos). Todo se calcula con HubL
   en el servidor; no hay tablas HubDB ni listas fijas que mantener.

### Por qué es dinámico

Los filtros salen de [`blog_tags()`][fn-blog_tags], que devuelve los tags que
existen en el blog. Si se crea un tag nuevo y se aplica a un post publicado, el
filtro aparece solo; si se borra, desaparece. No hay que tocar código.

---

## Estructura

```
cms-hub/
├── theme.json                       Declara la carpeta como theme
├── fields.json                      Campos del theme (vacío)
├── templates/
│   ├── home.html                    templateType: page
│   ├── blog-listing.html            templateType: blog_listing
│   └── blog-post.html               templateType: blog_post
└── modules/
    ├── blog-list.module/            Tarjetas de los blogs (home)
    │   ├── module.html
    │   ├── fields.json
    │   ├── meta.json
    │   └── module.css
    └── article-list.module/         Filtros + tarjetas de artículos (listado)
        ├── module.html
        ├── fields.json
        ├── meta.json
        └── module.css
```

El `theme.json` es lo que hace que las plantillas aparezcan agrupadas bajo el
theme **"Centro de contenido"** en los selectores de página y de blog. Sin él,
el selector del blog filtra por el theme activo y las plantillas no se ven.
Ver [Themes][doc-themes].

---

## Plantillas

### `home.html` — Home centro de contenido

- **Tipo:** `page` (se usa para una página web, con slug `/home-hub`).
- **Qué muestra:** hero con título y bajada editables, y el módulo
  **Listado de blogs**.
- **HubL:**
  - Etiqueta [`rich_text`][doc-standard-tags] para el título y la bajada del
    hero, editables desde el editor de página.
  - Etiqueta [`module`][doc-standard-tags] para insertar `blog-list.module`.
  - `standard_header_includes` / `standard_footer_includes`, obligatorios en
    toda plantilla HTML. Ver [Plantillas][doc-templates].

### `blog-listing.html` — Listado de artículos

- **Tipo:** `blog_listing` (página principal de cada blog: `/colombia`,
  `/mexico`).
- **Qué muestra:** breadcrumb (`Inicio › {blog}`), título y descripción del
  blog, y el módulo **Listado de artículos**.
- **HubL:**
  - [`group.public_title`, `group.description`][var-blog] para el encabezado.
  - El enlace "Inicio" apunta a `/home-hub`.

### `blog-post.html` — Post centro de contenido

- **Tipo:** `blog_post`.
- **Qué muestra:** breadcrumb (`Inicio › {blog} › {post}`), tags limpios como
  pills, título, autor (con avatar si existe), fecha y el cuerpo del post.
- **HubL:**
  - [`content.name`, `content.post_body`, `content.tag_list`,
    `content.blog_post_author`, `content.publish_date_localized`][var-blog].
  - [`blog_tag_url(group.id, slug)`][fn-blog_tag_url] para que cada pill enlace
    a la página de su tag.
  - Filtros [`split`][flt-split] (quita `?hsLang=` de las URLs) y
    [`lower`][flt-lower] (compara prefijos sin importar mayúsculas).
- **Drag and drop:** HubSpot **no permite** `dnd_area` en plantillas
  `blog_post` (falla la validación). El drag and drop de un post viene de
  `{{ content.post_body }}`: ese cuerpo es el lienzo donde el editor arrastra
  módulos. Ver [Drag and drop areas][doc-dnd].

---

## Módulos

### `blog-list.module` — Listado de blogs

Tarjetas con nombre, descripción y enlace de cada blog, para el home.

- **Campos (`fields.json`):** `blogs`, un grupo repetible con un campo de tipo
  `blog`. El editor elige qué blogs mostrar y en qué orden.
  Ver [Campos de módulo][doc-fields].
- **Por qué un campo repetible:** HubL no tiene una función para listar todos
  los blogs del portal. [`blog_by_id()`][fn-blog_by_id] necesita el ID, y el
  campo `blog` lo entrega.
- **HubL:**
  ```hubl
  {% for item in module.blogs %}
    {% set b = blog_by_id(item.blog) %}
    {% set blog_url = b.absolute_url|split('?')|first %}
    {{ b.public_title }} · {{ b.description }} · {{ blog_url }}
  {% endfor %}
  ```
- **`meta.json`:** disponible en plantillas de tipo `PAGE`.
  Ver [Configuración de módulos][doc-module-config].

### `article-list.module` — Listado de artículos

Filtros de industria y solución, y tarjetas de artículos (título, fecha, autor).

- **Campos:** ninguno. Toma el blog actual de `group`.
- **HubL:**
  - [`blog_tags(group.id, 250)`][fn-blog_tags] arma los chips de filtro.
  - [`blog_recent_posts(group.id, 250)`][fn-blog_recent_posts] trae los posts.
  - Cada tarjeta guarda los slugs de sus tags en `data-tags`.
- **Filtrado:** client-side. Al tocar un chip, el script muestra solo las
  tarjetas que tienen **todos** los tags elegidos (industria **y** solución),
  actualiza el conteo y muestra un estado vacío si no hay resultados. Todo el
  contenido ya está en el HTML, así que Google lo indexa.
- **`meta.json`:** disponible en plantillas `BLOG_LISTING` y `PAGE`.

---

## Convención de tags

| Grupo     | Slug del tag      | Nombre del tag (cualquiera de los dos) | Se muestra |
|-----------|-------------------|----------------------------------------|------------|
| Industria | `ind-alimentos`   | `Ind-Alimentos` o `Alimentos`          | Alimentos  |
| Solución  | `sol-wms`         | `Sol-WMS` o `WMS`                      | WMS        |

- **El grupo se decide por el slug**, que siempre lleva el prefijo en
  minúscula (`ind-` / `sol-`). El nombre visible puede tenerlo o no.
- **La etiqueta visible** es el nombre del tag; si el nombre empieza con
  `Ind-` o `Sol-`, se quita el prefijo. Las mayúsculas del resto se respetan
  (`Sol-CRM` → "CRM").
- Un tag cuyo slug no empieza con `ind-` ni `sol-` no aparece en los filtros.

---

## Decisiones de diseño

- **HubL en lugar de GraphQL.** Ambos leen los tags en vivo, así que los dos
  cumplen el requisito dinámico. HubL gana aquí porque no necesita un archivo
  `.graphql` ni `dataQueryPath`, y usa atributos de post documentados. GraphQL
  convendría si se necesitara la intersección de tags **en el servidor** con
  paginación para miles de posts.
- **Filtro client-side.** Da filtrado instantáneo sin recargar y el contenido
  sigue en el HTML para SEO.
- **Theme.** Hace que las plantillas aparezcan en los selectores sin depender
  del filtro por theme.

---

## Desplegar

1. Autenticar la cuenta (una vez):
   ```bash
   hs auth
   ```
2. Subir el theme. Hay que ejecutarlo **fuera** de la carpeta del proyecto (el
   CLI bloquea `hs cms upload` dentro de un project):
   ```bash
   cd ~ && hs cms upload /ruta/al/repo/cms-hub cms-hub --account=<cuenta>
   ```
   Ver [Referencia del CLI][doc-cli].

Los cambios de plantillas y módulos se aplican a **todas** las páginas que los
usan; no hay que actualizar página por página. Solo el contenido que cada
página configuró (textos editables, blogs elegidos) se mantiene intacto.

## Activar en HubSpot

1. **Home:** crear una página web con la plantilla **"Home centro de
   contenido"**, slug `/home-hub`. En el módulo **Listado de blogs**, agregar
   Colombia y México.
2. **Blogs:** en *Settings → Content → Blog → Templates*, para cada blog
   elegir el theme **"Centro de contenido"** y asignar **"Listado de
   artículos"** (listado) y **"Post centro de contenido"** (post).
   Ver [Seleccionar plantillas de blog][kb-blog-templates].

## Caché

HubSpot sirve las páginas desde su CDN con `s-maxage=36000` (hasta 10 horas
por nodo). Si un tag o post nuevo no aparece:

- Comprobar con un parámetro en la URL (`/colombia?v=2`): trae una copia nueva.
- Forzar la regeneración: re-publicar el post (**Actualizar**), guardar la
  configuración del blog o volver a subir la plantilla o el módulo.

No hay un botón público para purgar el CDN.

## Limitaciones conocidas

- El listado pide hasta **250 posts** por blog. Si un blog crece más, hay que
  paginar o pasar el cruce de tags al servidor.
- El chip activo cambia el color del texto, pero los estilos de botón del theme
  base pueden pisar el fondo.
- Si el portal expone el autor como `full_name` en lugar de `display_name`,
  ajustar `blog-post.html` y `article-list.module/module.html`.

---

## Referencias

- [Funciones HubL][doc-functions]
- [Variables HubL (blog)][var-blog]
- [Filtros HubL][doc-filters]
- [Etiquetas estándar HubL][doc-standard-tags]
- [Plantillas][doc-templates] · [Plantillas de blog][doc-blog-templates]
- [Módulos][doc-modules] · [Configuración de módulos][doc-module-config] ·
  [Campos][doc-fields]
- [Themes][doc-themes]
- [Drag and drop areas][doc-dnd]
- [CLI de HubSpot][doc-cli]

[doc-functions]: https://developers.hubspot.com/docs/reference/cms/hubl/functions
[fn-blog_tags]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_tags
[fn-blog_recent_posts]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_recent_posts
[fn-blog_by_id]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_by_id
[fn-blog_tag_url]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_tag_url
[var-blog]: https://developers.hubspot.com/docs/reference/cms/hubl/variables#blog-variables
[doc-filters]: https://developers.hubspot.com/docs/reference/cms/hubl/filters
[flt-split]: https://developers.hubspot.com/docs/reference/cms/hubl/filters#split
[flt-lower]: https://developers.hubspot.com/docs/reference/cms/hubl/filters#lower
[doc-standard-tags]: https://developers.hubspot.com/docs/reference/cms/hubl/tags/standard-tags
[doc-templates]: https://developers.hubspot.com/docs/cms/building-blocks/templates
[doc-blog-templates]: https://developers.hubspot.com/docs/cms/building-blocks/templates/blog
[doc-modules]: https://developers.hubspot.com/docs/cms/building-blocks/modules
[doc-module-config]: https://developers.hubspot.com/docs/reference/cms/modules/configuration
[doc-fields]: https://developers.hubspot.com/docs/reference/cms/fields/module-theme-fields
[doc-themes]: https://developers.hubspot.com/docs/cms/building-blocks/themes
[doc-dnd]: https://developers.hubspot.com/docs/cms/hubl/tags/dnd-areas
[doc-cli]: https://developers.hubspot.com/docs/developer-tooling/local-development/hubspot-cli/reference
[kb-blog-templates]: https://knowledge.hubspot.com/blog/manage-your-blog-template-and-settings#select-your-blog-templates
