# Dockerización
***
## Elección de imagen base
***
### Requisitos (Criterios de búsqueda)
- Imágenes oficiales o de comunidades relevantes y/o verificadas ya nos dará ciertas garantías de que lo que lleva es mantenido por un equipo con cierta experiencia y relevancia. (básicamente, nos asegura que la imagen del contenedor no la ha hecho un solo chaval de 16 años como hobby)

- Se priorizará el open source, por lo que si alguna organización que se dedica al software privativo tiene su propia imagen (como IBM), no se tendrá en cuenta. Así como las imágenes de contenedores basados en Windows serán ignorados.

- Debe ser una versión específica y no una latest para asegurarnos de que en dichas versiones todo funciona y que no vayan a sufrir algún fallo debido a algún cambio de alguna dependencia externa. Por lo que se usarán inmutable tags. Además de buscar siempre imágenes estables, descartando las beta o similares.

- Dado que actualmente el proyecto no tiene ninguna dependencia fuerte de funciones exclusivas de versiones concretas de Python, en busca de evitar conflictos o posibles errores, se utilizará la misma versión de Python que la empleada en el equipo de desarrollo. En concreto, Python 3.9.10 o subversiones de 3.9 (3.9, 3.9.7, 3.9.10, etc).

### Requisitos (Criterios de selección) en orden de prioridad
1.  Menor tamaño posible

2.  Tiempos de construcción mínimos 

3.  Menor número de vulnerabilidades posible

***
### Opciones Existentes

Una vez establecidos los criterios de búsqueda, las opciones que encontramos y que analizaremos usando los criterios de selección son las siguientes:

- Imagen SO base (Ubuntu, Debian, CentOS, etc) + instalación manual Python.
- [Imágenes de Bitnami](https://hub.docker.com/r/bitnami/python), en concreto la que se ajusta a mis criterios de búsqueda es **bitnami/python:3.9.10 (174.85 MB)**. Tienen también bitnami/python:3.9.10-debian-10-r16 pero es la misma solo que renombrada pues tienen exactamente los mismos Image Layers (líneas que componen el Dockerfile que muestra Docker Hub). Esta organización no tiene distinción de imágenes según su tamaño por lo que no hay más que elegir.
- [Imágenes de CircleCI](https://hub.docker.com/r/cimg/python), en concreto **cimg/python:3.9.10 (629.73 MB)**. Esta organización tiene otras variantes como 3.9.10-node y 3.9.10-browsers, las cuales quedan descartadas y no entran a esta valoración pues son imágenes que añaden contenido a la 3.9.10 de base, la de 3.9.10-node añade Node.js (innecesario para este proyecto) y la de 3.9.10-browsers añade todo lo mencionado anteriormente (incluido Node.js) y además Java y Selenium, también innecesarios.
- [Imágenes oficiales de Python](https://hub.docker.com/_/python), las cuales se dividen a su vez en las siguientes variantes:
  - **python:3.9.10 (324.21 MB)**
  - **python:3.9.10-bullseye (324.21 MB)**
  - **python:3.9.10-buster (315.23 MB)**
  - **python:3.9.10-slim (43.97 MB)**
  - **python:3.9.10-slim-bullseye (43.97 MB)**
  - **python:3.9.10-slim-buster (41.56 MB)**
  - **python:3.9.10-alpine3.15 (17.96 MB)**
- Se han encontrado también las [imágenes de PyPy](https://hub.docker.com/_/pypy?tab=description), las cuales no sabía lo que eran pero tras una investigación he visto que PyPy es otro intérprete para Python, el cual presume de ser más rápido. Al no haber usado PyPy en el entorno de desarrollo del proyecto, no me he querido arriesgar a usarlo ahora por lo que las imágenes de PyPy han sido descartadas. 

***
### Proceso de selección y estudio

Para el estudio primero me centraré en las que vienen ya hechas, y más tarde valoraré frente a una hecha manualmente.

Una vez vistas todas las opciones, y tras un vistazo a la diferencia entre todas los variantes diferentes de las oficiales de Python, lo primero que haré será descartar las bullseye ya que estas no son más que una especie de pseudo betas, es decir, son las siguientes versiones que van a salir, pero que aún no han salido porque todavía no son stable, así que siguiendo los criterios de búsqueda, las dos bullseye que aparecen en la lista (python:3.9.10-bullseye y python:3.9.10-slim-bullseye) son descartadas. 

A continuación, viendo los tamaños de cada una de las imágenes, siguiendo el criterio del menor tamaño posible podemos descartar varias de ellas, en concreto: la de CicleCI cimg/python:3.9.10 (629.73 MB), así como las siguientes oficiales de python: python:3.9.10 (324.21 MB) y python:3.9.10-buster (315.23 MB) ya que en un primer vistazo y a falta de comparar más criterios, ya vemos alternativas mejores en cuanto a espacio ocupado, siendo este uno de los puntos más importantes en mi criterio de selección.

Como siguiente paso voy a comparar los tiempos de ejecución necesarios para cada una, creando un Dockerfile con las mismas tareas para cada una de ellas. El Dockerfile será el siguiente para todas las imágenes a excepción de la alpine. (El motivo por el que el de alpine usará otro Dockerfile lo explicaré más adelante.) Variando únicamente la línea del FROM XXXXXX/xxxxx:xxxx entre las diferentes alternátivas. (La justificación por la que se usa este Dockerfile también está redactada a continuación)

 ```md
FROM XXXXXX/xxxxx:xxxx

#Creo el grupo y usuario testeo
RUN groupadd testeo && useradd -ms /bin/bash -g testeo testeo

#Cambia al usuario testeo
USER testeo

#Establezco un directorio en el que haré las instalaciones y copiaré el task.yaml
WORKDIR /home/testeo

#Actualizo pip pues es el instalador de paquetes que voy a usar
RUN python -m pip install --upgrade pip

#Instalo el task runner
RUN pip3 install --user pypyr

#Copio el código con las tareas del task runner
COPY task.yaml ./

#Configuro path para que encuentre pypyr
ENV PATH=$PATH:/home/testeo/.local/bin

#Instalo dependencias del proyecto
RUN pypyr task installdeps

#Cambio a un directorio exclusivo para pasar los test
WORKDIR /app/test

ENTRYPOINT ["pypyr","task","test"]

 ```

Sin embargo para la imagen de python:3.9.10-alpine3.15 he necesitado hacer un par de modificaciones, entre ellas, cambiar el comando `groupadd` y `useradd` por `addgroup` y `adduser` respectivamente. Así como la necesidad de instalar gcc pues al ser la versión tan mínima, no lleva gcc y en mi caso es necesario para usar pypyr. Por lo que he tenido que añadir la orden `RUN apk add build-base` según la [Alpine Linux Wiki](https://wiki.alpinelinux.org/wiki/GCC) para instalar gcc. Quedando el Dockerfile de alpine de la siguiente manera:

```md
FROM python:3.9.10-alpine3.15

#Como estoy en alpine, no trae gcc y lo necesito, por tanto
RUN apk add build-base

#Creo el grupo y usuario testeo
RUN addgroup -S testeo && adduser -S testeo -G testeo

#Cambia al usuario testeo
USER testeo

#Establezco un directorio en el que haré las instalaciones y copiaré el task.yaml
WORKDIR /home/testeo

#Actualizo pip pues es el instalador de paquetes que voy a usar
RUN python -m pip install --upgrade pip

#Instalo el task runner
RUN pip3 install --user pypyr

#Copio el código con las tareas del task runner
COPY task.yaml ./

#Configuro path para que encuentre pypyr
ENV PATH=$PATH:/home/testeo/.local/bin

#Instalo dependencias del proyecto
RUN pypyr task installdeps

#Cambio a un directorio exclusivo para pasar los test
WORKDIR /app/test

ENTRYPOINT ["pypyr","task","test"]

```

Una vez construidas correctamente todas las imágenes, he obtenido los siguientes resultados:

| REPOSITORY            |  IMAGE ID     | SIZE  | Building Time | Building Time (Cached) |
|-----------------------|---------------|-------|---------------|------------------------|
| python_bitnami        |  9561c56f4203 | 526MB | 32.5s (14/14) | 0.8s (13/13) |
| python_alpine         |  3473a9b4006b | 268MB | 39.1s (16/16) | 1.4s (15/15) |
| python_slim_buster    |  73fbf2d98f4f | 139MB | 18.2s (14/14) | 0.7s (13/13) |
| python_slim           |  92b29a2b3753 | 147MB | 22.5s (14/14) | 0.8s (13/13) |