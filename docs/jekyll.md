# Práctica: Creación de blogs con Jekyll y GitHub Pages
 Jekyll es un generador de sitios web estáticos que nos permite crear de forma sencilla blogs, sitios webs personales o webs para proyectos. Los sitios webs generados con Jekyll no usan una base de datos, el contenido del sitio web está escrito en archivos de texto plano en formato Markdown (o Textile) y plantillas Liquid.

Jekyll es el motor de GitHub Pages, un servicio que ofrece GitHub a sus usuarios para que puedan publicar sitios webs estáticos alojados en los repositorios que tienen en GitHub.

## Creación de un contenedor Docker con Jekyll
Si buscamos la imagen oficial de Jekyll en Docker Hub encontraremos el repositorio oficicial en GitHub.

En esta práctica utilizaresmos la imagen por defecto jekyll/jekyll.

## Repositorio de GitHub
Vamos a crear un repositorio en GitHub con nuestro nombre seguido del dominio github.io
![repositorio](images/repositorio.PNG)
Con el repositorio creado, lo clonamos en la máquina con la que estamos trabajando.

Cuando se haya clonado entramos y lanzamos el contenedor de Jekyll que nos creará la infraestructura de nuestro blog.

````
docker run -it --rm -v "$PWD:/srv/jekyll" jekyll/jekyll jekyll new blog
````
Se nos creará un nuevo directorio llamado blog, debemos mover todo el contenido de ese directorio a la raíz del directorio clonado.

Antes de subir nuestra página, aprenderemos a manejar los posts.

Con todos los pasos anteriores hechos, a la hora de crear/editar posts tenemos que hacer lo siguiente:

En la carpeta _posts se almacenan todos los posts de nuestro sitio, si lo desplegamos encontraremos un archivo de ejemplo bajo el nombre de
````
Fecha-de-creacion-welcome-to-jekyll.markdown
````

Los posts se escriben en lenguaje markdown, tienen una estructura que siempre se debe respetar y que podemos encontrar en ese post de ejemplo.

Dentro del archivo encontraremos las secciones correspondientes a el título que le podemos poner, fecha de redacción, y categoría.

A continuación podemos empezar a redactar el contenido de nuestro post utilizando el lenguaje de markdown

## Publicar el blog
Para subir nuestro blog a GitHub y que esté disponible podremos hacer lo siguiente:
````
git add --all

git commit -m "Comentario para el commit"

git push
````
A continuación, cuando todo esté listo podremos acceder a nuestro blog a través del nombre de nuestro repositorio.