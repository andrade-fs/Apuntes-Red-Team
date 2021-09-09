---
description: >-
  En este artículo repasaremos las herramientas más utilizadas para el ámbito
  del Buffer Overflow así como su configuración habitual.
---

# HERRAMIENTAS

{% hint style="info" %}
Artículo copiado de la Fundación Sadosky escrito por Teresa Alberto. 
{% endhint %}

{% embed url="https://fundacion-sadosky.github.io/guia-escritura-exploits/herramientas.html\#secciones-de-un-binario" %}

## COMPILADOR GCC <a id="compilador-gcc"></a>

GCC es el compilador GNU para C por línea de comandos. Compila archivos en C y genera:

1. el archivo ejecutable: `gcc -o stack1 stack1.c` \(sin crear un archivo objeto intermedio\)
2. el archivo objeto intermedio, es decir el programa en assembler: `gcc -S -masm=intel -o stack1.s stack1.c`.
3. Utilizando ciertas Flags podemos eliminar las mitigaciones como hemos visto en el artículo [**Creando un laboratorio sin mitigaciones**](https://ajcruz15.gitbook.io/red-team/pwn/creando-un-laboratorio-sin-mitigaciones).

## OBJDUMP <a id="objdump"></a>

`objdump` es un desensamblador por línea de comandos que genera el código en assembler de un archivo ejecutable. Se usan desensambladores para generar el código en assembler y comprender el funcionamiento de un binario. Es decir, es necesario pasar de los ceros y unos almacenados en memoria a su representación simbólica legible para humanos. Aunque de acuerdo al código del binario del que se parta, el código en assembler resultante puede contener errores.

Un programa simple que hace una suma con el código fuente:

```text

  int suma(int a, int b) {
    return a+b;
  }

  int main() {
    int resultado = suma(1,2);
  }
```

Al compilar con `gcc -o` generamos el ejecutable:

```text
user@abos:~$ gcc -m32 -fno-stack-protector -ggdb -mpreferred-stack-boundary=2 -z execstack -o suma suma.c
```

Si el programa fue compilado con información de debugging \(`gcc -g...`\) sería posible ver el código fuente intercalado con: `objdump -M intel -S suma`

```text
user@abos:~$ objdump -D -M intel suma
```

```text
  080483cb <suma>:
   int suma(int a, int b) {
   80483cb: 55                   push   ebp
   80483cc: 89 e5                mov    ebp,esp
           
   return a+b;
   80483ce: 8b 55 08             mov    edx,DWORD PTR [ebp+0x8]
   80483d1: 8b 45 0c             mov    eax,DWORD PTR [ebp+0xc]
   80483d4: 01 d0                add    eax,edx
         
   }
   80483d6: 5d                   pop    ebp
   80483d7: c3                   ret    
  080483d8 <main>:     
   int main() {
   80483d8: 55                   push   ebp
   80483d9: 89 e5                mov    ebp,esp
   80483db: 83 ec 04             sub    esp,0x4
        
   int resultado = suma(1,2);
   80483de: 6a 02                push   0x2
   80483e0: 6a 01                push   0x1
   80483e2: e8 e4 ff ff ff       call   80483cb <suma>
   80483e7: 83 c4 08             add    esp,0x8
   80483ea: 89 45 fc             mov    DWORD PTR [ebp-0x4],eax
   
   }
   80483ed: c9                   leave  
   80483ee: c3                   ret
```

La primer columna son las direcciones dentro del espacio de direcciones del programa. La segunda muestra el código máquina de cada instrucción y la última el código desensamblado del programa en assembler.

## GDB <a id="debugger-gdb"></a>

`gdb` es un debugger por línea de comandos que permite ejecutar un programa con “puntos de ruptura” o _breakpoints_ para monitorear los contenidos de la memoria y de los registros del procesador en cualquier momento de la ejecución. Permite llevar a cabo el análisis dinámico de un binario para seguir o modificar el flujo de ejecución.

Para debuggear un programa se lo debe compilar con la opción `-ggdb` de debugging.

```text
user@abos:~$ gcc -m32 -fno-stack-protector -ggdb -mpreferred-stack-boundary=2 -z execstack -o programa programa.c
user@abos:~$ gdb programa 
GNU gdb (Debian 7.7.1+dfsg-5) 7.7.1
Copyright (C) 2014 Free Software Foundation, Inc.
...
Reading symbols from secciones...(no debugging symbols found)...done.
>>> break main
Breakpoint 1 at 0x80484c1
>>> run 
...
```

Un recurso muy útil para simplificar la tarea de debugging es usar herramientas como [Pwndbg](https://github.com/pwndbg/pwndbg), [Dashbord GDB](https://github.com/cyrus-and/gdb-dashboard), [Voltron](https://github.com/snare/voltron) u otros similares.  
Otro recurso valioso que condensa las directivas más útiles de `gdb` es [esta hoja de referencias](http://users.ece.utexas.edu/~adnan/gdb-refcard.pdf) y también la guía de [Exploit Database](https://www.exploit-db.com/papers/13205/).

### **Incluir input por entrada estándar**:

Frecuentemente vamos a necesitar debuggear un programa vulnerable que toma un input por entrada estándar \(es decir, por `stdin`\). Por ejemplo el programa [stack 1](https://fundacion-sadosky.github.io/guia-escritura-exploits/buffer-overflow/1-practica.html#stack-1) con `gets(buf)`.  
Generamos un archivo con el input y al debuggearlo con `gdb` es posible procesarlo como un input por stdin de la siguiente manera:

```text
user@u:~$ python input.py > in
user@u:~$ gdb ./stack1
GNU gdb (Debian 7.7.1+dfsg-5) 7.7.1
...
(gdb) r < in
...
```

### **Incluir argumentos**:

Si en cambio queremos debuggear un programa recibe un argumento. Por ejemplo el [abo 3](https://fundacion-sadosky.github.io/guia-escritura-exploits/buffer-overflow/3-practica.html#abo-3) con `strcpy(buf,argc[1])` y `fn(argc[2])` espera dos argumentos:

```text
user@u:~$ gdb ./abo3
GNU gdb (Debian 7.7.1+dfsg-5) 7.7.1
...
(gdb) r "$(./arg1.py)" "$(./arg2.py)"
...
```

### **Desensamblar**:

Al igual que con `objdump`, con `gdb` también es posible desensamblar un binario, intercalando el código fuente y el assembler con la directiva `disas`:

```text
user@u:~$ gdb ./abo1
GNU gdb (Debian 7.7.1+dfsg-5) 7.7.1
...
(gdb) disas /m main 
Dump of assembler code for function main:
5  int main() {
   0x080483d8 <+0>:   push   ebp
   0x080483d9 <+1>:   mov    ebp,esp
   0x080483db <+3>:   sub    esp,0x4

6  int resultado = suma(1,2);
   0x080483de <+6>:    push   0x2
   0x080483e0 <+8>:    push   0x1
   0x080483e2 <+10>:   call   0x80483cb 
   0x080483e7 <+15>:   add    esp,0x8
   0x080483ea <+18>:   mov    DWORD PTR [ebp-0x4],eax

7  }
   0x080483ed <+21>:   leave  
   0x080483ee <+22>:   ret    
 
End of assembler dump.
>>> 
```

### **Simulación de valores**:

Con `gdb` es posible modificar valores de variables durante la ejecución de un programa. Es un recurso muy útil para probar el funcionamiento de un exploit antes de comenzar a construirlo. Por ejemplo, si se quiere modificar el valor de una variable como `cookie` en el Stack 1 \(o de una dirección de retorno ya identificada\), es posible testear previamente el funcionamiento de esta estrategia con `gdb`:

```text
user@u:~$ gdb ./stack1
GNU gdb (Debian 7.7.1+dfsg-5) 7.7.1
...
(gdb) x/wx &cookie
0xbffff684: 0x00000000
(gdb) set {int}0xbffff684 = 0x41424344
(gdb) x/wx 0xbffff684
0xbffff684: 0x41424344
```

## STRACE

Con `strace` es posible **interceptar las llamadas al sistema** que realiza un programa. Por ejemplo, el siguiente programa es posible rastrear como `printf()` redunda en una llamada `write`:

```text
test-syscalls.c

#include 
int main() {
  printf("Hola mundo!\n");
  return 0;
}
```

Y con `strace` podemos ver la llamada a la syscall `write`.

```text
    user@abos:~$ gcc -m32 -o test-syscalls test-syscalls.c
    user@abos:~$ strace ./test-syscalls
    execve("./llamada-sistema", ["./llamada-sistema"], [/* 18 vars */]) = 0
    brk(0)                                  = 0x804a000
    ....
    fstat64(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 2), ...}) = 0
    mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb7fd8000
    
 => write(1, "Hola mundo!\n", 12Hola mundo!
    )           = 12

    exit_group(0)                           = ?
    +++ exited with 0 +++
```

## CHECKSEC

Para verificar las tecnicas de mitigación habilitadas en un binario es de utilidad usar el script [checksec](http://www.trapkit.de/tools/checksec.html)

```text
user@abos:~$  checksec.sh --file programa
RELRO           STACK CANARY      NX             PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX disabled    No PIE          No RPATH   No RUNPATH   programa
```

## SECCIONES DE UN BINARIO <a id="secciones-de-un-binario"></a>

### OBJDUMP <a id="objdump-1"></a>

`objdump` permite ver las diferentes secciones de un archivo ejecutable.

```text
#include 

int main(){
printf("Inspección de secciones.");
}
```

Compilamos:

```text
user@abos:~$ gcc -m32 -o test test.c
```

Desensamblamos con objdump y vemos las secciones.

```text
user@abos:~$ objdump -h test

secciones:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .interp       00000013  08048134  08048134  00000134  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.ABI-tag 00000020  08048148  08048148  00000148  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  08048168  08048168  00000168  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .gnu.hash     00000020  0804818c  0804818c  0000018c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .dynsym       00000050  080481ac  080481ac  000001ac  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynstr       0000004c  080481fc  080481fc  000001fc  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .gnu.version  0000000a  08048248  08048248  00000248  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version_r 00000020  08048254  08048254  00000254  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .rel.dyn      00000008  08048274  08048274  00000274  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rel.plt      00000018  0804827c  0804827c  0000027c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .init         00000023  08048294  08048294  00000294  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 11 .plt          00000040  080482c0  080482c0  000002c0  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .text         000001a2  08048300  08048300  00000300  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .fini         00000014  080484a4  080484a4  000004a4  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .rodata       00000021  080484b8  080484b8  000004b8  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 15 .eh_frame_hdr 0000002c  080484dc  080484dc  000004dc  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 16 .eh_frame     000000bc  08048508  08048508  00000508  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 17 .init_array   00000004  080495c4  080495c4  000005c4  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 18 .fini_array   00000004  080495c8  080495c8  000005c8  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 19 .jcr          00000004  080495cc  080495cc  000005cc  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 20 .dynamic      000000e8  080495d0  080495d0  000005d0  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 21 .got          00000004  080496b8  080496b8  000006b8  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 22 .got.plt      00000018  080496bc  080496bc  000006bc  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 23 .data         00000008  080496d4  080496d4  000006d4  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 24 .bss          00000004  080496dc  080496dc  000006dc  2**0
                  ALLOC
 25 .comment      00000039  00000000  00000000  000006dc  2**0
                  CONTENTS, READONLY
........

```

* Size: tamaño de la sección.
* VMA: dirección de memoria virtual.
* LMA: dirección lógica de memoria.
* File off: offset de la sección desde el principio del archivo.
* Algn: Alineación.
* Y las flags: CONTENTS, ALLOC, LOAD, READONLY, DATA incluyen más información sobre las secciones.

También con `objdump` es posible ver el contenido de cada sección.

```text
user@abos:~$ objdump -s test
......
Contents of section .rodata:
 80484b8 03000000 01000200 496e7370 65636369  ........Inspecci
 80484c8 c36e2064 65207365 6363696f 6e65732e  .n de secciones.
 80484d8 00		
```

En la sección `.rodata` encontramos el string que imprime el programa. Y también podemos averiguar las direcciones de las entradas de la tabla GOT del binario.

```text
user@abos:~$ objdump --dynamic-reloc test
test:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE 
080496b8 R_386_GLOB_DAT    __gmon_start__
080496c8 R_386_JUMP_SLOT   printf
080496cc R_386_JUMP_SLOT   __gmon_start__
080496d0 R_386_JUMP_SLOT   __libc_start_main
```

### READELF <a id="readelf"></a>

También es posible obtener información relacionada a las secciones de un binario con:

```text
user@abos:~$ readelf -S test
There are 30 section headers, starting at offset 0xea0:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        08048134 000134 000013 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            08048148 000148 000020 00   A  0   0  4
  [ 3] .note.gnu.build-i NOTE            08048168 000168 000024 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        0804818c 00018c 000020 04   A  5   0  4
  [ 5] .dynsym           DYNSYM          080481ac 0001ac 000050 10   A  6   1  4
  [ 6] .dynstr           STRTAB          080481fc 0001fc 00004c 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          08048248 000248 00000a 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         08048254 000254 000020 00   A  6   1  4
  [ 9] .rel.dyn          REL             08048274 000274 000008 08   A  5   0  4
  [10] .rel.plt          REL             0804827c 00027c 000018 08  AI  5  12  4
  [11] .init             PROGBITS        08048294 000294 000023 00  AX  0   0  4
  [12] .plt              PROGBITS        080482c0 0002c0 000040 04  AX  0   0 16
  [13] .text             PROGBITS        08048300 000300 0001a2 00  AX  0   0 16
  [14] .fini             PROGBITS        080484a4 0004a4 000014 00  AX  0   0  4
  [15] .rodata           PROGBITS        080484b8 0004b8 000021 00   A  0   0  4
  [16] .eh_frame_hdr     PROGBITS        080484dc 0004dc 00002c 00   A  0   0  4
  [17] .eh_frame         PROGBITS        08048508 000508 0000bc 00   A  0   0  4
  [18] .init_array       INIT_ARRAY      080495c4 0005c4 000004 00  WA  0   0  4
  [19] .fini_array       FINI_ARRAY      080495c8 0005c8 000004 00  WA  0   0  4
  [20] .jcr              PROGBITS        080495cc 0005cc 000004 00  WA  0   0  4
  [21] .dynamic          DYNAMIC         080495d0 0005d0 0000e8 08  WA  6   0  4
  [22] .got              PROGBITS        080496b8 0006b8 000004 04  WA  0   0  4
  [23] .got.plt          PROGBITS        080496bc 0006bc 000018 04  WA  0   0  4
  [24] .data             PROGBITS        080496d4 0006d4 000008 00  WA  0   0  4
  [25] .bss              NOBITS          080496dc 0006dc 000004 00  WA  0   0  1
  [26] .comment          PROGBITS        00000000 0006dc 000039 01  MS  0   0  1
  [27] .shstrtab         STRTAB          00000000 000715 000106 00      0   0  1
  [28] .symtab           SYMTAB          00000000 00081c 000430 10     29  45  4
  [29] .strtab           STRTAB          00000000 000c4c 000254 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

## OTRAS HERRAMIENTAS

### FILE <a id="file"></a>

Imprime el tipo de archivo del que se trata.

### STRINGS <a id="strings"></a>

GNU strings imprime las secuencias de más de 4 caracteres imprimibles que encuentra dentro de un binario.

### HEXDUMP <a id="hexdump"></a>

Muestra el contenido de un binario en hexadecimal.

### NM <a id="nm"></a>

Lista los símbolos de un archivo objeto.

