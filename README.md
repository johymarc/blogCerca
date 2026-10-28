# Home centro de contenido (CMS)

Plantilla de página web para el **home** del centro de contenido y el módulo que
lista los **blogs activados** (nombre, descripción y enlace), obtenidos con HubL.

Estos son assets de **CMS (Design Manager)**, independientes del proyecto de
UI Extension en `src/`. Se despliegan con `hs cms upload`, no con `hs project upload`.

## Estructura

```
cms-hub/
├── templates/
│   └── home.html                      Plantilla de página del home
└── modules/
    └── blog-list.module/              Módulo "Listado de blogs"
        ├── module.html                HubL: recorre los blogs y pinta cada tarjeta
        ├── fields.json                Campo "blogs" (grupo repetible con campo blog)
        ├── meta.json                  Metadatos del módulo
        └── module.css                 Estilos del módulo
```

## Cómo funciona el listado de blogs

HubSpot **no** tiene una función HubL para enumerar todos los blogs del portal.
El patrón correcto: el editor **activa** los blogs desde el módulo (campo `blog`
repetible) y HubL obtiene la metadata de cada uno con `blog_by_id()`:

```hubl
{% for item in module.blogs %}
  {% set b = blog_by_id(item.blog) %}
  {% set blog_url = b.absolute_url|split('?')|first %}  {# ruta limpia, sin ?hsLang= #}
  {{ b.public_title }}   {# nombre #}
  {{ b.description }}     {# descripción (meta description del blog) #}
  {{ blog_url }}          {# enlace a la página del blog #}
{% endfor %}
```

El editor añade/ordena los blogs en el editor de página → "los blogs activados".

## Desplegar a HubSpot

1. Autentica la cuenta (una sola vez):
   ```bash
   hs auth
   ```
2. Sube la carpeta al Design Manager (la raíz queda como `cms-hub`):
   ```bash
   hs cms upload cms-hub cms-hub --account=<nombre-de-cuenta>
   ```
   > `hs upload` ahora es `hs cms upload`. Ejecútalo desde fuera de la carpeta del
   > proyecto (el CLI bloquea el comando dentro de un project).
3. En HubSpot: **Marketing → Website → Website Pages → Create → Website page**,
   elige la plantilla **"Home centro de contenido"**.
4. En el editor, abre el módulo **Listado de blogs** y agrega los blogs
   (Colombia, México). Publica.

> La plantilla referencia el módulo en `/cms-hub/modules/blog-list.module`.
> Si lo subes a otra ruta del Design Manager, ajusta el `path` en `home.html`.

## Verificar en tu portal

La doc de HubL confirma `absolute_url` y `html_title` en el objeto de blog;
`public_title` y `description` provienen del objeto `group` (blog). Tras subirlo,
confirma en una página de prueba que `b.public_title` y `b.description` traen los
valores esperados; si tu portal expone el nombre como `b.html_title`, ajústalo en
`module.html`.
