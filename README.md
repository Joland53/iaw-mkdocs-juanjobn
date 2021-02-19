# Práctica: Creación de un sitio web estático con MkDocs y GitHub Pages
 MkDocs es un generador de sitios web estáticos que nos permite crear de forma sencilla un sitio web para documentar un proyecto. El contenido del sitio web está escrito en texto plano en formato Markdown y se configura con un único archivo de configuración en formato YAML.

En este ejemplo vamos a necesitar crear un proyecto que tenga la siguiente estructura de archivos.
````
.
└── proyecto
    ├── docs
    │   ├── about.md
    │   └── home.md
    └── mkdocs.yml
````
En primer lugar vamos a crear un directorio para el proyecto.
````
mkdir proyecto
````
Dentro del directorio de nuestro proyecto deberemos crear el archivo de configuración YAML mkdocs.yml.

Por ejemplo, en el siguiente archivo configuración mkdocs.yml estamos indicando el nombre del sitio web que estamos creando (site_name), los enlaces a las páginas que van a aparecer en el menú de navegación (nav) y el theme que vamos a utilizar (theme).

````
site_name: Proyecto Juanjo

nav:
    - Principal: index.md
    - Acerca de: about.md

theme: material
````
A continuación se muestra un ejemplo completo de todas las opciones de configuración del theme Material para el archivo mkdocs.yml.

````
# Project information
site_name: 'Material for MkDocs'
site_description: 'A Material Design theme for MkDocs'
site_author: 'Martin Donath'
site_url: 'https://squidfunk.github.io/mkdocs-material/'

# Repository
repo_name: 'squidfunk/mkdocs-material'
repo_url: 'https://github.com/squidfunk/mkdocs-material'

# Copyright
copyright: 'Copyright &copy; 2016 - 2017 Martin Donath'

# Configuration
theme:
  name: 'material'
  language: 'en'
  palette:
    primary: 'indigo'
    accent: 'indigo'
  font:
    text: 'Roboto'
    code: 'Roboto Mono'

# Customization
extra:
  manifest: 'manifest.webmanifest'
  social:
    - type: 'github'
      link: 'https://github.com/squidfunk'
    - type: 'twitter'
      link: 'https://twitter.com/squidfunk'
    - type: 'linkedin'
      link: 'https://www.linkedin.com/in/squidfunk'

# Google Analytics
google_analytics:
  - 'UA-XXXXXXXX-X'
  - 'auto'

# Extensions
markdown_extensions:
  - admonition
  - codehilite:
      guess_lang: false
  - toc:
      permalink: true
````
Los archivos con el contenido en Markdown los tenemos que crear dentro del directorio docs. En este ejemplo vamos a crear dos archivos index.md y about.md.
````
.
├── docs
│   ├── about.md
│   └── index.md
└── mkdocs.yml
````
Una vez que hemos creado la estructura del proyecto podemos empezar el desarrollo de nuestro sitio iniciando un contenedor Docker con MkDocs y el theme Material.

Nota: Vamos a crear el contenedor desde el directorio principal el proyecto porque necesitamos crear un volumen de tipo bind mount entre nuestra máquina y el contenedor Docker.
````
docker run --rm -it -p 8000:8000 -v "$PWD":/docs squidfunk/mkdocs-material
````
Una vez iniciado el contenedor podemos acceder a la URL http://localhost:8000 desde un navegador web para ver el estado actual del sitio web que estamos creando.

Ahora podemos editar nuestros archivos Markdown y ver cómo se va generando el sitio web de forma inmediata.
También es posible generar directamente el sitio web sin tener que iniciar un servidor local de desarrollo. Para generar el sitio web automáticamente podemos ejecutar el siguiente comando:
````
docker run --rm -it -v "$PWD":/docs squidfunk/mkdocs-material build
````
El comando anterior creará un directorio llamado site donde guarda el sitio web que se ha generado.
Es posible publicar la el sitio web en GitHub Pages con el siguiente comando:
````
docker run --rm -it -v ~/.ssh:/root/.ssh -v "$PWD":/docs squidfunk/mkdocs-material gh-deploy
````
