# Centro de contenido (theme CMS)

Theme de HubSpot CMS que agrupa varios blogs en un centro de contenido con
navegación en tres niveles: **blog → industria → solución**.

Cada blog representa un segmento (un país, una línea de negocio, un idioma…).
Dentro de cada blog, los artículos se filtran por **industria** y por
**solución** usando tags.

Son assets de **CMS (Design Manager)**. Se suben con `hs cms upload`, no con
`hs project upload`.

En este README:

- `<home>` es el slug de la página home (por defecto `/home-hub`).
- `<blog>` es la ruta de un blog (por ejemplo `/blog-a`).
- `<post>` es el slug de un artículo.
- `<tag>` es el slug de un tag.

---

## Arquitectura

```
  /<home>                  home.html + módulo blog-list
     │                     Muestra los blogs como tarjetas.
     │ elige un blog
     ▼
  /<blog>                  blog-listing.html + módulo article-list
  /<blog>/tag/<tag>        Lista los artículos con filtros de
     │                     Industria y Solución.
     │ abre un artículo
     ▼
  /<blog>/<post>           blog-post.html
                           Muestra el artículo con sus tags limpios.
```

Tres capas:

1. **Contenido:** un blog nativo de HubSpot por segmento. El segmento lo define
   el blog (`group`), así que no hace falta un tag para él.
2. **Taxonomía:** industria y solución son **tags** del blog con una convención
   de prefijo (ver [Convención de tags](#convención-de-tags)).
3. **Presentación:** este theme (plantillas y módulos). Todo se calcula con
   HubL en el servidor: no hay tablas HubDB ni listas fijas que mantener.

### Por qué es dinámico

Los filtros salen de [`blog_tags()`][fn-blog_tags], que devuelve los tags del
blog. Si se crea un tag y se aplica a un post publicado, el filtro aparece
solo; si se borra, desaparece. No hay que tocar código.

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
    └── article-list.module/         Filtros y tarjetas de artículos (listado)
        ├── module.html
        ├── fields.json
        ├── meta.json
        └── module.css
```

`theme.json` hace que las plantillas aparezcan agrupadas bajo el theme
**"Centro de contenido"** en los selectores de página y de blog. Sin él, el
selector del blog filtra por el theme activo y las plantillas no se ven.
Ver [Themes][doc-themes].

---

## Plantillas

### `home.html` — Home centro de contenido

- **Tipo:** `page`. Se usa para la página web `/<home>`.
- **Qué muestra:** un hero con título y bajada editables, y el módulo
  **Listado de blogs**.
- **HubL:**
  - Etiqueta [`rich_text`][doc-standard-tags] para el título y la bajada,
    editables desde el editor de página.
  - Etiqueta [`module`][doc-standard-tags] para insertar `blog-list.module`.
  - `standard_header_includes` y `standard_footer_includes`, obligatorios en
    toda plantilla HTML. Ver [Plantillas][doc-templates].

### `blog-listing.html` — Listado de artículos

- **Tipo:** `blog_listing`. HubSpot la usa para la portada de cada blog
  (`/<blog>`) y también para sus páginas de tag (`/<blog>/tag/<tag>`).
- **Qué muestra:** breadcrumb, título y descripción del blog, y el módulo
  **Listado de artículos**.
- **HubL:** [`group.public_title` y `group.description`][var-blog] para el
  encabezado.
- **Breadcrumb:** `Inicio › {blog}` y, a continuación, los filtros activos
  (`› Industria › Solución`). El módulo los escribe en el elemento
  `<span data-eb-crumb-tail>`. Si se quita ese elemento, el breadcrumb queda
  fijo y el filtro sigue funcionando.

### `blog-post.html` — Post centro de contenido

- **Tipo:** `blog_post`.
- **Qué muestra:** breadcrumb (`Inicio › {blog} › {post}`), los tags limpios
  como pills, título, autor (con avatar si existe), fecha y el cuerpo del post.
- **HubL:**
  - [`content.name`, `content.post_body`, `content.tag_list`,
    `content.blog_post_author` y `content.publish_date_localized`][var-blog].
  - [`blog_tag_url(group.id, slug)`][fn-blog_tag_url], para que cada pill
    enlace a la página de su tag.
  - Filtros [`split`][flt-split] (quita `?hsLang=` de las URLs) y
    [`lower`][flt-lower] (compara prefijos sin importar mayúsculas).
- **Drag and drop:** HubSpot **no permite** `dnd_area` en plantillas
  `blog_post` (falla la validación). El drag and drop de un post viene de
  `{{ content.post_body }}`: ese cuerpo es el lienzo donde el editor arrastra
  módulos. Ver [Drag and drop areas][doc-dnd].

> El enlace **Inicio** de `blog-listing.html` y `blog-post.html` apunta a
> `/home-hub`. Si tu home usa otro slug, cámbialo en esas dos plantillas.

---

## Módulos

### `blog-list.module` — Listado de blogs

Tarjetas con el nombre, la descripción y el enlace de cada blog, para el home.

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

Filtros de industria y solución, y tarjetas de artículos (título, fecha y
autor).

- **Campos:** ninguno. Toma el blog actual de `group`.
- **HubL:**
  - [`blog_tags(group.id, 250)`][fn-blog_tags] arma los chips de filtro.
  - [`blog_recent_posts(group.id, 250)`][fn-blog_recent_posts] trae los posts.
  - Cada tarjeta guarda los slugs de sus tags en `data-tags`.
- **Filtrado:** client-side. Al tocar un chip, el script muestra solo las
  tarjetas que tienen **todos** los tags elegidos (industria **y** solución),
  actualiza el conteo y el breadcrumb, y muestra un estado vacío si no hay
  resultados. Todo el contenido ya está en el HTML.
- **Páginas de tag:** en `/<blog>/tag/<tag>` el script lee el slug de la URL
  ([`request.path`][var-request] no expone un "tag actual", así que se toma de
  `location.pathname`) y preselecciona ese chip. Desde ahí se puede sumar el
  otro filtro o quitarlo. Funciona con slugs con acentos (por ejemplo
  `ind-educación`).
- **`meta.json`:** disponible en plantillas `BLOG_LISTING` y `PAGE`.

---

## Convención de tags

| Grupo     | Slug del tag     | Nombre del tag (cualquiera de los dos) | Se muestra  |
|-----------|------------------|----------------------------------------|-------------|
| Industria | `ind-<nombre>`   | `Ind-<Nombre>` o `<Nombre>`            | `<Nombre>`  |
| Solución  | `sol-<nombre>`   | `Sol-<Nombre>` o `<Nombre>`            | `<Nombre>`  |

Por ejemplo: un tag con slug `ind-retail` y nombre `Retail` aparece como
**Retail** en el filtro de Industria.

- **El grupo se decide por el slug**, que lleva el prefijo en minúscula
  (`ind-` o `sol-`). El nombre visible puede tenerlo o no.
- **La etiqueta visible** es el nombre del tag. Si empieza con `Ind-` o `Sol-`,
  se quita el prefijo; el resto respeta mayúsculas (`Sol-CRM` → "CRM").
- Un tag cuyo slug no empieza con `ind-` ni `sol-` no aparece en los filtros.
- Un tag sin posts publicados en ese blog tampoco aparece.

---

## Decisiones de diseño

- **HubL en lugar de GraphQL.** Los dos leen los tags en vivo, así que ambos
  cumplen el requisito dinámico. HubL gana aquí porque no necesita un archivo
  `.graphql` ni `dataQueryPath`, y usa atributos de post documentados. GraphQL
  convendría si se necesitara la intersección de tags **en el servidor** con
  paginación para miles de posts.
- **Filtro client-side.** Da filtrado instantáneo sin recargar, y el contenido
  sigue en el HTML.
- **Páginas de tag reutilizadas.** No hay rutas nuevas: las páginas de tag que
  HubSpot ya genera sirven como enlaces directos a un filtro.
- **Theme.** Hace que las plantillas aparezcan en los selectores sin depender
  del filtro por theme.

---

## Limitaciones conocidas

- El listado pide hasta **250 posts** por blog. Si un blog crece más, hay que
  paginar o pasar el cruce de tags al servidor.
- En una página de tag, Google ve en el HTML todos los posts del blog; el
  filtro por el tag se aplica al cargar el script.
- Los estilos de botón del theme base pueden pisar el fondo del chip activo.
- Si el portal expone el autor como `full_name` en lugar de `display_name`,
  ajustar `blog-post.html` y `article-list.module/module.html`.

---

## Referencias

- [Funciones HubL][doc-functions]
- [Variables HubL (blog)][var-blog] · [Variables de request][var-request]
- [Filtros HubL][doc-filters]
- [Etiquetas estándar HubL][doc-standard-tags]
- [Plantillas][doc-templates] · [Plantillas de blog][doc-blog-templates]
- [Módulos][doc-modules] · [Configuración de módulos][doc-module-config] ·
  [Campos][doc-fields]
- [Themes][doc-themes]
- [Drag and drop areas][doc-dnd]

[doc-functions]: https://developers.hubspot.com/docs/reference/cms/hubl/functions
[fn-blog_tags]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_tags
[fn-blog_recent_posts]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_recent_posts
[fn-blog_by_id]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_by_id
[fn-blog_tag_url]: https://developers.hubspot.com/docs/reference/cms/hubl/functions#blog_tag_url
[var-blog]: https://developers.hubspot.com/docs/reference/cms/hubl/variables#blog-variables
[var-request]: https://developers.hubspot.com/docs/reference/cms/hubl/variables
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
