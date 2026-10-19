# Informe de resolución de laboratorio — Práctica 01: FirstHacking

**Alumno:** Martín Fraga Louzán
**Módulo:** Hacking Ético

---

## 1. Identificación y objetivo

Esta práctica consiste en desplegar FirstHacking, un laboratorio vulnerable preparado para el aprendizaje de ciberseguridad, identificar el servicio que ofrece, investigar una posible vulnerabilidad y realizar una comprobación controlada que distinga una sospecha de un problema demostrado mediante evidencias.

El objetivo es cargar la imagen desde un archivo TAR, crear e identificar el contenedor, reconocer su servicio y versión con Nmap, investigar la vulnerabilidad asociada, comprobarla de forma controlada, explicar el impacto observado y proponer una corrección, documentándolo todo con sus evidencias.

## 2. Entorno y alcance

El trabajo se realiza en un equipo Windows que ejecuta Debian sobre WSL 2. Desde Debian se administra Docker Engine y se utiliza Nmap. El laboratorio se despliega como un contenedor conectado a una red interna dedicada (`practica01-red`).

- **Windows:** acceso al entorno de trabajo.
- **Debian en WSL 2:** ejecución de Docker y Nmap y almacenamiento de los archivos.
- **Docker Engine:** creación y gestión del contenedor.
- **Nmap:** identificación de servicios y comprobación controlada.

**Alcance:** el único objetivo autorizado de las pruebas es la IP del contenedor de esta práctica, guardada en la variable `IP_OBJETIVO` (**172.18.0.2**). No se analizan otros equipos. Las pruebas terminan al comprobar la ejecución de los comandos de identificación previstos (`id` y `hostname`); no se instalan herramientas dentro del objetivo, no se extraen datos ni se intenta acceder a otros sistemas.

## 3. Procedimiento realizado

1. **Preparación del entorno:** se comprobó que `systemd` es el gestor de servicios y se instaló Docker Engine desde el repositorio oficial de Docker para Debian. Se verificó el funcionamiento con `hello-world`.
2. **Carpetas de trabajo:** se creó la estructura `origen`, `imagen`, `archivos` y `evidencias/nmap`.
3. **Carga de la imagen:** se cargó `firsthacking.tar` con `docker load`, obteniendo la imagen `firsthacking:latest`.
4. **Contenedor e IP:** se creó la red interna `practica01-red` y el contenedor `practica01-firsthacking`, cuya IP (172.18.0.2) se guardó en `IP_OBJETIVO`.
5. **Identificación de servicios:** con Nmap se detectó el puerto 21/TCP abierto y el servicio `vsftpd 2.3.4`.
6. **Investigación:** se relacionó la versión con la vulnerabilidad CVE-2011-2523 (puerta trasera de vsftpd 2.3.4).
7. **Comprobación controlada:** se ejecutó el script `ftp-vsftpd-backdoor`, que confirmó la vulnerabilidad y ejecutó `id` y `hostname` en el objetivo.
8. **Reproducción:** se empaquetó la imagen de entrega (`laboratorio.tar`) y se verificó arrancando un segundo contenedor (`practica01-comprobacion`).
9. **Cierre:** se detuvieron ambos contenedores, quedando en estado `Exited`.

Los comandos exactos utilizados están recogidos en `archivos/comandos.txt`.

## 4. Comprobaciones y resultados

> Las capturas que respaldan cada respuesta están en `evidencias/capturas/` y los resultados de Nmap en `evidencias/nmap/`.

### Paso 4 — Identificar los servicios

**¿Qué puerto has encontrado abierto?**
El puerto 21/TCP.

**¿Qué servicio parece atenderlo?**
FTP, el servicio de transferencia de archivos (el 21 es su puerto estándar de control).

**¿Qué programa y versión identifica Nmap?**
vsftpd 2.3.4.

**¿Qué información necesitas comprobar antes de afirmar que existe una vulnerabilidad?**
Hemos encontrado abierto el puerto 21/TCP, este es el puerto estándar del canal de control de FTP. Nmap identifica el servicio como vsftpd 2.3.4. Antes de confirmar que existe una vulnerabilidad debemos comprobar dos cosas: que la versión detectada coincide con la del paquete comprometido asociado a CVE-2011-2523 y que la puerta trasera (backdoor) responde realmente, lo cual se verifica mediante el script ftp-vsftpd-backdoor. La versión por sí sola es una sospecha, no una vulnerabilidad demostrada.

### Paso 5 — Investigar y formular la hipótesis

**¿Qué has observado realmente y qué parte sigue siendo una sospecha?**
El puerto 21/TCP está abierto en 172.18.0.2 y Nmap identifica el servicio como vsftpd 2.3.4. Eso es un hecho confirmado por el escaneo. Lo que sigue siendo una sospecha es que ese servicio sea vulnerable: la versión 2.3.4 se asocia a la puerta trasera CVE-2011-2523, pero coincidir en versión no demuestra que esta instalación tenga realmente el backdoor ni que pueda aprovecharse. Hasta ejecutar la comprobación, solo tengo un indicio, no una vulnerabilidad demostrada.

**¿Qué condición necesitas comprobar para confirmar el problema?**
Para confirmar el problema necesito comprobar que, a través del servicio FTP, se pueden ejecutar comandos en el objetivo. Eso se confirma con el script ftp-vsftpd-backdoor, comprobando que devuelve la salida de comandos como id y hostname ejecutados dentro del contenedor. Si el script logra ejecutar esos comandos, el problema queda demostrado; si no, la versión por sí sola seguiría siendo únicamente una sospecha.

### Paso 6 — Comprobación controlada

**¿Qué resultado respalda que se han ejecutado comandos dentro del objetivo?**
El resultado guardado en evidencias/nmap/03-validacion.txt. El script ftp-vsftpd-backdoor marca el servicio como vulnerable e incluye en su salida el texto devuelto por los comandos id y hostname ejecutados en el objetivo. Que esos comandos devuelvan información del sistema remoto a través del propio servicio FTP demuestra que se ejecutaron dentro del contenedor mediante la puerta trasera, y no desde mi terminal ni con mis permisos de administración de Docker.

**¿Qué identidad muestra la salida?**
La salida del comando id muestra uid=0(root) gid=0(root) groups=0(root). Esto indica que el comando se ejecutó con identidad root dentro del objetivo, es decir, con los máximos privilegios del contenedor.

**¿Qué nombre de sistema devuelve y coincide con el consultado en Docker?**
El comando hostname ejecutado a través del backdoor devuelve e752b65db81c. La consulta `sudo docker inspect -f '{{.Config.Hostname}}' practica01-firsthacking` devuelve también e752b65db81c. Ambos valores coinciden, lo que confirma que el comando se ejecutó en ese contenedor y no en otro equipo. La consulta a Docker solo sirve para contrastar el nombre; la evidencia del impacto es la obtenida a través del servicio vulnerable.

**¿Qué puedes afirmar con esas evidencias y qué queda sin demostrar?**
Puedo afirmar que vsftpd 2.3.4 contiene la puerta trasera de CVE-2011-2523 y que permite ejecutar comandos como root sin autenticación, según la salida de id y hostname. Queda sin demostrar el alcance real del acceso: no he comprobado persistencia, robo o borrado de datos, modificación de archivos ni movimiento hacia otros sistemas, porque la prueba se limitó a los comandos de identificación previstos y el alcance autorizado era únicamente este contenedor.

### Paso 7 — Explicar el hallazgo y proponer corrección

**Ficha del hallazgo**

| Campo | Contenido |
|---|---|
| Activo afectado | Contenedor `practica01-firsthacking` (IP 172.18.0.2), servicio FTP en el puerto 21. |
| Problema y causa | Puerta trasera de vsftpd 2.3.4 (CVE-2011-2523): la versión procede de una distribución del código comprometida. |
| Evidencia | `evidencias/nmap/03-validacion.txt`: VULNERABLE (Exploitable), id=uid=0(root), hostname=e752b65db81c. |
| Impacto demostrado | Ejecución de comandos como root dentro del contenedor, sin autenticación. |
| Corrección propuesta | Sustituir vsftpd 2.3.4 por una versión mantenida de origen confiable o eliminar el servicio FTP. |
| Verificación posterior | Comprobar que el servicio legítimo responde y que el backdoor ya no se explota. |

**¿Qué parte del impacto has demostrado y qué parte sería una posibilidad todavía no comprobada?**
El impacto demostrado es la ejecución de comandos dentro del contenedor a través del servicio FTP vulnerable, con identidad root, lo que supone el control del sistema del contenedor sin necesidad de autenticación. Queda como posibilidad todavía no comprobada el daño real que ese acceso permitiría: borrar o robar información, instalar software malicioso, mantener acceso persistente tras un reinicio o moverse hacia otros sistemas de la red. Son consecuencias plausibles del mismo fallo, pero no las he probado porque la comprobación se limitó a los comandos de identificación previstos y el alcance autorizado era únicamente este contenedor.

**¿Cómo eliminarías la causa del problema?**
Para eliminar la causa del problema sustituiría vsftpd 2.3.4 por una versión actual y mantenida, obtenida de un repositorio oficial de la distribución, reconstruyendo la imagen del contenedor a partir de una base limpia y verificando la integridad del paquete. Además valoraría si el servicio FTP es realmente necesario: si no lo es, la medida más segura es eliminarlo y cerrar el puerto 21, reduciendo así la superficie de exposición. No basta con detener el servicio temporalmente, porque el binario comprometido seguiría presente en la imagen.

**¿Cómo verificarías la corrección sin confundir un fallo de conexión con una solución efectiva?**
Para verificar la corrección sin confundir un fallo de conexión con una solución efectiva comprobaría, en orden: primero, que el contenedor está en marcha y responde; segundo, si el FTP sigue siendo necesario, que el servicio legítimo atiende en el puerto 21 y que Nmap informa de una versión distinta de la 2.3.4; y tercero, que al volver a lanzar el script ftp-vsftpd-backdoor el resultado es negativo estando el servicio activo. Solo si el servicio responde con normalidad pero la puerta trasera ya no puede explotarse puedo afirmar que la corrección es efectiva. Si en cambio opté por eliminar el servicio, la verificación sería confirmar que el puerto 21 está cerrado de forma intencionada y que ninguna función necesaria depende de él.

### Paso 8 — Reproducción

**¿Qué has comprobado al arrancar el contenedor nuevo y qué comprobaciones siguen pendientes?**
Al arrancar el contenedor nuevo (practica01-comprobacion) a partir de la imagen cargada desde el TAR de entrega, he comprobado que el servicio vuelve a estar disponible: Nmap sobre 172.18.0.2 muestra de nuevo el puerto 21/TCP abierto con el servicio FTP vsftpd 2.3.4. Esto demuestra que la imagen empaquetada se puede recuperar y que reproduce el mismo servicio del laboratorio original en mi equipo. Siguen pendientes de comprobar: que el laboratorio funcione en un ordenador distinto del mío, y la repetición completa de la explotación sobre este contenedor nuevo. La prueba de reproducción confirma únicamente que la imagen arranca y ofrece el mismo servicio, no que se reproduzca todo el proceso de explotación.

## 5. Problemas encontrados y soluciones

- **Confusión al abrir Debian:** al ejecutar `wsl -d Debian` parecía que no se abría nada, pero en realidad WSL abre Debian dentro de la misma ventana de la terminal. Se confirmó comprobando el prompt y `cat /etc/os-release`.
- **Red o contenedor ya existentes:** al rehacer el laboratorio aparecían errores de "network/name already exists". Se resolvió eliminando los contenedores con `docker rm -f` y la red con `docker network rm` antes de volver a crearlos.
- **Imagen en uso al intentar borrarla:** `docker rmi` fallaba porque un contenedor seguía referenciando la imagen. Se solucionó borrando primero los contenedores.
- **Reinstalación del entorno:** se reinstaló Debian desde cero, lo que obligó a reinstalar Docker Engine y a volver a cargar la imagen desde el TAR descargado.
- **Ubicación del TAR:** la carpeta Descargas estaba sincronizada con OneDrive y no se encontraba por la ruta habitual; se resolvió copiando el archivo directamente a la carpeta `origen`/`imagen` desde el Explorador de Windows a través de `\\wsl.localhost\Debian\...`.

## 6. Contenido de la entrega e instrucciones de reproducción

**Estructura de la entrega:**

```
practica-01-firsthacking/
├── README.md                     (este informe)
├── archivos/
│   └── comandos.txt
├── imagen/
│   └── laboratorio.tar
└── evidencias/
    ├── nmap/
    │   ├── 01-puertos.txt
    │   ├── 02-servicio-ftp.txt
    │   ├── 03-validacion.txt
    │   └── 04-reproduccion.txt
    └── capturas/
        ├── image firsthackinglatest.png   (Paso 2)
        ├── IPObjetivo.png                 (Paso 3)
        ├── puerto_servicio_version.png    (Paso 4)
        ├── dockerinspect.png              (Paso 6)
        ├── captura8.png                   (Paso 8)
        ├── laboratiocerrado.png           (Paso 9)
        └── HelloFromDocker.png            (entorno)
```

**Instrucciones de reproducción:**

1. Cargar la imagen: `sudo docker load -i imagen/laboratorio.tar`
2. Crear la red: `sudo docker network create --internal practica01-red`
3. Arrancar el contenedor: `sudo docker run -dit --name practica01-firsthacking --network practica01-red hacketilabs:practica01-firsthacking`
4. Obtener la IP: `IP_OBJETIVO=$(sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' practica01-firsthacking)` y `echo "$IP_OBJETIVO"`
5. Comprobar el servicio: `nmap -sV -p 21 -n -Pn "$IP_OBJETIVO"`
6. Cerrar: `sudo docker stop practica01-firsthacking`

## 7. Conclusiones y fuentes

La práctica demuestra la diferencia entre una sospecha y una vulnerabilidad demostrada. Observar el puerto 21 abierto y la versión vsftpd 2.3.4 era solo un indicio; la vulnerabilidad quedó demostrada únicamente al ejecutar el script `ftp-vsftpd-backdoor`, que logró ejecutar comandos como root dentro del contenedor (CVE-2011-2523). El impacto demostrado es el control del contenedor como root sin autenticación; el daño que ese acceso permitiría (robo de datos, persistencia, pivotaje) es plausible pero no se comprobó, por respetar el alcance. La corrección pasa por sustituir el componente comprometido por software mantenido y de origen confiable, o eliminar el servicio si no es necesario, verificando después que el servicio legítimo funciona y que el backdoor ya no se explota.

