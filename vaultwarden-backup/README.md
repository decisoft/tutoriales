# Copia de seguridad automatizada de Vaultwarden

> *Nota: este tutorial ha sido realizado en una Raspberry Pi 4, pero puede ser llevado a cabo sobre cualquier sistema con Docker instalado (imprescindible) y Portainer, aunque también se puede hacer mediante SSH.* 

## Configurar la copia de seguridad

En esta primera parte, veremos cómo levantar los contenedores de Vaultwarden y de db-backup, para configurar la copia de seguridad: hora, frecuencia y limpieza. Vaultwarden, como otros contenedores con bases de datos, necesitan de una doble copia de seguridad: primero en local de la base de datos, y luego respaldarla con nuestro software de copias de seguridad a cualquier otra ubicación (un disco externo, una nube u otro NAS/Raspberry Pi). 

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

### Comprobar que las copias de seguridad se realizan

Si todo ha ido bien, y ya el contenedor ha hecho su(s) primera(s) copia(s) de seguridad, deberíamos ver las copias de la base de datos en el directorio de Vaultwarden (el que hemos declarado en `volumes`). Para comprobarlo, o bien usamos el explorador de archivos que tengamos en nuestro NAS, o a través de SSH con el comando `ls` dentro de la carpeta de Vaultwarden. En mi caso, vemos que hay 3 copias de seguridad de los últimos días, indicando la fecha y la hora en la que se ha hecho, además de sus firmas MD5. Fijémonos que también está la base de datos que estamos respaldando, la `db.sqlite3` (lo veremos más adelante).

![Captura de pantalla de las diferentes copias de seguridad de Vaultwarden](https://github.com/decisoft/tutoriales/blob/main/vaultwarden-backup/img/copias-seguridad-lista.png?raw=true)

Si no te aparece la copia comprimida en un .gz, comprueba el log del contenedor de db-backup a ver qué error te puede estar dando.

En mi caso, ahora que se están haciendo las copias de seguridad del gestor de contraseñas, respaldo la carpeta de Vaultwarden junto con el resto de mi carpeta de dockers. Por ejemplo, HyperBackup, Duplicati, Borg o cualquier otro software que utilicemos para respaldar nuestros archivos. 

Como mencionamos al principio, es importante esto, porque aunque tengamos varias copias de nuestra base de datos en local, si les pasa algo a los discos, sufrimos un ransomware o se corrompen los archivos, tenemos que tener siempre una copia fuera que podamos recuperar.

### Restaurar la copia de seguridad de Vaultwarden

Sí, todos nuestros peores presagios se han cumplido: un disco duro se ha estropeado, se ha corrompido la información, un ransomware ha cifrado y nos ha dejado sin acceso a los archivos del NAS o una tormenta solar lo ha dejado inutilizado. 

Sea el caso que sea, siempre (y esto aplica a cualquier copia de seguridad) tenemos que haber comprobado con anterioridad que todo ha ido bien, restaurando periódicamente las copias de seguridad. En el caso de las contraseñas, es valiosísimo hacerlo porque nos quedaríamos sin acceder a numerosas cuentas si pensábamos que todo iba bien y, a la hora de la verdad, no ha ido tan así.

#### Crear una nueva carpeta donde situar la restauración de los archivos

Para empezar y que no se nos mezcle el docker de Vaultwarden que estamos usando con el que vamos a restaurar (y la liemos peor), lo mejor es crear una nueva carpeta en nuestro NAS, donde tengamos los dockers, para comprobar esto. También (y así hacemos doble, triple o las necesarias comprobaciones), podemos hacerlo en una máquina virtual de Ubuntu, por ejemplo, o en cualquier otro dispositivo Linux que tengamos cerca (otro NAS o Raspberry Pi, un portátil o sobremesa con Linux como SO, etc). 

Anotamos la ruta donde hemos creado este contenedor porque la tendremos que utilizar más adelante. 

#### Restaurar la carpeta de Vaultwarden con sus copias dentro en este nuevo directorio/máquina virtual

Con nuestro software de copias de seguridad, elegimos restaurar únicamente la carpeta de Vaultwarden (y así evitamos que se pase tiempo descargando otras carpetas innecesarias para el tutorial) y le decimos que lo haga en esta nueva carpeta y no en el directorio original. Saber restaurar nuestras copias es fundamental, así practicaremos y sabremos cómo hacerlo en el caso de que sea necesario. Aquí depende del programa que utilicemos. 

En mi caso, utilizo Duplicati, y lo haría de la siguiente manera:

![Captura de pantalla del paso 1 de restauración en Duplicati](https://raw.githubusercontent.com/decisoft/tutoriales/main/vaultwarden-backup/img/duplicati-01.png)

- Aquí, de toda mi carpeta de Docker que respaldo diariamente en OneDrive, elijo solo la carpeta de vaultwarden.

![Captura de pantalla dos donde configuramos la restauración en Duplicati](https://raw.githubusercontent.com/decisoft/tutoriales/main/vaultwarden-backup/img/duplicati-02.png)

- Aquí elegimos la nueva ruta (en mi caso `/mnt/appdata/vaultwarden`), le decimos que sobreescriba los archivos existentes (no debería haber ninguno) y que restaure los permisos de lectura/escritura.

#### Descomprimir y renombrar la copia de seguridad para que Vaultwarden reconozca la base de datos

Como vimos antes, las copias de seguridad se comprimen y se nombran según el día y la hora en la que se ha hecho, pero Vaultwarden no será capaz de leerlas así. Así que descomprimimos la última copia de seguridad (o la que queramos recuperar), ya sea mediante la interfaz gráfica de nuestro explorador de archivos, o a través de de SSH con el comando `gzip`: `gzip -dk nombre_base_datos.gz`. Por ejemplo, si queremos restaurar a través de SSH la del día 15 de febrero, ponemos lo siguiente: `gzip -dk sqlite3_db_20220215-083549.sqlite3.gz`.

Nos extraerá la base de datos, con el mismo nombre que el comprimido sin el .gz. Pero todavía Vaultwarden no puede reconocerla. Así que de `sqlite3_db_20220215-083549.sqlite3` tendremos que renombrarla a `db.sqlite3`, que como vimos en la captura de archivos anterior, es como Vaultwarden nombra a su base de datos. 

#### Levantar el contenedor restaurado de Vaultwarden en la nueva ruta 

Ahora que ya tenemos nuestra base de datos extraída y renombrada, es hora de levantar un nuevo contenedor de Vaultwarden para comprobar que todo se ha restaurado como debíamos y que nuestras contraseñas están intactas. 

Para ello, desplegamos un nuevo `docker-compose.yml` en Portainer, que contenga la información de este nuevo contenedor. [Tienes el ejemplo aquí](https://github.com/decisoft/tutoriales/blob/main/vaultwarden-backup/docker-compose-restaurar.yml). Básicamente, es igual que el anterior con db-backup, pero sin la información de este último. Por supuesto, tendremos que mapear la ruta donde esté la carpeta en la que hemos restaurado la base de datos, darle un nuevo nombre para el contenedor y ajustar los puertos para que no haya conflictos con el original.

Una vez hecho, le damos a "Deploy the stack" o "Levantar stack" y ya estaría el contenedor funcionando. Si todo ha ido bien, accederemos a este nuevo Vaultwarden y nos logearemos en él como si fuese el original, mismo email y misma contraseña. 