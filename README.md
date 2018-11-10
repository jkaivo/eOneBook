# eOneBook
notes on reverse engineering the Progress Technologies eOneBook

The eOneBook is an ereader with two eink displays, intended for reading
manga with artwork that spans facing pages. The device is described in broad
strokes at https://progresstech.jp/zenkan/index_en.html.

Though only available for purchase at a handful of retailers in Japan,
I was able to get one by going through WEBUY, a company that specializes in
buying and shipping Japanese goods to international buyers. My experience with
WEBUY was quite good. The transaction was very smooth, and shipping was much
faster than anticpated. WEBUY's website is https://www.webuyjapan.com/.

# Official Software

In addition to the device, I bought a copy of Fist of the North Star. I got it
not just to read, but also to study see how difficult it would be to add my
own content to the device. Of particular note is that Fist of the North Star
is the only title currently available in both English and Japanese. My Japanese
is nowhere good enough to read an entire manga series, and I was also
interested in the technical details of the language switching.

Published titles come on an SDHC card. with "DATA PROTECTED" displayed on a
sparkly banner on the label. Fortunately, the card itself simply contains a
FAT32 partition, and "DATA PROTECTION" doesn't prevent examining its contents
on a computer. The device doesn't seem to verify whether the SDHC card in use
actually supports Content Protection, as I was able to copy the contents of
my Fist of the North Star card to a normal 64GB microSDHC card and use it in
the device to read the unecnrypted pages. Encrypted pages appeared scrambled,
so it's possible that the decryption key is derived from the SDHC card in some
manner.

## Disk Layout

The root of the SDHC card contains one file, `.A001`, and two directories,
`C01` and `C02`. The directory structures underneath `C01` and `C02` contain
an identical set of subdirectories, `000` through `019`. As Fist of the North
Star is listed on the eOneBook site as having 18 volumes plus a bonus episode,
it is clear that these directories each represent one volume of the manga.
Each volume contains one file per page of the manga, numbered from `000`. Some
pages are encrypted, which is indicated by suffixing an `E` to the filename.
Volume `000` consists of only two pages, which are the instructions on how
to use the device presented on boot. Examining the files as raw bitmap data
confirms this, as well as indicating that `C01` holds the Japanese version of
the manga, while `C02` holds the English.

The only caveat I have so far encountered is that the FAT32 partition must be
the first partition on the SDHC card. My first attempt at copying the data was
on a card that had the FAT32 partition as the fourth parition, with the first
three partitions blank, which resulted in an error screen on the eOneBook
instructing me (in Japanese, so very roughly paraphrased) to remove the close
the device, insert a different SDHC card, and open the device back up.

### The Executable

It was simple to see that the purpose of the two directories in the root of
disk, but the file `.A001` was somewhat less obvious. Naturally, the first
thing I did was try to identify its type with the POSIX `file` utility:

    $ file .A001                                   
    .A001: ELF 32-bit LSB executable, ARM, version 1

Well, that was easy. After installing a copy of GNU binutils targetting ARM,
I started poking a bit more.

    $ readelf -a .A001
    
    ELF Header:
      Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
      Class:                             ELF32
      Data:                              2's complement, little endian
      Version:                           1 (current)
      OS/ABI:                            UNIX - System V
      ABI Version:                       0
      Type:                              EXEC (Executable file)
      Machine:                           ARM
      Version:                           0x1
      Entry point address:               0x10cac
      Start of program headers:          52 (bytes into file)
      Start of section headers:          102352 (bytes into file)
      Flags:                             0x5000400, Version5 EABI, <unknown>
      Size of this header:               52 (bytes)
      Size of program headers:           32 (bytes)
      Number of program headers:         8
      Size of section headers:           40 (bytes)
      Number of section headers:         38
      Section header string table index: 35
    
    Section Headers:
      [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
      [ 0]                   NULL            00000000 000000 000000 00      0   0  0
      [ 1] .interp           PROGBITS        00010134 000134 000019 00   A  0   0  1
      [ 2] .note.ABI-tag     NOTE            00010150 000150 000020 00   A  0   0  4
      [ 3] .note.gnu.build-i NOTE            00010170 000170 000024 00   A  0   0  4
      [ 4] .gnu.hash         LOOS+ffffff6    00010194 000194 000180 04   A  5   0  4
      [ 5] .dynsym           DYNSYM          00010314 000314 000340 10   A  6   1  4
      [ 6] .dynstr           STRTAB          00010654 000654 0001c7 00   A  0   0  1
      [ 7] .gnu.version      VERSYM          0001081c 00081c 000068 02   A  5   0  2
      [ 8] .gnu.version_r    VERNEED         00010884 000884 000040 00   A  6   2  4
      [ 9] .rel.dyn          REL             000108c4 0008c4 000008 08   A  5   0  4
      [10] .rel.plt          REL             000108cc 0008cc 000180 08  AI  5  22  4
      [11] .init             PROGBITS        00010a4c 000a4c 00000c 00  AX  0   0  4
      [12] .plt              PROGBITS        00010a58 000a58 000254 04  AX  0   0  4
      [13] .text             PROGBITS        00010cac 000cac 009530 00  AX  0   0  4
      [14] .fini             PROGBITS        0001a1dc 00a1dc 000008 00  AX  0   0  4
      [15] .rodata           PROGBITS        0001a1e4 00a1e4 001da9 00   A  0   0  4
      [16] .ARM.exidx        ARM_EXIDX       0001bf90 00bf90 000008 00  AL 13   0  4
      [17] .eh_frame         PROGBITS        0001bf98 00bf98 000004 00   A  0   0  4
      [18] .init_array       INIT_ARRAY      0002c000 00c000 000004 00  WA  0   0  4
      [19] .fini_array       FINI_ARRAY      0002c004 00c004 000004 00  WA  0   0  4
      [20] .jcr              PROGBITS        0002c008 00c008 000004 00  WA  0   0  4
      [21] .dynamic          DYNAMIC         0002c00c 00c00c 0000f0 08  WA  6   0  4
      [22] .got              PROGBITS        0002c0fc 00c0fc 0000d0 04  WA  0   0  4
      [23] .data             PROGBITS        0002c1cc 00c1cc 00091c 00  WA  0   0  4
      [24] .bss              NOBITS          0002cae8 00cae8 0ce764 00  WA  0   0  4
      [25] .comment          PROGBITS        00000000 00cae8 000011 01  MS  0   0  1
      [26] .ARM.attributes   ARM_ATTRIBUTES  00000000 00caf9 00003f 00      0   0  1
      [27] .debug_aranges    PROGBITS        00000000 00cb38 000210 00      0   0  8
      [28] .debug_info       PROGBITS        00000000 00cd48 003f15 00      0   0  1
      [29] .debug_abbrev     PROGBITS        00000000 010c5d 001089 00      0   0  1
      [30] .debug_line       PROGBITS        00000000 011ce6 001d1d 00      0   0  1
      [31] .debug_frame      PROGBITS        00000000 013a04 000d1c 00      0   0  4
      [32] .debug_str        PROGBITS        00000000 014720 001df7 01  MS  0   0  1
      [33] .debug_loc        PROGBITS        00000000 016517 0000e9 00      0   0  1
      [34] .debug_ranges     PROGBITS        00000000 016600 000068 00      0   0  8
      [35] .shstrtab         STRTAB          00000000 018e5f 000170 00      0   0  1
      [36] .symtab           SYMTAB          00000000 016668 0016a0 10     37 212  4
      [37] .strtab           STRTAB          00000000 017d08 001157 00      0   0  1
    Key to Flags:
      W (write), A (alloc), X (execute), M (merge), S (strings)
      I (info), L (link order), G (group), x (unknown)
      O (extra OS processing required) o (OS specific), p (processor specific)
    
    There are no section groups in this file.
    
    Program Headers:
      Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
      EXIDX          0x00bf90 0x0001bf90 0x0001bf90 0x00008 0x00008 R   0x4
      PHDR           0x000034 0x00010034 0x00010034 0x00100 0x00100 R E 0x4
      INTERP         0x000134 0x00010134 0x00010134 0x00019 0x00019 R   0x1
          [Requesting program interpreter: /lib/ld-linux-armhf.so.3]
      LOAD           0x000000 0x00010000 0x00010000 0x0bf9c 0x0bf9c R E 0x10000
      LOAD           0x00c000 0x0002c000 0x0002c000 0x00ae8 0xcf24c RW  0x10000
      DYNAMIC        0x00c00c 0x0002c00c 0x0002c00c 0x000f0 0x000f0 RW  0x4
      NOTE           0x000150 0x00010150 0x00010150 0x00044 0x00044 R   0x4
      GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x10
    
     Section to Segment mapping:
      Segment Sections...
       00     .ARM.exidx 
       01     
       02     .interp 
       03     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .rodata .ARM.exidx .eh_frame 
       04     .init_array .fini_array .jcr .dynamic .got .data .bss 
       05     .dynamic 
       06     .note.ABI-tag .note.gnu.build-id 
       07     
    
    Dynamic section at offset 0xc00c contains 25 entries:
      Tag        Type                         Name/Value
     0x00000001 (NEEDED)                     Shared library: [libpthread.so.0]
     0x00000001 (NEEDED)                     Shared library: [libc.so.6]
     0x0000000c (INIT)                       0x10a4c
     0x0000000d (FINI)                       0x1a1dc
     0x00000019 (INIT_ARRAY)                 0x2c000
     0x0000001b (INIT_ARRAYSZ)               4 (bytes)
     0x0000001a (FINI_ARRAY)                 0x2c004
     0x0000001c (FINI_ARRAYSZ)               4 (bytes)
     0x6ffffef5 (GNU_HASH)                   0x10194
     0x00000005 (STRTAB)                     0x10654
     0x00000006 (SYMTAB)                     0x10314
     0x0000000a (STRSZ)                      455 (bytes)
     0x0000000b (SYMENT)                     16 (bytes)
     0x00000015 (DEBUG)                      0x0
     0x00000003 (PLTGOT)                     0x2c0fc
     0x00000002 (PLTRELSZ)                   384 (bytes)
     0x00000014 (PLTREL)                     REL
     0x00000017 (JMPREL)                     0x108cc
     0x00000011 (REL)                        0x108c4
     0x00000012 (RELSZ)                      8 (bytes)
     0x00000013 (RELENT)                     8 (bytes)
     0x6ffffffe (VERNEED)                    0x10884
     0x6fffffff (VERNEEDNUM)                 2
     0x6ffffff0 (VERSYM)                     0x1081c
     0x00000000 (NULL)                       0x0
    
    Relocation section '.rel.dyn' at offset 0x8c4 contains 1 entries:
     Offset     Info    Type            Sym.Value  Sym. Name
    0002c1c8  00000215 R_ARM_GLOB_DAT    00000000   __gmon_start__
    
    Relocation section '.rel.plt' at offset 0x8cc contains 48 entries:
     Offset     Info    Type            Sym.Value  Sym. Name
    0002c108  00003116 R_ARM_JUMP_SLOT   00000000   pthread_mutex_unlock
    0002c10c  00002916 R_ARM_JUMP_SLOT   00000000   strcmp
    0002c110  00001016 R_ARM_JUMP_SLOT   00000000   printf
    0002c114  00000d16 R_ARM_JUMP_SLOT   00000000   fopen
    0002c118  00001216 R_ARM_JUMP_SLOT   00000000   read
    0002c11c  00001116 R_ARM_JUMP_SLOT   00000000   free
    0002c120  00002d16 R_ARM_JUMP_SLOT   00000000   fgets
    0002c124  00002f16 R_ARM_JUMP_SLOT   00000000   pthread_mutex_lock
    0002c128  00001f16 R_ARM_JUMP_SLOT   00000000   memcpy
    0002c12c  00000816 R_ARM_JUMP_SLOT   00000000   shmget
    0002c130  00000b16 R_ARM_JUMP_SLOT   00000000   pthread_mutex_init
    0002c134  00002216 R_ARM_JUMP_SLOT   00000000   sleep
    0002c138  00001c16 R_ARM_JUMP_SLOT   00000000   shmat
    0002c13c  00001516 R_ARM_JUMP_SLOT   00000000   perror
    0002c140  00001d16 R_ARM_JUMP_SLOT   00000000   __xstat
    0002c144  00003316 R_ARM_JUMP_SLOT   00000000   fwrite
    0002c148  00000c16 R_ARM_JUMP_SLOT   00000000   strcat
    0002c14c  00002316 R_ARM_JUMP_SLOT   00000000   ioctl
    0002c150  00002016 R_ARM_JUMP_SLOT   00000000   usleep
    0002c154  00002416 R_ARM_JUMP_SLOT   00000000   strcpy
    0002c158  00000916 R_ARM_JUMP_SLOT   00000000   gettimeofday
    0002c15c  00001716 R_ARM_JUMP_SLOT   00000000   fread
    0002c160  00000516 R_ARM_JUMP_SLOT   00000000   pthread_create
    0002c164  00003216 R_ARM_JUMP_SLOT   00000000   puts
    0002c168  00000616 R_ARM_JUMP_SLOT   00000000   malloc
    0002c16c  00001416 R_ARM_JUMP_SLOT   00000000   __libc_start_main
    0002c170  00002a16 R_ARM_JUMP_SLOT   00000000   system
    0002c174  00000216 R_ARM_JUMP_SLOT   00000000   __gmon_start__
    0002c178  00000716 R_ARM_JUMP_SLOT   00000000   open
    0002c17c  00001b16 R_ARM_JUMP_SLOT   00000000   strlen
    0002c180  00002616 R_ARM_JUMP_SLOT   00000000   mmap
    0002c184  00001616 R_ARM_JUMP_SLOT   00000000   strchr
    0002c188  00001816 R_ARM_JUMP_SLOT   00000000   fcntl
    0002c18c  00001e16 R_ARM_JUMP_SLOT   00000000   memset
    0002c190  00002116 R_ARM_JUMP_SLOT   00000000   strncpy
    0002c194  00002716 R_ARM_JUMP_SLOT   00000000   write
    0002c198  00000f16 R_ARM_JUMP_SLOT   00000000   shmdt
    0002c19c  00002816 R_ARM_JUMP_SLOT   00000000   fclose
    0002c1a0  00001316 R_ARM_JUMP_SLOT   00000000   munmap
    0002c1a4  00002b16 R_ARM_JUMP_SLOT   00000000   strtok
    0002c1a8  00002c16 R_ARM_JUMP_SLOT   00000000   sprintf
    0002c1ac  00000e16 R_ARM_JUMP_SLOT   00000000   ftok
    0002c1b0  00002e16 R_ARM_JUMP_SLOT   00000000   atoi
    0002c1b4  00003016 R_ARM_JUMP_SLOT   00000000   chmod
    0002c1b8  00000a16 R_ARM_JUMP_SLOT   00000000   fputs
    0002c1bc  00002516 R_ARM_JUMP_SLOT   00000000   strncmp
    0002c1c0  00001916 R_ARM_JUMP_SLOT   00000000   abort
    0002c1c4  00001a16 R_ARM_JUMP_SLOT   00000000   close
    
    There are no unwind sections in this file.
    
    Symbol table '.dynsym' contains 52 entries:
       Num:    Value  Size Type    Bind   Vis      Ndx Name
         0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
         1: 00000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
         2: 00000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
         3: 00000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
         4: 00000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
         5: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_create@GLIBC_2.4 (2)
         6: 00000000     0 FUNC    GLOBAL DEFAULT  UND malloc@GLIBC_2.4 (3)
         7: 00000000     0 FUNC    GLOBAL DEFAULT  UND open@GLIBC_2.4 (2)
         8: 00000000     0 FUNC    GLOBAL DEFAULT  UND shmget@GLIBC_2.4 (3)
         9: 00000000     0 FUNC    GLOBAL DEFAULT  UND gettimeofday@GLIBC_2.4 (3)
        10: 00000000     0 FUNC    GLOBAL DEFAULT  UND fputs@GLIBC_2.4 (3)
        11: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_mutex_init@GLIBC_2.4 (2)
        12: 00000000     0 FUNC    GLOBAL DEFAULT  UND strcat@GLIBC_2.4 (3)
        13: 00000000     0 FUNC    GLOBAL DEFAULT  UND fopen@GLIBC_2.4 (3)
        14: 00000000     0 FUNC    GLOBAL DEFAULT  UND ftok@GLIBC_2.4 (3)
        15: 00000000     0 FUNC    GLOBAL DEFAULT  UND shmdt@GLIBC_2.4 (3)
        16: 00000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.4 (3)
        17: 00000000     0 FUNC    GLOBAL DEFAULT  UND free@GLIBC_2.4 (3)
        18: 00000000     0 FUNC    GLOBAL DEFAULT  UND read@GLIBC_2.4 (2)
        19: 00000000     0 FUNC    GLOBAL DEFAULT  UND munmap@GLIBC_2.4 (3)
        20: 00000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.4 (3)
        21: 00000000     0 FUNC    GLOBAL DEFAULT  UND perror@GLIBC_2.4 (3)
        22: 00000000     0 FUNC    GLOBAL DEFAULT  UND strchr@GLIBC_2.4 (3)
        23: 00000000     0 FUNC    GLOBAL DEFAULT  UND fread@GLIBC_2.4 (3)
        24: 00000000     0 FUNC    GLOBAL DEFAULT  UND fcntl@GLIBC_2.4 (2)
        25: 00000000     0 FUNC    GLOBAL DEFAULT  UND abort@GLIBC_2.4 (3)
        26: 00000000     0 FUNC    GLOBAL DEFAULT  UND close@GLIBC_2.4 (2)
        27: 00000000     0 FUNC    GLOBAL DEFAULT  UND strlen@GLIBC_2.4 (3)
        28: 00000000     0 FUNC    GLOBAL DEFAULT  UND shmat@GLIBC_2.4 (3)
        29: 00000000     0 FUNC    GLOBAL DEFAULT  UND __xstat@GLIBC_2.4 (3)
        30: 00000000     0 FUNC    GLOBAL DEFAULT  UND memset@GLIBC_2.4 (3)
        31: 00000000     0 FUNC    GLOBAL DEFAULT  UND memcpy@GLIBC_2.4 (3)
        32: 00000000     0 FUNC    GLOBAL DEFAULT  UND usleep@GLIBC_2.4 (3)
        33: 00000000     0 FUNC    GLOBAL DEFAULT  UND strncpy@GLIBC_2.4 (3)
        34: 00000000     0 FUNC    GLOBAL DEFAULT  UND sleep@GLIBC_2.4 (3)
        35: 00000000     0 FUNC    GLOBAL DEFAULT  UND ioctl@GLIBC_2.4 (3)
        36: 00000000     0 FUNC    GLOBAL DEFAULT  UND strcpy@GLIBC_2.4 (3)
        37: 00000000     0 FUNC    GLOBAL DEFAULT  UND strncmp@GLIBC_2.4 (3)
        38: 00000000     0 FUNC    GLOBAL DEFAULT  UND mmap@GLIBC_2.4 (3)
        39: 00000000     0 FUNC    GLOBAL DEFAULT  UND write@GLIBC_2.4 (2)
        40: 00000000     0 FUNC    GLOBAL DEFAULT  UND fclose@GLIBC_2.4 (3)
        41: 00000000     0 FUNC    GLOBAL DEFAULT  UND strcmp@GLIBC_2.4 (3)
        42: 00000000     0 FUNC    GLOBAL DEFAULT  UND system@GLIBC_2.4 (3)
        43: 00000000     0 FUNC    GLOBAL DEFAULT  UND strtok@GLIBC_2.4 (3)
        44: 00000000     0 FUNC    GLOBAL DEFAULT  UND sprintf@GLIBC_2.4 (3)
        45: 00000000     0 FUNC    GLOBAL DEFAULT  UND fgets@GLIBC_2.4 (3)
        46: 00000000     0 FUNC    GLOBAL DEFAULT  UND atoi@GLIBC_2.4 (3)
        47: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_mutex_lock@GLIBC_2.4 (2)
        48: 00000000     0 FUNC    GLOBAL DEFAULT  UND chmod@GLIBC_2.4 (3)
        49: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_mutex_unlock@GLIBC_2.4 (2)
        50: 00000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.4 (3)
        51: 00000000     0 FUNC    GLOBAL DEFAULT  UND fwrite@GLIBC_2.4 (3)
    
    Symbol table '.symtab' contains 362 entries:
       Num:    Value  Size Type    Bind   Vis      Ndx Name
         0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
         1: 00010134     0 SECTION LOCAL  DEFAULT    1 
         2: 00010150     0 SECTION LOCAL  DEFAULT    2 
         3: 00010170     0 SECTION LOCAL  DEFAULT    3 
         4: 00010194     0 SECTION LOCAL  DEFAULT    4 
         5: 00010314     0 SECTION LOCAL  DEFAULT    5 
         6: 00010654     0 SECTION LOCAL  DEFAULT    6 
         7: 0001081c     0 SECTION LOCAL  DEFAULT    7 
         8: 00010884     0 SECTION LOCAL  DEFAULT    8 
         9: 000108c4     0 SECTION LOCAL  DEFAULT    9 
        10: 000108cc     0 SECTION LOCAL  DEFAULT   10 
        11: 00010a4c     0 SECTION LOCAL  DEFAULT   11 
        12: 00010a58     0 SECTION LOCAL  DEFAULT   12 
        13: 00010cac     0 SECTION LOCAL  DEFAULT   13 
        14: 0001a1dc     0 SECTION LOCAL  DEFAULT   14 
        15: 0001a1e4     0 SECTION LOCAL  DEFAULT   15 
        16: 0001bf90     0 SECTION LOCAL  DEFAULT   16 
        17: 0001bf98     0 SECTION LOCAL  DEFAULT   17 
        18: 0002c000     0 SECTION LOCAL  DEFAULT   18 
        19: 0002c004     0 SECTION LOCAL  DEFAULT   19 
        20: 0002c008     0 SECTION LOCAL  DEFAULT   20 
        21: 0002c00c     0 SECTION LOCAL  DEFAULT   21 
        22: 0002c0fc     0 SECTION LOCAL  DEFAULT   22 
        23: 0002c1cc     0 SECTION LOCAL  DEFAULT   23 
        24: 0002cae8     0 SECTION LOCAL  DEFAULT   24 
        25: 00000000     0 SECTION LOCAL  DEFAULT   25 
        26: 00000000     0 SECTION LOCAL  DEFAULT   26 
        27: 00000000     0 SECTION LOCAL  DEFAULT   27 
        28: 00000000     0 SECTION LOCAL  DEFAULT   28 
        29: 00000000     0 SECTION LOCAL  DEFAULT   29 
        30: 00000000     0 SECTION LOCAL  DEFAULT   30 
        31: 00000000     0 SECTION LOCAL  DEFAULT   31 
        32: 00000000     0 SECTION LOCAL  DEFAULT   32 
        33: 00000000     0 SECTION LOCAL  DEFAULT   33 
        34: 00000000     0 SECTION LOCAL  DEFAULT   34 
        35: 00000000     0 FILE    LOCAL  DEFAULT  ABS /home/tsumikii/fsl-releas
        36: 00010150     0 NOTYPE  LOCAL  DEFAULT    2 $d
        37: 00000000     0 FILE    LOCAL  DEFAULT  ABS /home/tsumikii/fsl-releas
        38: 00010cac     0 NOTYPE  LOCAL  DEFAULT   13 $a
        39: 0001bf90     0 NOTYPE  LOCAL  DEFAULT   16 $d
        40: 00010cdc     0 NOTYPE  LOCAL  DEFAULT   13 $d
        41: 00000000     0 FILE    LOCAL  DEFAULT  ABS init.c
        42: 0001a1e4     0 NOTYPE  LOCAL  DEFAULT   15 $d
        43: 0002c1cc     0 NOTYPE  LOCAL  DEFAULT   23 $d
        44: 00000000     0 FILE    LOCAL  DEFAULT  ABS /home/tsumikii/fsl-releas
        45: 00010ce8     0 NOTYPE  LOCAL  DEFAULT   13 $a
        46: 00010ce8     0 FUNC    LOCAL  DEFAULT   13 call_weak_fn
        47: 00010d04     0 NOTYPE  LOCAL  DEFAULT   13 $d
        48: 00010a4c     0 NOTYPE  LOCAL  DEFAULT   11 $a
        49: 0001a1dc     0 NOTYPE  LOCAL  DEFAULT   14 $a
        50: 00000000     0 FILE    LOCAL  DEFAULT  ABS /home/tsumikii/fsl-releas
        51: 00010a54     0 NOTYPE  LOCAL  DEFAULT   11 $a
        52: 0001a1e0     0 NOTYPE  LOCAL  DEFAULT   14 $a
        53: 00000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
        54: 0002c008     0 OBJECT  LOCAL  DEFAULT   20 __JCR_LIST__
        55: 00010d0c     0 NOTYPE  LOCAL  DEFAULT   13 $a
        56: 00010d0c     0 FUNC    LOCAL  DEFAULT   13 deregister_tm_clones
        57: 00010d38     0 NOTYPE  LOCAL  DEFAULT   13 $d
        58: 00010d3c     0 NOTYPE  LOCAL  DEFAULT   13 $a
        59: 00010d3c     0 FUNC    LOCAL  DEFAULT   13 register_tm_clones
        60: 0002c1d0     0 NOTYPE  LOCAL  DEFAULT   23 $d
        61: 00010d74     0 FUNC    LOCAL  DEFAULT   13 __do_global_dtors_aux
        62: 0002cae8     1 OBJECT  LOCAL  DEFAULT   24 completed.10375
        63: 0002c004     0 NOTYPE  LOCAL  DEFAULT   19 $d
        64: 0002c004     0 OBJECT  LOCAL  DEFAULT   19 __do_global_dtors_aux_fin
        65: 00010d9c     0 FUNC    LOCAL  DEFAULT   13 frame_dummy
        66: 0002c000     0 NOTYPE  LOCAL  DEFAULT   18 $d
        67: 0002c000     0 OBJECT  LOCAL  DEFAULT   18 __frame_dummy_init_array_
        68: 0002cae8     0 NOTYPE  LOCAL  DEFAULT   24 $d
        69: 00000000     0 FILE    LOCAL  DEFAULT  ABS eOneBook_HOKUTO_MP0.c
        70: 0002cb04     4 OBJECT  LOCAL  DEFAULT   24 _cntPageNext
        71: 0002cb04     0 NOTYPE  LOCAL  DEFAULT   24 $d
        72: 0002cb08     4 OBJECT  LOCAL  DEFAULT   24 _cntPagePrev
        73: 0002cb0c     4 OBJECT  LOCAL  DEFAULT   24 _cntBookNext
        74: 0002cb10     4 OBJECT  LOCAL  DEFAULT   24 _cntBookPrev
        75: 0002cb14     4 OBJECT  LOCAL  DEFAULT   24 _cntChptNext
        76: 0002cb18     4 OBJECT  LOCAL  DEFAULT   24 _cntMultiFunc
        77: 0002cb1c     4 OBJECT  LOCAL  DEFAULT   24 _eventPushedButton
        78: 0001a1e8     0 NOTYPE  LOCAL  DEFAULT   15 $d
        79: 00010dd4     0 NOTYPE  LOCAL  DEFAULT   13 $a
        80: 00011194  1156 FUNC    LOCAL  DEFAULT   13 thrd_countPushButton
        81: 00011618   544 FUNC    LOCAL  DEFAULT   13 getPushCount
        82: 00011838   300 FUNC    LOCAL  DEFAULT   13 clearPushCount
        83: 00011858     0 NOTYPE  LOCAL  DEFAULT   13 $d
        84: 00011874     0 NOTYPE  LOCAL  DEFAULT   13 $a
        85: 00000010     0 NOTYPE  LOCAL  DEFAULT   31 $d
        86: 00000000     0 FILE    LOCAL  DEFAULT  ABS GPIO_control.c
        87: 0001a1f8     0 NOTYPE  LOCAL  DEFAULT   15 $d
        88: 00011a94     0 NOTYPE  LOCAL  DEFAULT   13 $a
        89: 00000124     0 NOTYPE  LOCAL  DEFAULT   31 $d
        90: 00000000     0 FILE    LOCAL  DEFAULT  ABS PRC_control.c
        91: 0002cb20     4 OBJECT  LOCAL  DEFAULT   24 _fdEpd_L
        92: 0002cb20     0 NOTYPE  LOCAL  DEFAULT   24 $d
        93: 0002cb24     4 OBJECT  LOCAL  DEFAULT   24 _fdEpd_R
        94: 0002c1d4     0 NOTYPE  LOCAL  DEFAULT   23 $d
        95: 0002c1d4     4 OBJECT  LOCAL  DEFAULT   23 _displayMode
        96: 0002c1d8     4 OBJECT  LOCAL  DEFAULT   23 _isDisplayR2L
        97: 0002cb28     4 OBJECT  LOCAL  DEFAULT   24 _p_imageBuf
        98: 0002cb2c     4 OBJECT  LOCAL  DEFAULT   24 _p_preImageBuf
        99: 0001baac     0 NOTYPE  LOCAL  DEFAULT   15 $d
       100: 000145b0     0 NOTYPE  LOCAL  DEFAULT   13 $a
       101: 000145b0   316 FUNC    LOCAL  DEFAULT   13 PRC_enabletPanel
       102: 000146ec   184 FUNC    LOCAL  DEFAULT   13 PRC_enableBothPanel
       103: 000147a4   184 FUNC    LOCAL  DEFAULT   13 PRC_disableBothPanel
       104: 0001485c   644 FUNC    LOCAL  DEFAULT   13 PRC_displayImage
       105: 00014f7c   156 FUNC    LOCAL  DEFAULT   13 PRC_isPushedButtonDuringU
       106: 00014ae0   268 FUNC    LOCAL  DEFAULT   13 PRC_displayPartialImage
       107: 00014bec   168 FUNC    LOCAL  DEFAULT   13 PRC_displayShortImage
       108: 00014c94   508 FUNC    LOCAL  DEFAULT   13 PRC_displayShortImageEnd
       109: 00014e90    68 FUNC    LOCAL  DEFAULT   13 PRC_saveFilePath
       110: 00014ed4    36 FUNC    LOCAL  DEFAULT   13 PRC_getDisplayMode
       111: 00014ef8    48 FUNC    LOCAL  DEFAULT   13 PRC_setDisplayMode
       112: 00014f28    36 FUNC    LOCAL  DEFAULT   13 PRC_getIsDisplayR2L
       113: 00014f4c    48 FUNC    LOCAL  DEFAULT   13 PRC_setIsDisplayR2L
       114: 00000278     0 NOTYPE  LOCAL  DEFAULT   31 $d
       115: 00000000     0 FILE    LOCAL  DEFAULT  ABS FLC_control.c
       116: 0002c1dc     0 NOTYPE  LOCAL  DEFAULT   23 $d
       117: 0002c1dc    80 OBJECT  LOCAL  DEFAULT   23 FLC_MAX_PAGE_COUNTS_01
       118: 0002c22c    80 OBJECT  LOCAL  DEFAULT   23 FLC_FIRST_CHAPT_OF_BOOK_0
       119: 0002c27c   992 OBJECT  LOCAL  DEFAULT   23 FLC_PAGE_OF_CHAPT_01
       120: 0002c65c    80 OBJECT  LOCAL  DEFAULT   23 FLC_MAX_PAGE_COUNTS_02
       121: 0002c6ac    80 OBJECT  LOCAL  DEFAULT   23 FLC_FIRST_CHAPT_OF_BOOK_0
       122: 0002c6fc   992 OBJECT  LOCAL  DEFAULT   23 FLC_PAGE_OF_CHAPT_02
       123: 0002cb30     4 OBJECT  LOCAL  DEFAULT   24 _currentPageNo_R
       124: 0002cb30     0 NOTYPE  LOCAL  DEFAULT   24 $d
       125: 0002cadc     4 OBJECT  LOCAL  DEFAULT   23 _currentPageNo_L
       126: 0002cb34     4 OBJECT  LOCAL  DEFAULT   24 _currentBookNo
       127: 0002cb38     4 OBJECT  LOCAL  DEFAULT   24 _currentChapterNo
       128: 0002cb3c     4 OBJECT  LOCAL  DEFAULT   24 _currentLanguage
       129: 0002cb40 0x67200 OBJECT  LOCAL  DEFAULT   24 _filePath01
       130: 00093d40 0x67200 OBJECT  LOCAL  DEFAULT   24 _filePath02
       131: 0001baec     0 NOTYPE  LOCAL  DEFAULT   15 $d
       132: 0001baec    64 OBJECT  LOCAL  DEFAULT   15 _lowBatteryDirPath
       133: 0001bb2c    64 OBJECT  LOCAL  DEFAULT   15 _sleepDirPath
       134: 0001bb6c    64 OBJECT  LOCAL  DEFAULT   15 _sdRemoveDirPath
       135: 000faf40   156 OBJECT  LOCAL  DEFAULT   24 last_path_file_t
       136: 00016230     0 NOTYPE  LOCAL  DEFAULT   13 $a
       137: 000162dc   132 FUNC    LOCAL  DEFAULT   13 FLC_getMaxPageNo
       138: 00016bdc     0 NOTYPE  LOCAL  DEFAULT   13 $d
       139: 00016be0     0 NOTYPE  LOCAL  DEFAULT   13 $a
       140: 00016edc    48 FUNC    LOCAL  DEFAULT   13 FLC_setBookNo
       141: 00016f0c   596 FUNC    LOCAL  DEFAULT   13 FLC_setChapterNo
       142: 00017754   348 FUNC    LOCAL  DEFAULT   13 FLC_getBookNoFromChpt
       143: 00017dd4     0 NOTYPE  LOCAL  DEFAULT   13 $d
       144: 00017dd8     0 NOTYPE  LOCAL  DEFAULT   13 $a
       145: 00017e24     0 NOTYPE  LOCAL  DEFAULT   13 $d
       146: 00017e28     0 NOTYPE  LOCAL  DEFAULT   13 $a
       147: 00017e74     0 NOTYPE  LOCAL  DEFAULT   13 $d
       148: 0000061c     0 NOTYPE  LOCAL  DEFAULT   31 $d
       149: 00000000     0 FILE    LOCAL  DEFAULT  ABS EPDC_control.c
       150: 0001bc40     0 NOTYPE  LOCAL  DEFAULT   15 $d
       151: 00017e78     0 NOTYPE  LOCAL  DEFAULT   13 $a
       152: 000185d4   400 FUNC    LOCAL  DEFAULT   13 EPDC_updatePanel
       153: 00000900     0 NOTYPE  LOCAL  DEFAULT   31 $d
       154: 00000000     0 FILE    LOCAL  DEFAULT  ABS MEM_control.c
       155: 000fafdc    16 OBJECT  LOCAL  DEFAULT   24 _p_image
       156: 000fafdc     0 NOTYPE  LOCAL  DEFAULT   24 $d
       157: 000fafec     4 OBJECT  LOCAL  DEFAULT   24 _screenResolution_X
       158: 000faff0     4 OBJECT  LOCAL  DEFAULT   24 _screenResolution_Y
       159: 0002cae0     0 NOTYPE  LOCAL  DEFAULT   23 $d
       160: 0002cae0     4 OBJECT  LOCAL  DEFAULT   23 _imageResolution_X
       161: 0002cae4     4 OBJECT  LOCAL  DEFAULT   23 _imageResolution_Y
       162: 0001bdbc     0 NOTYPE  LOCAL  DEFAULT   15 $d
       163: 00018764     0 NOTYPE  LOCAL  DEFAULT   13 $a
       164: 000009f0     0 NOTYPE  LOCAL  DEFAULT   31 $d
       165: 00000000     0 FILE    LOCAL  DEFAULT  ABS AESC_control.c
       166: 0001be14     0 NOTYPE  LOCAL  DEFAULT   15 $d
       167: 00018f58     0 NOTYPE  LOCAL  DEFAULT   13 $a
       168: 00019418   696 FUNC    LOCAL  DEFAULT   13 decrypt
       169: 00019180   664 FUNC    LOCAL  DEFAULT   13 encrypt
       170: 00000a84     0 NOTYPE  LOCAL  DEFAULT   31 $d
       171: 00000000     0 FILE    LOCAL  DEFAULT  ABS SDIA_accessor.c
       172: 000faff4    11 OBJECT  LOCAL  DEFAULT   24 _sdSerial
       173: 000faff4     0 NOTYPE  LOCAL  DEFAULT   24 $d
       174: 000196d0     0 NOTYPE  LOCAL  DEFAULT   13 $a
       175: 000196e8   120 FUNC    LOCAL  DEFAULT   13 SDIA_readSerialNumber
       176: 0001bea4     0 NOTYPE  LOCAL  DEFAULT   15 $d
       177: 00000b1c     0 NOTYPE  LOCAL  DEFAULT   31 $d
       178: 00000000     0 FILE    LOCAL  DEFAULT  ABS CCC_control.c
       179: 000fb000    17 OBJECT  LOCAL  DEFAULT   24 _key
       180: 000fb000     0 NOTYPE  LOCAL  DEFAULT   24 $d
       181: 000fb014     4 OBJECT  LOCAL  DEFAULT   24 _content_size
       182: 000fb018     4 OBJECT  LOCAL  DEFAULT   24 _p_imate
       183: 0001978c     0 NOTYPE  LOCAL  DEFAULT   13 $a
       184: 00019ed0   152 FUNC    LOCAL  DEFAULT   13 CCC_getkey
       185: 0001bed4     0 NOTYPE  LOCAL  DEFAULT   15 $d
       186: 00000b88     0 NOTYPE  LOCAL  DEFAULT   31 $d
       187: 00000000     0 FILE    LOCAL  DEFAULT  ABS KEY_control.c
       188: 0001bee8     0 NOTYPE  LOCAL  DEFAULT   15 $d
       189: 00019f68     0 NOTYPE  LOCAL  DEFAULT   13 $a
       190: 00000c94     0 NOTYPE  LOCAL  DEFAULT   31 $d
       191: 00000000     0 FILE    LOCAL  DEFAULT  ABS elf-init.c
       192: 0001a168     0 NOTYPE  LOCAL  DEFAULT   13 $a
       193: 0001a1c0     0 NOTYPE  LOCAL  DEFAULT   13 $d
       194: 0001a1c8     0 NOTYPE  LOCAL  DEFAULT   13 $a
       195: 00000cc8     0 NOTYPE  LOCAL  DEFAULT   31 $d
       196: 00000000     0 FILE    LOCAL  DEFAULT  ABS stat.c
       197: 0001a1cc     0 NOTYPE  LOCAL  DEFAULT   13 $a
       198: 00000d0c     0 NOTYPE  LOCAL  DEFAULT   31 $d
       199: 00000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
       200: 0001bf98     0 NOTYPE  LOCAL  DEFAULT   17 $d
       201: 0001bf98     0 OBJECT  LOCAL  DEFAULT   17 __FRAME_END__
       202: 0002c008     0 NOTYPE  LOCAL  DEFAULT   20 $d
       203: 0002c008     0 OBJECT  LOCAL  DEFAULT   20 __JCR_END__
       204: 00000000     0 FILE    LOCAL  DEFAULT  ABS 
       205: 0002c004     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_end
       206: 0002c00c     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
       207: 0002c000     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_start
       208: 0002c0fc     0 OBJECT  LOCAL  DEFAULT   22 _GLOBAL_OFFSET_TABLE_
       209: 00010a58     0 NOTYPE  LOCAL  DEFAULT   12 $a
       210: 00010a68     0 NOTYPE  LOCAL  DEFAULT   12 $d
       211: 00010a6c     0 NOTYPE  LOCAL  DEFAULT   12 $a
       212: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_mutex_unlock@@GLI
       213: 0001a1c8     4 FUNC    GLOBAL DEFAULT   13 __libc_csu_fini
       214: 000162ac    48 FUNC    GLOBAL DEFAULT   13 FLC_setCurrentLanguage
       215: 000fb15c    68 OBJECT  GLOBAL DEFAULT   24 scrn_inf_fix_t
       216: 0001a1cc    16 FUNC    GLOBAL HIDDEN    13 __stat
       217: 000fb01c   128 OBJECT  GLOBAL DEFAULT   24 _filePathLeft
       218: 000119fc    76 FUNC    GLOBAL DEFAULT   13 getBookNextCnt
       219: 00019b60   476 FUNC    GLOBAL DEFAULT   13 CCC_decryptContent
       220: 00000000     0 FUNC    GLOBAL DEFAULT  UND strcmp@@GLIBC_2.4
       221: 00014344   124 FUNC    GLOBAL DEFAULT   13 GPIO_isPushedPrevPage
       222: 000150e0   260 FUNC    GLOBAL DEFAULT   13 PRC_displayBeforeShutdown
       223: 00000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
       224: 0002c1cc     0 NOTYPE  WEAK   DEFAULT   23 data_start
       225: 000143c0   124 FUNC    GLOBAL DEFAULT   13 GPIO_isPushedNextChapter
       226: 00000000     0 FUNC    GLOBAL DEFAULT  UND printf@@GLIBC_2.4
       227: 0002cae8     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start__
       228: 00000000     0 FUNC    GLOBAL DEFAULT  UND fopen@@GLIBC_2.4
       229: 00018ef4   100 FUNC    GLOBAL DEFAULT   13 MEM_getImageBuf
       230: 00000000     0 FUNC    GLOBAL DEFAULT  UND read@@GLIBC_2.4
       231: 0002caec    24 OBJECT  GLOBAL DEFAULT   24 mutex
       232: 00016230    88 FUNC    GLOBAL DEFAULT   13 FLC_clearBookInfo
       233: 00014534   124 FUNC    GLOBAL DEFAULT   13 GPIO_isPushedPrevBook
       234: 000188f8   120 FUNC    GLOBAL DEFAULT   13 MEM_copyImageBuf
       235: 00000000     0 FUNC    GLOBAL DEFAULT  UND free@@GLIBC_2.4
       236: 00016838   872 FUNC    GLOBAL DEFAULT   13 FLC_openLastPathFile
       237: 00000000     0 FUNC    GLOBAL DEFAULT  UND fgets@@GLIBC_2.4
       238: 0001a1cc    16 FUNC    WEAK   HIDDEN    13 stat
       239: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_mutex_lock@@GLIBC
       240: 00000000     0 FUNC    GLOBAL DEFAULT  UND memcpy@@GLIBC_2.4
       241: 000198ac   260 FUNC    GLOBAL DEFAULT   13 CCC_readCryptedContent
       242: 00017c9c   236 FUNC    GLOBAL DEFAULT   13 FLC_getCurrentBookPath
       243: 000fb24c     0 NOTYPE  GLOBAL DEFAULT   24 _bss_end__
       244: 000119b0    76 FUNC    GLOBAL DEFAULT   13 getPagePrevCnt
       245: 00000000     0 FUNC    GLOBAL DEFAULT  UND shmget@@GLIBC_2.4
       246: 0002cae8     0 NOTYPE  GLOBAL DEFAULT   23 _edata
       247: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_mutex_init@@GLIBC
       248: 00000000     0 FUNC    GLOBAL DEFAULT  UND sleep@@GLIBC_2.4
       249: 00015e78   160 FUNC    GLOBAL DEFAULT   13 PRC_prepareDisplayforPage
       250: 000178b0   408 FUNC    GLOBAL DEFAULT   13 FLC_getCurrentPagePath
       251: 00011a48    76 FUNC    GLOBAL DEFAULT   13 getBookPrevCnt
       252: 0001a1dc     0 FUNC    GLOBAL DEFAULT   14 _fini
       253: 000fb24c     0 NOTYPE  GLOBAL DEFAULT   24 __bss_end__
       254: 000199b0   432 FUNC    GLOBAL DEFAULT   13 CCC_encryptContent
       255: 000fb1a0     4 OBJECT  GLOBAL DEFAULT   24 _fdFb
       256: 00017d88    80 FUNC    GLOBAL DEFAULT   13 FLC_getLowBatteryFilePath
       257: 000fb1a4   160 OBJECT  GLOBAL DEFAULT   24 scrn_inf_var_t
       258: 0001424c   124 FUNC    GLOBAL DEFAULT   13 GPIO_getShutdownReq4LowBa
       259: 000142c8   124 FUNC    GLOBAL DEFAULT   13 GPIO_isPushedNextPage
       260: 000153cc  1844 FUNC    GLOBAL DEFAULT   13 PRC_loadImageBeforeDispla
       261: 000fb09c   128 OBJECT  GLOBAL DEFAULT   24 _filePathRight
       262: 00000000     0 FUNC    GLOBAL DEFAULT  UND shmat@@GLIBC_2.4
       263: 00015b9c   260 FUNC    GLOBAL DEFAULT   13 PRC_loadImageforShortEnd
       264: 000173ec   336 FUNC    GLOBAL DEFAULT   13 FLC_updateChptNext
       265: 00000000     0 FUNC    GLOBAL DEFAULT  UND perror@@GLIBC_2.4
       266: 000152b4   280 FUNC    GLOBAL DEFAULT   13 PRC_displayInitImage
       267: 00000000     0 FUNC    GLOBAL DEFAULT  UND __xstat@@GLIBC_2.4
       268: 00014160   112 FUNC    GLOBAL DEFAULT   13 GPIO_setUpdating
       269: 00019808   104 FUNC    GLOBAL DEFAULT   13 CCC_makeKey
       270: 00018970  1412 FUNC    GLOBAL DEFAULT   13 MEM_updateImageBuf
       271: 00017e28    80 FUNC    GLOBAL DEFAULT   13 FLC_getSdRemoveFilePath
       272: 00000000     0 FUNC    GLOBAL DEFAULT  UND fwrite@@GLIBC_2.4
       273: 00000000     0 FUNC    GLOBAL DEFAULT  UND strcat@@GLIBC_2.4
       274: 00000000     0 FUNC    GLOBAL DEFAULT  UND ioctl@@GLIBC_2.4
       275: 00000000     0 FUNC    GLOBAL DEFAULT  UND usleep@@GLIBC_2.4
       276: 00000000     0 FUNC    GLOBAL DEFAULT  UND strcpy@@GLIBC_2.4
       277: 00000000     0 FUNC    GLOBAL DEFAULT  UND gettimeofday@@GLIBC_2.4
       278: 00000000     0 FUNC    GLOBAL DEFAULT  UND fread@@GLIBC_2.4
       279: 00000000     0 FUNC    GLOBAL DEFAULT  UND pthread_create@@GLIBC_2.4
       280: 00019870    60 FUNC    GLOBAL DEFAULT   13 CCC_deinit
       281: 000160f8   156 FUNC    GLOBAL DEFAULT   13 PRC_prepareDisplayforChap
       282: 0002c1cc     0 NOTYPE  GLOBAL DEFAULT   23 __data_start
       283: 00000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.4
       284: 00000000     0 FUNC    GLOBAL DEFAULT  UND malloc@@GLIBC_2.4
       285: 00019084   252 FUNC    GLOBAL DEFAULT   13 AESC_encrypt
       286: 00018464   128 FUNC    GLOBAL DEFAULT   13 EPDC_execPreDisplayImage
       287: 00000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
       288: 00000000     0 FUNC    GLOBAL DEFAULT  UND system@@GLIBC_2.4
       289: 00016194   156 FUNC    GLOBAL DEFAULT   13 PRC_prepareDisplayforMult
       290: 000172a8   324 FUNC    GLOBAL DEFAULT   13 FLC_updatePagePrev
       291: 00000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
       292: 000183e4   128 FUNC    GLOBAL DEFAULT   13 EPDC_execDisplayImage
       293: 00000000     0 FUNC    GLOBAL DEFAULT  UND open@@GLIBC_2.4
       294: 00017e78  1388 FUNC    GLOBAL DEFAULT   13 EPDC_init
       295: 0002c1d0     0 OBJECT  GLOBAL HIDDEN    23 __dso_handle
       296: 00018564    76 FUNC    GLOBAL DEFAULT   13 EPDC_end
       297: 0001a1e4     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
       298: 000184e4   128 FUNC    GLOBAL DEFAULT   13 EPDC_execPartDisplayImage
       299: 00019760    44 FUNC    GLOBAL DEFAULT   13 SDIA_getSdInfo
       300: 00000000     0 FUNC    GLOBAL DEFAULT  UND strlen@@GLIBC_2.4
       301: 00000000     0 FUNC    GLOBAL DEFAULT  UND mmap@@GLIBC_2.4
       302: 00000000     0 FUNC    GLOBAL DEFAULT  UND strchr@@GLIBC_2.4
       303: 00015018   180 FUNC    GLOBAL DEFAULT   13 PRC_init
       304: 000144b8   124 FUNC    GLOBAL DEFAULT   13 GPIO_isPushedNextBook
       305: 000fb244     4 OBJECT  GLOBAL DEFAULT   24 _p_Fb
       306: 00016288    36 FUNC    GLOBAL DEFAULT   13 FLC_getCurrentLanguage
       307: 00000000     0 FUNC    GLOBAL DEFAULT  UND fcntl@@GLIBC_2.4
       308: 0001a168    96 FUNC    GLOBAL DEFAULT   13 __libc_csu_init
       309: 00016058   160 FUNC    GLOBAL DEFAULT   13 PRC_prepareDisplayforBook
       310: 00018f58   300 FUNC    GLOBAL DEFAULT   13 AESC_decrypt
       311: 00000000     0 FUNC    GLOBAL DEFAULT  UND memset@@GLIBC_2.4
       312: 000fb24c     0 NOTYPE  GLOBAL DEFAULT   24 _end
       313: 00000000     0 FUNC    GLOBAL DEFAULT  UND strncpy@@GLIBC_2.4
       314: 00017dd8    80 FUNC    GLOBAL DEFAULT   13 FLC_getSleepFilePath
       315: 00017628   300 FUNC    GLOBAL DEFAULT   13 FLC_updateBookPrev
       316: 000141d0   124 FUNC    GLOBAL DEFAULT   13 GPIO_getShutdownReq
       317: 00010cac     0 FUNC    GLOBAL DEFAULT   13 _start
       318: 00017160   328 FUNC    GLOBAL DEFAULT   13 FLC_updatePageNext
       319: 00000000     0 FUNC    GLOBAL DEFAULT  UND write@@GLIBC_2.4
       320: 000fb24c     0 NOTYPE  GLOBAL DEFAULT   24 __end__
       321: 00015b00   156 FUNC    GLOBAL DEFAULT   13 PRC_loadImageAfterDisplay
       322: 00000000     0 FUNC    GLOBAL DEFAULT  UND shmdt@@GLIBC_2.4
       323: 0002cae8     0 NOTYPE  GLOBAL DEFAULT   24 __bss_start
       324: 00000000     0 FUNC    GLOBAL DEFAULT  UND fclose@@GLIBC_2.4
       325: 00010dd4   960 FUNC    GLOBAL DEFAULT   13 main
       326: 00000000     0 FUNC    GLOBAL DEFAULT  UND munmap@@GLIBC_2.4
       327: 00000000     0 FUNC    GLOBAL DEFAULT  UND strtok@@GLIBC_2.4
       328: 00016be0   764 FUNC    GLOBAL DEFAULT   13 FLC_saveLastPagePath
       329: 0001753c   236 FUNC    GLOBAL DEFAULT   13 FLC_updateBookNext
       330: 00011964    76 FUNC    GLOBAL DEFAULT   13 getPageNextCnt
       331: 0001978c   124 FUNC    GLOBAL DEFAULT   13 CCC_init
       332: 00018764   404 FUNC    GLOBAL DEFAULT   13 MEM_init
       333: 000150cc    20 FUNC    GLOBAL DEFAULT   13 PRC_end
       334: 00000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
       335: 00019f68   512 FUNC    GLOBAL DEFAULT   13 KEY_getkey_
       336: 00000000     0 FUNC    GLOBAL DEFAULT  UND sprintf@@GLIBC_2.4
       337: 00000000     0 FUNC    GLOBAL DEFAULT  UND ftok@@GLIBC_2.4
       338: 000fb248     4 OBJECT  GLOBAL DEFAULT   24 _fbLength
       339: 00000000     0 FUNC    GLOBAL DEFAULT  UND atoi@@GLIBC_2.4
       340: 000185b0    36 FUNC    GLOBAL DEFAULT   13 EPDC_getFbSize
       341: 00015ca0   312 FUNC    GLOBAL DEFAULT   13 PRC_runDisplaySequence
       342: 0002cae8     0 OBJECT  GLOBAL HIDDEN    23 __TMC_END__
       343: 0001443c   124 FUNC    GLOBAL DEFAULT   13 GPIO_isPushedMultiFunctio
       344: 00017a48   596 FUNC    GLOBAL DEFAULT   13 FLC_getNextPagePath
       345: 00015f18   160 FUNC    GLOBAL DEFAULT   13 PRC_prepareDisplayforPage
       346: 000196d0    24 FUNC    GLOBAL DEFAULT   13 SDIA_init
       347: 00000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
       348: 000151e4   208 FUNC    GLOBAL DEFAULT   13 PRC_updatePageNo
       349: 00000000     0 FUNC    GLOBAL DEFAULT  UND chmod@@GLIBC_2.4
       350: 00019d3c   404 FUNC    GLOBAL DEFAULT   13 CCC_fopenWithDecryption
       351: 00011a94  9932 FUNC    GLOBAL DEFAULT   13 GPIO_init
       352: 00000000     0 FUNC    GLOBAL DEFAULT  UND fputs@@GLIBC_2.4
       353: 00000000     0 FUNC    GLOBAL DEFAULT  UND strncmp@@GLIBC_2.4
       354: 00016360  1240 FUNC    GLOBAL DEFAULT   13 FLC_init
       355: 00000000     0 FUNC    GLOBAL DEFAULT  UND abort@@GLIBC_2.4
       356: 00010a4c     0 FUNC    GLOBAL DEFAULT   11 _init
       357: 00000000     0 FUNC    GLOBAL DEFAULT  UND close@@GLIBC_2.4
       358: 00015dd8   160 FUNC    GLOBAL DEFAULT   13 PRC_endDisplaySequence
       359: 00016ba0    64 FUNC    GLOBAL DEFAULT   13 FLC_getLastPagePath
       360: 00015fb8   160 FUNC    GLOBAL DEFAULT   13 PRC_prepareDisplayforBook
       361: 000fb11c    64 OBJECT  GLOBAL DEFAULT   24 _imagePath
    
    Version symbols section '.gnu.version' contains 52 entries:
     Addr: 000000000001081c  Offset: 0x00081c  Link: 5 (.dynsym)
      000:   0 (*local*)       0 (*local*)       0 (*local*)       0 (*local*)    
      004:   0 (*local*)       2 (GLIBC_2.4)     3 (GLIBC_2.4)     2 (GLIBC_2.4)  
      008:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     2 (GLIBC_2.4)  
      00c:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)  
      010:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     2 (GLIBC_2.4)     3 (GLIBC_2.4)  
      014:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)  
      018:   2 (GLIBC_2.4)     3 (GLIBC_2.4)     2 (GLIBC_2.4)     3 (GLIBC_2.4)  
      01c:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)  
      020:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)  
      024:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     2 (GLIBC_2.4)  
      028:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)  
      02c:   3 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)     2 (GLIBC_2.4)  
      030:   3 (GLIBC_2.4)     2 (GLIBC_2.4)     3 (GLIBC_2.4)     3 (GLIBC_2.4)  
    
    Version needs section '.gnu.version_r' contains 2 entries:
     Addr: 0x0000000000010884  Offset: 0x000884  Link to section: 6 (.dynstr)
      000000: Version: 1  File: libc.so.6  Cnt: 1
      0x0010:   Name: GLIBC_2.4  Flags: none  Version: 3
      0x0020: Version: 1  File: libpthread.so.0  Cnt: 1
      0x0030:   Name: GLIBC_2.4  Flags: none  Version: 2
    
    Notes at offset 0x00000150 with length 0x00000020:
      Owner		Data size	Description
      GNU		0x00000010	NT_VERSION (version)
    
    Notes at offset 0x00000170 with length 0x00000024:
      Owner		Data size	Description
      GNU		0x00000014	Unknown note type: (0x00000003)
    Attribute Section: aeabi
    File Attributes
      Tag_CPU_name: "Cortex-A9"
      Tag_CPU_arch: v7
      Tag_CPU_arch_profile: Application
      Tag_ARM_ISA_use: Yes
      Tag_THUMB_ISA_use: Thumb-2
      Tag_VFP_arch: ??? (3)
      Tag_NEON_arch: NEONv1
      Tag_ABI_PCS_wchar_t: 4
      Tag_ABI_FP_rounding: Needed
      Tag_ABI_FP_denormal: Needed
      Tag_ABI_FP_exceptions: Needed
      Tag_ABI_FP_number_model: IEEE 754
      Tag_ABI_align8_needed: Yes
      Tag_ABI_align8_preserved: Yes, except leaf SP
      Tag_ABI_enum_size: int
      Tag_ABI_VFP_args: VFP registers
      Tag_unknown_34: 1 (0x1)
      Tag_unknown_42: 1 (0x1)
      Tag_unknown_68: 1 (0x1)

Wow! So, not only is this file an ELF for ARM, it's a standard GNU/Linux
executable, and they didn't strip symbols. And they used function names that
are actually descriptive. This will save a lot of time in reverse engineering
how the program actually runs so that I can develop a free replacement.

### Image Files

As mentioned before, the manga are stored one image per page, one directory
per volume. A quick perusal shows that every image file is 1186848 bytes. This
is a pretty good indicator that the images are stored in a non-compressed
format, likely to simplify the process of mapping them from disk to display
memory. Sure enough, tinkering in GIMP by opening an image file as "Raw image
data", with an "Image Type" of "Indexed", "Offset" 0, "Width" 936, and "Height"
1268 yields a black and white bitmap. Curiously, the image opens in GIMP
rotated 90° counterclockwise from normal reading orientation. This would
indicate that the origin of the eink screens is at the top right corner, and
the memory addresses of individual pixels increases top-to-bottom,
right-to-left. As expected, the page numbering places right-hand pages on
lower numbers than the corresponding left-hand page (e.g. page `000` is
displayed on the right-hand screen, while `001` is on the left-hand screen),
following Japanese reading order.

As noted before, some pages are encrypted, with filenames suffixed with `E`
to indicate this. In fact, only pages `000` and `001` of any volume are
not encrypted. Based on the function names present in `.A001`, the encryption
algorithm in use is most likely AES, though there also seems to be mentions
of "CCC", which I am not familiar with. As mentioned before, the encryption
key appears to be derived somehow from the SDHC card. It will take some poking
through the key-related fucntions in `.A001` to see if I can extract it.

I'm currently working on a command line utility to convert eOneBook image files
back and forth to PNM (which can in turn be converted to and from pretty much
any format using ImageMagick).

Of note is that I am able to modify unencrypted pages using GIMP, and they
display just fine on an ordinary SDHC card.
