# Práctica 01 — FirstHacking

**Informe de resolución de laboratorio**

**Alumno:** Martín Fraga Louzán
**Módulo:** Hacking Ético

---

## 1. Identificación y objetivo

Esta práctica consiste en desplegar **FirstHacking**, un laboratorio vulnerable preparado para el aprendizaje de ciberseguridad: identificar el servicio que ofrece, investigar una posible vulnerabilidad y realizar una comprobación controlada que distinga una **sospecha** de un **problema demostrado** mediante evidencias.

El objetivo es cargar la imagen desde un archivo TAR, crear e identificar el contenedor, reconocer su servicio y versión con Nmap, investigar la vulnerabilidad asociada, comprobarla de forma controlada, explicar el impacto observado y proponer una corrección, documentándolo todo con sus evidencias.

---

## 2. Entorno y alcance

El trabajo se realiza en un equipo **Windows** que ejecuta **Debian** sobre **WSL 2**. Desde Debian se administra Docker Engine y se utiliza Nmap. El laboratorio se despliega como un contenedor conectado a una red interna dedicada (`practica01-red`).

| Elemento | Función |
|---|---|
| **Windows** | Acceso al entorno de trabajo. |
| **Debian en WSL 2** | Ejecución de Docker y Nmap y almacenamiento de los archivos. |
| **Docker Engine** | Creación y gestión del contenedor. |
| **Nmap** | Identificación de servicios y comprobación controlada. |

### Versiones utilizadas

- **Nmap 7.95** (aparece en los cuatro archivos de `evidencias/nmap/`).
- **Debian GNU/Linux 13 (trixie), versión 13.5.**
- **Docker Engine Community 29.9.1** (cliente y servidor).

> **Nota:** las versiones de Debian (13.5) y Docker (29.9.1) las miré el 22/10, no el día de las pruebas, así que no puedo asegurar al 100% que fueran las mismas el 16/10. La de Nmap (7.95) sí aparece en los propios archivos de esa sesión.

**Fechas de las pruebas:** las comprobaciones con Nmap se realizaron el **16/10/2026**, aproximadamente entre las **20:56 y las 21:01** (según la marca de tiempo de los archivos de `evidencias/nmap/`).

**Alcance:** el único objetivo autorizado de las pruebas es la IP del contenedor de esta práctica, guardada en la variable `IP_OBJETIVO` (**172.18.0.2**). No se analizan otros equipos. Las pruebas terminan al comprobar la ejecución de los comandos de identificación previstos (`id` y `hostname`); no se instalan herramientas dentro del objetivo, no se extraen datos ni se intenta acceder a otros sistemas.

**Regla de parada:** si `IP_OBJETIVO` está vacía, no corresponde al contenedor previsto o aparece un servicio distinto del esperado, me detengo y consulto al profesor antes de continuar.

---

## 3. Procedimiento realizado

1. **Preparación del entorno:** se comprobó que `systemd` es el gestor de servicios y se instaló Docker Engine desde el repositorio oficial de Docker para Debian. Se verificó el funcionamiento con `hello-world`.
2. **Carpetas de trabajo:** se creó la estructura `origen`, `imagen`, `archivos` y `evidencias/nmap`.
3. **Carga de la imagen:** se cargó `firsthacking.tar` con `docker load`, obteniendo la imagen `firsthacking:latest`.
4. **Contenedor e IP:** se creó la red interna `practica01-red` y el contenedor `practica01-firsthacking`, cuya IP (172.18.0.2) se guardó en `IP_OBJETIVO`.
5. **Identificación de servicios:** con Nmap se detectó el puerto 21/TCP abierto y el servicio `vsftpd 2.3.4`.
6. **Investigación:** se relacionó la versión con la vulnerabilidad **CVE-2011-2523** (puerta trasera de vsftpd 2.3.4).
7. **Comprobación controlada:** se ejecutó el script `ftp-vsftpd-backdoor`, que confirmó la vulnerabilidad y ejecutó `id` y `hostname` en el objetivo.
8. **Reproducción:** se etiquetó la imagen de entrega (`hacketilabs:practica01-firsthacking`) y se guardó como `imagen/laboratorio.tar` con `docker save`. Tras cargarla de nuevo con `docker load`, se creó un segundo contenedor (`practica01-comprobacion`) y se comprobó su servicio con Nmap. En `captura8.png` se ve que una **segunda** ejecución del `docker run` con ese mismo nombre devolvió un conflicto, porque el contenedor ya existía de la primera ejecución; la consulta de IP y el escaneo posteriores se hicieron sobre esa instancia ya creada (ver detalle y limitación en el apartado 4, «Paso 8»).
9. **Cierre:** se detuvieron ambos contenedores, quedando en estado `Exited`.

> Los comandos exactos utilizados están recogidos en `archivos/comandos.txt`.

---

## 4. Comprobaciones y resultados

### Paso 4 — Identificar los servicios

**¿Qué puerto has encontrado abierto?**
El puerto **21/TCP**.

**¿Qué servicio parece atenderlo?**
**FTP**, el servicio de transferencia de archivos (el 21 es su puerto estándar de control).

**¿Qué programa y versión identifica Nmap?**
**vsftpd 2.3.4**.

**¿Qué información necesitas comprobar antes de afirmar que existe una vulnerabilidad?**
Hemos encontrado abierto el puerto 21/TCP, este es el puerto estándar del canal de control de FTP. Nmap identifica el servicio como vsftpd 2.3.4. Antes de confirmar que existe una vulnerabilidad debemos comprobar dos cosas: que la versión detectada coincide con la del paquete comprometido asociado a CVE-2011-2523 y que la puerta trasera (backdoor) responde realmente, lo cual se verifica mediante el script `ftp-vsftpd-backdoor`. La versión por sí sola es una sospecha, no una vulnerabilidad demostrada.

> **Evidencias:**
> - `evidencias/capturas/IPObjetivo.png` — obtención de la IP del contenedor (172.18.0.2).
> - `evidencias/capturas/puerto_servicio_version.png`, `evidencias/nmap/01-puertos.txt` y `02-servicio-ftp.txt` — puerto 21/TCP abierto y servicio vsftpd 2.3.4.

### Paso 5 — Investigar y formular la hipótesis

**¿Qué has observado realmente y qué parte sigue siendo una sospecha?**
El puerto 21/TCP está abierto en 172.18.0.2 y Nmap identifica el servicio como vsftpd 2.3.4. Eso es un hecho confirmado por el escaneo. Lo que sigue siendo una sospecha es que ese servicio sea vulnerable: la versión 2.3.4 se asocia a la puerta trasera CVE-2011-2523, pero coincidir en versión no demuestra que esta instalación tenga realmente el backdoor ni que pueda aprovecharse. Hasta ejecutar la comprobación, solo tengo un indicio, no una vulnerabilidad demostrada.

**¿Qué condición necesitas comprobar para confirmar el problema?**
Para confirmar el problema necesito comprobar que, a través del servicio FTP, se pueden ejecutar comandos en el objetivo. Eso se confirma con el script `ftp-vsftpd-backdoor`, comprobando que devuelve la salida de comandos como `id` y `hostname` ejecutados dentro del contenedor. Si el script logra ejecutar esos comandos, el problema queda demostrado; si no, la versión por sí sola seguiría siendo únicamente una sospecha.

### Paso 6 — Comprobación controlada

**¿Qué resultado respalda que se han ejecutado comandos dentro del objetivo?**
El resultado guardado en `evidencias/nmap/03-validacion.txt`. El script `ftp-vsftpd-backdoor` marca el servicio como vulnerable e incluye en su salida el texto devuelto por los comandos `id` y `hostname` ejecutados en el objetivo. Que esos comandos devuelvan información del sistema remoto a través del propio servicio FTP demuestra que se ejecutaron dentro del contenedor mediante la puerta trasera, y no desde mi terminal ni con mis permisos de administración de Docker.

**¿Qué identidad muestra la salida?**
La salida del comando `id` muestra `uid=0(root) gid=0(root) groups=0(root)`. Esto indica que el comando se ejecutó con identidad **root** dentro del objetivo, es decir, con los máximos privilegios del contenedor.

**¿Qué nombre de sistema devuelve y coincide con el consultado en Docker?**
El comando `hostname` ejecutado a través del backdoor devuelve `e752b65db81c`. La consulta `sudo docker inspect -f '{{.Config.Hostname}}' practica01-firsthacking` devuelve también `e752b65db81c`. Ambos valores coinciden, lo que confirma que el comando se ejecutó en ese contenedor y no en otro equipo. La consulta a Docker solo sirve para contrastar el nombre; la evidencia del impacto es la obtenida a través del servicio vulnerable.

**¿Qué puedes afirmar con esas evidencias y qué queda sin demostrar?**
En el servicio de **esta instancia del laboratorio** he comprobado la puerta trasera de CVE-2011-2523, que permite ejecutar comandos como root sin autenticación, según la salida de `id` y `hostname`. No afirmo que toda instalación de vsftpd 2.3.4 tenga el backdoor, solo que esta instancia lo tiene y lo he aprovechado. Queda sin demostrar el alcance real del acceso: no he comprobado persistencia, robo o borrado de datos, modificación de archivos ni movimiento hacia otros sistemas, porque la prueba se limitó a los comandos de identificación previstos y el alcance autorizado era únicamente este contenedor.

> **Evidencias:**
> - `evidencias/capturas/dockerinspect.png` y `evidencias/nmap/03-validacion.txt` — `id` con uid=0(root) y hostname=e752b65db81c obtenidos mediante el script, y la consulta de Docker que devuelve el mismo nombre.

### Paso 7 — Explicar el hallazgo y proponer corrección

**Ficha del hallazgo**

| Campo | Contenido |
|---|---|
| **Activo afectado** | El contenedor `practica01-firsthacking`, con IP 172.18.0.2 y un servidor FTP escuchando en el puerto 21. |
| **Problema y causa** | El FTP lo sirve vsftpd 2.3.4, una versión que llegó a distribuirse con una puerta trasera metida en su código (CVE-2011-2523). Esa puerta trasera es la causa del fallo. |
| **Evidencia** | El script dejó la prueba en `03-validacion.txt`, que recogí en `dockerinspect.png`: marca el servicio como VULNERABLE y devuelve `uid=0(root)` y el hostname `e752b65db81c`. |
| **Impacto demostrado** | Pude ejecutar comandos dentro del contenedor como root y sin meter ninguna contraseña. |
| **Corrección propuesta** | Cambiar esa versión de vsftpd por una actual y de confianza, o quitar el FTP directamente si no hace falta. |
| **Verificación posterior** | Después de corregirlo, comprobar que el servicio normal sigue respondiendo pero que el backdoor ya no se puede usar. |

**¿Qué parte del impacto has demostrado y qué parte sería una posibilidad todavía no comprobada?**
El impacto demostrado es la ejecución de comandos dentro del contenedor a través del servicio FTP vulnerable, con identidad root, lo que supone el control del sistema del contenedor sin necesidad de autenticación. Lo que no he comprobado es el daño real que ese acceso permitiría: borrar o robar información, instalar software malicioso, mantener acceso tras un reinicio o saltar a otras máquinas de la red. Son cosas que ese acceso haría posibles, pero no las probé porque la comprobación se limitó a los comandos de identificación previstos y el alcance autorizado era únicamente este contenedor.

**¿Cómo eliminarías la causa del problema?**
Para eliminar la causa del problema sustituiría vsftpd 2.3.4 por una versión actual y mantenida, obtenida de un repositorio oficial de la distribución, reconstruyendo la imagen del contenedor a partir de una base limpia y verificando la integridad del paquete. Además valoraría si el servicio FTP es realmente necesario: si no lo es, la medida más segura es eliminarlo y cerrar el puerto 21, reduciendo así la superficie de exposición. No basta con detener el servicio temporalmente, porque el binario comprometido seguiría presente en la imagen.

**¿Cómo verificarías la corrección sin confundir un fallo de conexión con una solución efectiva?**
Comprobaría, en orden: primero, que el contenedor está en marcha y responde; segundo, si el FTP sigue siendo necesario, que el servicio legítimo atiende en el puerto 21 y que Nmap informa de una versión distinta de la 2.3.4; y tercero, que al volver a lanzar el script `ftp-vsftpd-backdoor` el resultado es negativo estando el servicio activo. Solo si el servicio responde con normalidad pero la puerta trasera ya no puede explotarse puedo afirmar que la corrección es efectiva. Si en cambio opté por eliminar el servicio, la verificación sería confirmar que el puerto 21 está cerrado de forma intencionada y que ninguna función necesaria depende de él.

### Paso 8 — Reproducción

**¿Qué has comprobado al arrancar el contenedor nuevo y qué comprobaciones siguen pendientes?**

Lo que puedo acreditar con las evidencias es lo siguiente. En `captura8.png`, un intento de crear `practica01-comprobacion` con `docker run` encontró un contenedor que **ya usaba ese nombre** (ID `2b5a812d02d5`), por lo que devolvió un error de conflicto. La consulta posterior de IP obtuvo 172.18.0.2 de esa instancia ya existente, y `evidencias/nmap/04-reproduccion.txt` confirma el servicio FTP vsftpd 2.3.4 sobre ella. Es decir, esta captura documenta el **análisis del servicio** en el contenedor de comprobación, pero no muestra la creación de una instancia nueva ni la carga previa del TAR.

El contenedor `practica01-comprobacion` (`2b5a812d02d5`) se había creado en un intento anterior, dentro de la misma sesión, tras cargar la imagen desde `imagen/laboratorio.tar`. Las órdenes `docker tag`, `docker save` y `docker load` que corresponden a ese paso están recogidas en `archivos/comandos.txt`.

> **Aclaración:** del paso de empaquetar y volver a cargar la imagen no guardé ninguna captura, así que ese paso queda reflejado solo en `comandos.txt` y no en una imagen que lo demuestre por sí sola.

La captura final del cierre identifica el contenedor original como `e752b65db81c` y el de comprobación como `2b5a812d02d5`: son **dos contenedores distintos**, aunque en estas pruebas reutilicen la misma dirección IP (172.18.0.2).

Siguen pendientes de comprobar: que el laboratorio funcione en un ordenador distinto del mío, y la repetición completa de la explotación sobre el contenedor de comprobación. La prueba de reproducción confirma únicamente que la imagen ofrece el mismo servicio, no que se reproduzca todo el proceso de explotación.

> **Evidencias:**
> - `evidencias/capturas/captura8.png` y `evidencias/nmap/04-reproduccion.txt` — conflicto de nombre, IP 172.18.0.2 de la instancia existente y servicio vsftpd 2.3.4.
> - `evidencias/capturas/image firsthackinglatest.png` — carga inicial de la imagen (Paso 2).
> - `evidencias/capturas/laboratiocerrado.png` — estado final de los dos contenedores en `Exited`.

---

## 5. Problemas encontrados y soluciones

- **Confusión al abrir Debian:** al ejecutar `wsl -d Debian` parecía que no se abría nada, pero en realidad WSL abre Debian dentro de la misma ventana de la terminal. Lo confirmé mirando el prompt y con `cat /etc/os-release`.
- **Nombre de contenedor ya en uso (reproducción):** al repetir el `docker run` del paso de reproducción, el nombre `practica01-comprobacion` ya estaba ocupado por el contenedor que había creado en el intento anterior (`2b5a812d02d5`), por lo que Docker devolvió un conflicto. En ese momento no eliminé nada; la consulta de IP y el escaneo se hicieron sobre esa instancia ya existente. Ante un nombre ocupado, lo correcto es identificar primero a qué pertenece el contenedor y, en un entorno real, consultar antes de retirarlo; no es algo que deba resolverse borrando por defecto.
- **Ubicación del TAR:** la carpeta Descargas estaba sincronizada con OneDrive y no la encontraba por la ruta habitual desde Debian; lo resolví copiando el archivo a la carpeta de la práctica desde el Explorador de Windows a través de `\\wsl.localhost\Debian\...`.

---

## 6. Contenido de la entrega e instrucciones de reproducción

### Estructura de la entrega (en GitHub)

```
practica-01-firsthacking/
├── README.md                     (este informe)
├── archivos/
│   └── comandos.txt
└── evidencias/
    ├── nmap/
    │   ├── 01-puertos.txt
    │   ├── 02-servicio-ftp.txt
    │   ├── 03-validacion.txt
    │   └── 04-reproduccion.txt
    └── capturas/
        ├── image firsthackinglatest.png   (Paso 2 — carga de la imagen)
        ├── IPObjetivo.png                 (Paso 3 — IP del contenedor)
        ├── puerto_servicio_version.png    (Paso 4 — puerto y servicio)
        ├── dockerinspect.png              (Paso 6 — validación del backdoor)
        ├── captura8.png                   (Paso 8 — reproducción)
        ├── laboratiocerrado.png           (Paso 9 — cierre)
        └── HelloFromDocker.png            (entorno — prueba de Docker)
```

> El archivo `imagen/laboratorio.tar` **no se incluye** en esta entrega de GitHub por indicación del profesor. Se conserva en mi equipo local, dentro de la carpeta `imagen/` de la práctica. Las instrucciones siguientes describen cómo reproducir el laboratorio localmente a partir de ese TAR.

### Instrucciones de reproducción

> Todas las órdenes se ejecutan desde la carpeta de la práctica: `~/HackEtiLabs/docker/practica-01-firsthacking`.

**1.** Cargar la imagen:
```bash
sudo docker load -i imagen/laboratorio.tar
```
**2.** Antes de crear la red y el contenedor, comprobar si ya existen con esos nombres. Si `practica01-red` o `practica01-firsthacking` ya existen, revisar a qué pertenecen y consultar antes de continuar, en lugar de borrarlos directamente:
```bash
sudo docker network ls
sudo docker ps -a
```
**3.** Crear la red:
```bash
sudo docker network create --internal practica01-red
```
**4.** Arrancar el contenedor:
```bash
sudo docker run -dit --name practica01-firsthacking --network practica01-red hacketilabs:practica01-firsthacking
```
**5.** Obtener la IP:
```bash
IP_OBJETIVO=$(sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' practica01-firsthacking)
echo "$IP_OBJETIVO"
```
**6.** Comprobar el servicio (con protección frente a variable vacía):
```bash
test -n "$IP_OBJETIVO" && nmap -sV -p 21 -n -Pn "$IP_OBJETIVO"
```
**7.** Cerrar el contenedor y comprobar el estado final (debe aparecer en `Exited`):
```bash
sudo docker stop practica01-firsthacking
sudo docker ps -a
```

---

## 7. Conclusiones y fuentes

### Conclusiones

Lo que más me llevo de esta práctica es la diferencia entre **sospechar** que algo es vulnerable y **comprobarlo de verdad**. Al principio solo tenía una pista: el puerto 21 abierto y la versión vsftpd 2.3.4, que es conocida por tener una puerta trasera. Pero eso por sí solo no prueba nada; podría haber una versión parcheada o que el backdoor no respondiera.

La prueba real llegó al lanzar el script `ftp-vsftpd-backdoor`, que consiguió ejecutar comandos dentro del contenedor como usuario **root** y sin pedir contraseña. Ahí dejó de ser una sospecha y pasó a ser un problema confirmado en esta instancia del laboratorio (CVE-2011-2523).

Aun así, quiero dejar claro que solo probé lo justo para confirmar el fallo (los comandos `id` y `hostname`). Con ese acceso se podría hacer mucho más daño —robar o borrar datos, dejar el equipo controlado o saltar a otras máquinas—, pero eso no lo hice porque no entraba en lo que tenía permitido. Para arreglar el problema de raíz habría que cambiar esa versión de vsftpd por una actual y de confianza, o quitar el FTP si no hace falta, y después comprobar que el servicio normal sigue funcionando pero el backdoor ya no se puede usar.

### Fuentes

Durante la ejecución de la práctica, la información sobre CVE-2011-2523 la obtuve directamente de la salida del propio script `ftp-vsftpd-backdoor` de Nmap (archivo `evidencias/nmap/03-validacion.txt`), que identifica la vulnerabilidad e imprime varias referencias. Mientras completaba los pasos no consulté páginas externas.

Para documentar correctamente la fuente, he consultado después la ficha oficial del CVE:

- **MITRE — CVE-2011-2523:** https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523 — confirma que la vulnerabilidad corresponde a la puerta trasera introducida en vsftpd 2.3.4 (descarga comprometida en julio de 2011), que permite obtener una shell de comandos. Coincide con lo que detectó el script en esta práctica.

*Enlaces que el script de Nmap imprimió automáticamente en su salida* (recogidos en `03-validacion.txt`, no abiertos uno a uno): `securityfocus.com/bid/48539`, `scarybeastsecurity.blogspot.com`, `github.com/rapid7/metasploit-framework` y la propia ficha de MITRE.
