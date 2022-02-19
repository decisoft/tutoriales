# Copia de seguridad automatizada de Vaultwarden

> *Nota: este tutorial ha sido realizado en una Raspberry Pi 4, pero puede ser llevado a cabo sobre cualquier sistema con Docker instalado (imprescindible) y Portainer, aunque también se puede hacer mediante SSH.* 

## Configurar la copia de seguridad

En esta primera parte, veremos cómo levantar los contenedores de Vaultwarden y de db-backup, para configurar la copia de seguridad: hora, frecuencia y limpieza.

### Poner en marcha ambos contenedores con docker-compose

Para hacerlo, nos vamos a ayudar del [docker-compose.yml](https://github.com/decisoft/tutoriales/blob/main/vaultwarden-backup/docker-compose.yml) de ejemplo que está en el repositorio. Si ya tienes uno con tu contenedor de vaultwarden, simplemente añade la parte de db-backup, no hace falta que modifiques la anterior. El contenido del `docker-compose.yml` lo podemos pegar en un nuevo stack de Portainer, y configuraremos los dos servicios: vaultwarden y db-backup. *(Si tienes los conocimientos, también puedes descargar el `docker-compose.yml` mediante el acceso SSH a tu NAS/Raspberry Pi y editarlo con `nano`).*

#### Vaultwarden

> *Si ya tienes configurado este contenedor, salta este paso, pasa al de db-backup. Puedes copiar y pegar la parte de db-backup en tu docker-compose.yml de vaultwarden existente, modificar y levantarlos, y así tienes ambos servicios en el mismo sitio*. 

Para poner en marcha Vaultwarden, tenemos que configurar el volumen donde se almacenará la base de datos (`:/data`). En mi caso, es `/mnt/appdata/docker/vaultwarden`. Es en esta carpeta donde se creará la base de datos (`db.sqlite3`), y al que apuntaremos con db-backup para hacerle la copia de seguridad.

Nos aseguramos de que los puertos declarados no tengan ningún conflicto (en mi caso, he tenido que exponer el 88, porque el 80 ya lo tenía en uso).

![Captura de pantalla del docker-compose correspondiente a Vaultwarden en Portainer](https://raw.githubusercontent.com/decisoft/tutoriales/main/vaultwarden-backup/img/portainer-stack-01.png)

> *Se ampliará próximamente el tutorial sobre cómo dar los primeros pasos y cómo importar las contraseñas desde otros servicios*.

#### db-backup

Este es el contenedor que hará un *dump* a la base de datos de Vaultwarden. De esta forma, nos podemos asegurar de que la copia de la base de datos se realiza correctamente. En el caso de las bases de datos, lo usual es que nos dé problemas al restaurar si simplemente copiamos y pegamos como si de otros archivos se tratase, de ahí de la importancia de este contenedor.

Tenemos que cambiar los siguientes parámetros ya que dependen de nuestras preferencias y de la localización de la base de datos:

- **Volumes**: Aquí tenemos dos volúmenes, el `:/backup` y el del `post-script.sh`. 
  - `:/backup`: Aquí montaremos la misma ruta que montamos en vaultwarden, que es donde se almacenará, entre otros archivos de configuración, la base de datos. En mi caso, `/mnt/appdata/docker/vaultwarden`. 
  - `post-script.sh`: Declararemos la misma ruta que en la anterior, pero añadiéndole el `/post-script.sh`. En mi caso: `/mnt/appdata/docker/vaultwarden/post-script.sh`.
- **Variables de entorno o *environment***. La parte jugosa, donde configuraremos a fondo la copia.
  - `DB_TYPE=`. Aquí le decimos al contenedor qué tipo de base de datos tiene que respaldar. En el caso de Vaultwarden, la bbdd por defecto es `sqlite3`, así que lo dejamos tal cual como en el ejemplo.
  - `DB_HOST=`. Aquí señalamos la ruta exacta de la base de datos. Como ya hemos montado el volumen `:/backup`, que es la carpeta que ve el contenedor, trabajamos sobre ella: `:/backup/db.sqlite3`, tal y como está en el ejemplo y como llama Vaultwarden a su base de datos.
  - `DB_NAME=` y`DB_USER=`. Las dejamos tal cual.
  - `DB_DUMP_FREQ=`. Una parte importante: en minutos, la frecuencia con la que el contenedor debe respaldar nuestras contraseñas. `1440` es la frecuencia que le tengo puesta yo, que es una vez al día. Pero puedes poner la que consideres oportuna.
  - `DB_DUMP_BEGIN=`. Con esta variable, le indicamos a qué hora tiene que hacer la copia. Se expresa en HHMM, por lo que si queremos que se haga a las 4 y cuarto de la mañana, tendremos que poner: 0415. Si no tenemos especial preferencia, podemos comentar la línea colocando un # al principio y se hará a la hora a la que se levante el contenedor. 
  - `DB_CLEANUP_TIME=`. Este valor nos permite que el contenedor borre las copias de seguridad una vez que superen el tiempo establecido en minutos. Por ejemplo, si queremos que solo almacene las copias de seguridad de los últimos 7 días, y que las más antiguas se vayan borrando solas, lo fijamos en `8640` minutos, tal y como está puesto en el ejemplo. Lo podemos modificar, siempre en minutos, para adaptarlo a nuestras necesidades.
  - `MD5=` (Firma de la copia), `COMPRESSION=` (tipo de comprimido), y `BACKUP_LOCATION=` (en qué lugar estamos haciendo la copia). Los podemos dejar tal cual salvo que queramos especificarle algún tipo de comprimido especial, pero con estos va bien. 

Ahora que ya hemos terminado configurando las variables del docker de db-backup, podemos levantar ambos contenedores, a través del botón "Deploy the stack" o "Levantar stack" que encontraremos al final de la página. 

![Captura de pantalla del docker-compose correspondiente a Vaultwarden en Portainer](https://raw.githubusercontent.com/decisoft/tutoriales/main/vaultwarden-backup/img/portainer-stack-02.png)