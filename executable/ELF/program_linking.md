# 程序链接

# 静态链接

# 动态链接

动态链接主要是在程序初始化时或者程序执行的过程中解析变量或者函数的引用。ELF文件中某些节区以及头部元素就与动态链接有关。动态链接的模型由操作系统定义并实现。

## Program Interpreter

参与动态链接的可执行文件应该具有一个PT_INTERP类型的程序头元素。在exec (BA_OS)过程中，系统会从PT_INTERP中提取一个路径名，并从解释器文件的段创建初始时的程序镜像。也就是输，系统并不使用原始的可执行文件的镜像，而会为解释器构造一个内存镜像。因此，解释器有责任从系统处获取控制权，然后为应用程序提供执行环境。

解释器从两个方面获取控制权。首先，它会接收到一个指向文件头的文件描述符来读取可执行文件。它可以使用这个文件描述符来读取以及将可执行文件的段映射到内存中。其次，有时候根据可执行文件格式的不同，系统有可能不会把文件描述符给解释器，而是会直接将可执行文件加载到内存中。文件描述符可能会出现异常，但是解释器的初始状态仍然会与可执行文件的收到的相匹配，解释器本身不需要再有一个解释器。解释器本身可能是一个共享目标文件或者是一个可执行文件。

- 共享目标文件（正常情况下）被加载为位置独立的，也就是说，对于不同的进程来说，它的地址会有所不同。系统通过mmap (KE_OS)以及一些相关的操作来创建动态段中的内容。因此，共享目标文件的地址通常来说不会和原来的可执行文件的原有地址冲突。
- 可执行文件会被加载到固定的地址。系统通过程序头部表的虚拟地址来创建对应的段。因此，一个可执行文件的解释器的虚拟地址可能和第一个可执行文件冲突。解释器有责任来解决相应的冲突。

## Dynamic Linker

当使用动态链接来构造程序时，链接编辑器会在可执行文件的程序头部添加一个PT_INTERP类型的元素，以便于告诉系统将动态连接器作为程序解释器来调用。

> 需要注意的是，对于系统提供的动态链接器，不同的系统会不同。

可执行程序和动态链接器会合作起来为程序创建进程镜像，具体的细节如下：

1. 将可执行文件的内存段添加到进程镜像中。
2. 将共享目标文件的内存端添加到进程镜像中。
3. 为可执行文件以及共享目标文件进行重定位。
4. 如果传递给了动态链接器一个文件描述符的话，就将其关闭。
5. 将控制权传递给程序，就好像程序直接从可执行文件处拿到了执行权限。

链接编辑器同样也创建了各种各样的数据来协助动态连接器来处理可执行文件和共享目标文件，例如

- 类型为SHT_DYNAMIC的节.dynamic包含了各种各样的数据，在这个节的开始处包含了其它动态链接的信息。
- 类型为SHT_HASH的节.hash包含了符号哈希表。
- 类型为SHT_PROGBITS的节.got以及.plt包含了两个独立的表：全局偏移表，以及过程链接表。程序会利用过程链接表来处理位置独立代码。

因为所有的UNIX System V都会从一个共享目标文件中导入基本的系统服务，动态连接器会参与到每一个TIS ELF-conforming的程序执行过程中。

正如程序加载中所说的，共享目标文件中可能会占用与程序头部中记录的不一样的虚拟地址。动态连接器会在程序拿到控制权前，重定位内存镜像，更新绝对地址。如果共享目标文件确实加载到了其在程序头部中指定的地址的话，那么那些绝对地址的值就会使正确的。但通常情况下，这种情况不会发生。

如果进程的环境中有名叫LD_BIND_NOW的非空值，那么动态连接器在把权限传递给程序时，会执行所有的重定位。例如，所有的如下环境变量的取值都会专门指定这一行为。

- LD_BIND_NOW = 1
- LD_BIND_NOW = on
- LD_BIND_NOW = off

否则，LD_BIND_NOW要么不存在于当前的进程环境中，要么具有一个非空值。动态连接器可以延迟解析过程链接表的入口，这其实就是plt表的延迟绑定，即当程序需要使用某个符号时，再进行地址解析，这可以减少符号解析以及重定位的负载。

## Dynamic Section

如果一个目标文件参与到动态链接的过程中，那么它的程序头部表将会包含一个类型为PT_DYNAMIC的元素。这个段包含了.dynamic节。_DYNAMIC符号会用来标记这个节，它的结构如下

```c++
typedef struct {
	Elf32_Sword		d_tag;
	union {
		Elf32_Word	d_val;
		Elf32_Addr	d_ptr;
	} d_un;
} Elf32_Dyn;

extern Elf32_Dyn_DYNAMIC[];
```

其中，d_tag控制着d_un的解释。

- d_val
  - 这个字段表示一个整数值，可以有多种意思。
- d_ptr
  - 这个字段表示程序的虚拟地址。正如之前所说的，一个文件的虚拟地址在执行的过程中可能和内存的虚拟地址不匹配。当解释动态结构中的地址时，动态连接器会根据原始文件的值以及内存的基地址来计算真正的地址。为了保持一致性，文件中并不会包含重定位入口来"纠正"动态结构中的地址。


下表总结了可执行文件以及共享目标文件中的d_tag的需求。如果一个tag被标记为"mandatory"，那么对于一个TIS ELF conforming的文件来说，其动态链接数组必须包含对应入口的类型。同样的，“optional”意味着可以有，也可以有没有。

| 名称                    | 数值                     | d_un  | 可执行  | 共享 目标 | 说明                                       |
| --------------------- | ---------------------- | ----- | ---- | ----- | ---------------------------------------- |
| DT_NULL               | 0                      | 忽略    | 必需   | 必需    | 标志着 \_DYNAMIC 数组的末端。                     |
| DT_NEEDED             | 1                      | d_val | 可选   | 可选    | 包含以NULL 结尾的字符串的字符串表偏移，该字符串给出某个需要的库的名称。所使用的索引为DT_STRTAB的下标。动态数组中可以包含很多个这种类型的标记。这些项在这种类型标记中的相对顺序比较重要。但是与其它的标记之前的顺序倒无所谓。 |
| DT_PLTRELSZ           | 2                      | d_val | 可选   | 可选    | 给出与过程链接表相关的重定位项的总的大小。如果存在DT_JMPREL类型的项，那么DT_PLTRELSZ也必须存在。 |
| DT_PLTGOT             | 3                      | d_ptr | 可选   | 可选    | 给出与过程链接表或者全局偏移表相关联的地址。                   |
| DT_HASH               | 4                      | d_ptr | 必需   | 必需    | 此类型表项包含符号哈希表的地址。此哈希表指的是被 DT_SYMTAB 引用的符号表。 |
| DT_STRTAB             | 5                      | d_ptr | 必需   | 必需    | 此类型表项包含字符串表的地址。符号名、库名、和其它字符串都包含在此表中。     |
| DT_SYMTAB             | 6                      | d_ptr | 必需   | 必需    | 此类型表项包含符号表的地址。对 32 位的文件而言，这个符号表中的条目的类型为 Elf32_Sym。 |
| DT_RELA               | 7                      | d_ptr | 必需   | 可选    | 此类型表项包含重定位表的地址。此表中的元素包含显式的补齐，例如 32 位文件中的 Elf32_Rela。目标文件可能有多个重定位节区。在为可执行文件或者共享目标文件创建重定位表时，链接编辑器将这些节区连接起来，形成一个表。尽管在目标文件中这些节区相互独立，但是动态链接器把它们视为一个表。在动态链接器为可执行文件创建进程映像或者向一个进程映像中添加某个共享目标时，要读取重定位表并执行相关的动作。如果此元素存在，动态结构体中也必须包含 DT_RELASZ 和 DT_RELAENT 元素。如果对于某个文件来说，重定位是必需的话，那么 DT_RELA 或者 DT_REL 都可能存在。 |
| DT_RELASZ             | 8                      | d_val | 必需   | 可选    | 此类型表项包含 DT_RELA 重定位表的总字节大小。              |
| DT_RELAENT            | 9                      | d_val | 必需   | 可选    | 此类型表项包含 DT_RELA 重定位项的字节大小。               |
| DT_STRSZ              | 10                     | d_val | 必需   | 必需    | 此类型表项给出字符串表的字节大小，按字节数计算。                 |
| DT_SYMENT             | 11                     | d_val | 必需   | 必需    | 此类型表项给出符号表项的字节大小。                        |
| DT_INIT               | 12                     | d_ptr | 可选   | 可选    | 此类型表项给出初始化函数的地址。                         |
| DT_FINI               | 13                     | d_ptr | 可选   | 可选    | 此类型表项给出结束函数（Termination Function）的地址。    |
| DT_SONAME             | 14                     | d_val | 忽略   | 可选    | 此类型表项给出一个以 NULL 结尾的字符串的字符串表偏移，对应的字符串是某个共享目标的名称。该偏移实际上是 DT_STRTAB 中的索引。 |
| DT_RPATH              | 15                     | d_val | 可选   | 忽略    | 此类型表项包含以 NULL 结尾的字符串的字符串表偏移，对应的字符串是搜索库时使用的搜索路径。该偏移实际上是 DT_STRTAB 中的索引。 |
| DT_SYMBOLIC           | 16                     | 忽略    | 忽略   | 可选    | 如果这种类型表项出现在共享目标库中，那么这将会改变动态链接器的符号解析算法。动态连接器将首先选择从共享目标文件本身开始搜索符号，只有在搜索失败时，才会选择从可执行文件中搜索相应的符号。 |
| DT_REL                | 17                     | d_ptr | 必需   | 可选    | 此类型表项与 DT_RELA类型的表项类似，只是其表格中包含隐式的补齐，对 32 位文件而言，就是 Elf32_Rel。如果ELF文件中包含此元素，那么动态结构中也必须包含 DT_RELSZ 和 DT_RELENT 类型的元素。 |
| DT_RELSZ              | 18                     | d_val | 必需   | 可选    | 此类型表项包含 DT_REL 重定位表的总字节大小。               |
| DT_RELENT             | 19                     | d_val | 必需   | 可选    | 此类型表项包含 DT_REL 重定位项的字节大小。                |
| DT_PLTREL             | 20                     | d_val | 可选   | 可选    | 此类型表项给出过程链接表所引用的重定位项的类型。根据具体情况， d_val 成员可能包含 DT_REL 或者  DT_RELA。过程链接表中的所有重定位都必须采用相同的重定位方式。 |
| DT_DEBUG              | 21                     | d_ptr | 可选   | 忽略    | 此类型表项用于调试。ABI 未规定其内容，访问这些条目的程序可能与 ABI 不兼容。 |
| DT_TEXTREL            | 22                     | 忽略    | 可选   | 可选    | 如果文件中不包含此类型的表项，则表示没有任何重定位表项能够造成对不可写段的修改。如果存在的话，则可能存在若干重定位项请求对不可写段进行修改，因此，动态链接器可以做相应的准备。 |
| DT_JMPREL             | 23                     | d_ptr | 可选   | 可选    | 如果存在这种类型的表项，则表示条目的 d_ptr 成员包含了某个重定位项的地址，并且该重定位项仅与过程链接表相关。把重定位项分开有利于让动态链接器在进程初始化时忽略它们（开启了延迟绑定）。如果存在此成员，相关的 DT_PLTRELSZ 和  DT_PLTREL 必须也存在。 |
| DT_BIND_NOW           | 24                     | 忽略    | 可选   | 可选    | 如果可执行文件或者共享目标文件中存在此类型的表项的话，动态链接器在将控制权转交给程序前，应该将该文件的所有需要重定位的地址都进行重定位。这个表项的优先权高于延迟绑定，可以通过环境变量或者dlopen(BA_LIB)来设置。 |
| DT_LOPROC  ~DT_HIPROC | 0x70000000 ~0x7fffffff | 未指定   | 未指定  | 未指定   | 这个范围的表项是保留给处理器特定的语义的。                    |

没有出现在此表中的标记值是保留的。此外，除了数组末尾的 DT_NULL 元素以及 DT_NEEDED 元素的相对顺序约束以外， 其他表项可以以任意顺序出现。

## Shared Object Dependencies

当链接编辑器在处理一个归档库的时候，它会提取出库成员并且把它们拷贝到输出目标文件中。这种静态链接的操作在执行过程中是不需要动态连接器参与的。共享目标文件同时也提供了服务，动态链接器必须将合适的共享目标文件attach到进程镜像中，以便于执行。因此，可执行文件以及共享目标文件会专门描述他们的依赖关系。

当一个动态链接器为一个目标文件创建内存段时，在DT_NEEDED项中描述的依赖给出了需要什么依赖文件来支持程序的服务。通过不断地连接被引用的共享目标文件（即使一个共享目标文件被引用多次，它最后也只会被动态链接器连接一次）及他们的依赖，动态链接器建立了一个完全的进程镜像。当解析符号引用时，动态链接器会使用BFS来检查符号表。也就是说，首先，它会检查可执行文件本身的符号表，然后才会按照顺序检查DT_NEEDED入口中的符号表，然后才会继续查看下一次依赖，依次类推。共享目标文件必须可以被程序读取，其它权限不一定需要。

依赖列表中的名字要么是DT_SONAME中的字符串，要么是用于构建对应目标文件的共享目标文件的路径名。例如，如果一个链接器使用了一个带有DT_SONAME项名字叫做lib1的共享目标文件以及一个其他路径名为/usr/lib/lib2的共享目标文件，那么可执行文件中将会包含lib1以及/usr/lib/lib2依赖列表。

如果一个共享目标文件具有一个或者多个/，例如/usr/lib/lib2或者directory/file，那么动态链接器会直接使用那个字符串来作为路径的名字。如果名字中没有/，比如lib1，那么以下的三种机制给出了共享目标文件搜索的顺序。

- 首先，动态数组标记DT_RPATH可能会给出一个包含一系列以:分割的目录的字符串。例如 /home/dir/lib:/home/dir2/lib: 告诉我们先在`/home/dir/lib`目录搜索，然后再在`/home/dir2/lib`搜索，最后在当前目录搜索。

- 其次，进程环境变量中的名叫LD_LIBRARY_PATH的变量包含了一系列上述所说格式的目录，最后可能会有一个;，后面跟着另外一个目录列表后面跟着另外一个目录列表。这里给出一个例子，效果与第一个所说的效果相同

  - LD_LIBRARY_PATH=/home/dir/lib:/home/dir2/lib:
  - LD_LIBRARY_PATH=/home/dir/lib;/home/dir2/lib:
  - LD_LIBRARY_PATH=/home/dir/lib:/home/dir2/lib:;

  所有的LD_LIBRARY_PATH中的目录只会在搜索完DT_RPATH才会进行搜索。尽管有一些程序（如链接编辑器）在处理;前后的列表的方式不同，但是动态链接器处理的方式完全一样，除此之外，动态链接器接受分号表示语法，正如上面所描述的样子。

- 最后，如果以上两组目录无法定位期望的库，则动态链接器搜索`/usr/lib` 路径下的库。

注意

> 为了安全性，对于`set-user` 以及 `set-group` 标识的程序，动态链接器忽略搜索环境变量（例如`LD_LIBRARY_PATH`），仅仅搜索`DT_RPATH`指定的目录和`/usr/lib`。





## Global Offset Table

通常来说，位置独立代码不能包含绝对虚拟地址。GOT表中包含了隐藏的绝对地址，这使得在不违背位置无关性以及程序代码段兼容的情况下，仍然可以得到相关的地址。一个程序可以使用位置独立代码来引用它的GOT表，然后提取出来绝对的数值，以便于将位置独立的引用重定向绝对的地址。 这个表对于System V环境中的动态链接来说是必要的，但其具体的内容以及形式依赖于处理器。

初始时，got表中包含重定向入口所需要的信息。当一个系统为可加载的目标文件创建内存段时，动态链接器会处理重定位项，其中的一些项的类型可能是R\_386\_GLOB_DAT，这会指向got表。动态链接器会决定相关的符号的值，计算它们的绝对地址，然后将核实的内存表项设置为合适的值。尽管在链接器建立目标文件时，绝对地址还处于未知状态，动态链接器知道所有内存段的地址，因为可以计算所包含的符号的绝对地址。

如果一个程序需要直接访问一个符号的绝对地址，那么那个符号将会有一个got表项。由于可执行文件以及共享不同文件都有单独的表项，所以一个符号的地址可能会出现在多个表中。动态链接器在把权限给到进程镜像中的代码段前，会处理所有的got表中的重定位项，以便于确定所有的绝对地址在 执行过程中是可以访问的。

GOT表中的第0项包含动态结构的地址，用符号_DYNAMIC来进行引用。这使得一个程序，例如动态链接器，在没有执行其重定向前可以找到对应的动态结构。这对于动态链接器来说是非常重要的，因为它必须在不依赖其它程序的情况下可以重定位自己的内存镜像。在Intel架构中，GOT中的表项1和表项2用于处理PLT表。

在不同的程序中，系统可能会为同一共享目标文件选择不同的内存段地址；甚至对于同一个程序，在不同的执行过程中，也会有不同的库地址。然而，一旦进程奖项被建立，内存段的地址就不会再改变，只要一个进程还存在，它的内存段地址将处于固定的位置。

GOT表的形式以及解释依赖于具体的处理器，对于Intel架构来说，`_GLOBAL_OFFSET_TABLE_`符号可能被用来访问这个表。

```c
extern Elf32_Addr _GLOBAL_OFFSET_TABLE[];
```

\_GLOBAL_OFFSET_TABLE_ 可能会在.got节的中间，以便于可以使用正负索引来访问这个表。



## Procedure Linkage Table

GOT表用来将位置独立的地址重定向为绝对地址，与此类似，PLT表将位置独立的函数重定向到绝对地址。链接编辑器不能够解析执行流转换（比如程序调用），即从一个可执行文件或者共享目标文件到另一个文件。链接器安排程序将控制权交给过程链接表中的表项。在Intel架构中，过程链接表存在于共享代码段中，但是他们会使用在GOT表中的数据。动态链接器会决定目标的绝对地址，并且会修改相应的GOT表中的内存镜像。因此，动态链接器可以在不违背位置独立以及程序代码段兼容的情况下，重定向PLT项。可执行文件和共享目标文件都有独立的PLT表。

绝对地址的过程链接表如下

```assembly
.PLT0:pushl	got_plus_4
      jmp	*got_plus_8
	  nop; nop
	  nop; nop
.PLT1:jmp	*name1_in_GOT
      pushl	$offset@PC
.PLT2:jmp	*name2_in_GOT
      push	$offset
	  jmp	.PLT0@PC
	  ...
```

位置无关的过程链接表的地址如下

```assembly
.PLT0:pushl	4(%ebx)
      jmp	*8(%ebx)
	  nop; nop
	  nop; nop
.PLT1:jmp	*name1_in_GOT(%ebx)
      pushl	$offset
	  jmp	.PLT0@PC
.PLT2:jmp	*name2_in_GOT(%ebx)
      push	$offset
	  jmp	.PLT0@PC
	  ...
```

可以看出过程链接表针对于绝对地址以及位置独立的代码的处理不同。但是动态链接器处理它们时，所使用的接口是一样的。

动态链接器和程序按照如下方式解析过程链接表和全局偏移表的符号引用。

1. 当第一次建立程序的内存镜像时，动态链接器将全局偏移表的第二个和第三个项设置为特殊的值，下面的步骤会仔细解释这些数值。
2. 如果过程链接表是位置独立的话，那么GOT表的地址必须在ebx寄存器中。每一个进程镜像中的共享目标文件都有独立的PLT表，并且程序只在同一个目标文件将控制流交给PLT表项。因此，调用函数负责在调用PLT表项之前，将全局偏移表的基地址设置为寄存器中。
3. 这里举个例子，假设程序调用了name1，它将控制权交给了lable .PLT1。
4. 那么，第一条指令将会跳转到全局偏移表中name1的地址。初始时，全局偏移表中包含PLT中下一条pushl指令的地址，并不是name1的实际地址。
5. 因此，程序将一个重定向偏移压到栈上。重定位偏移是32位的，并且是非负的数值。此外，重定位表项的类型为R\_386\_JMP_SLOT，并且它将会说明在之前jmp指令中使用的全局偏移表项在GOT表中的偏移。重定位表项也包含了一个符号表索引，因此告诉动态链接器什么符号目前正在被引用。在这个例子中，就是name1了。
6. 在压入重定位偏移后，程序会跳转到.PLT0，这是过程链接表的第一个表项。pushl指令将GOT表的第二个表项(got_plus_4 或者4(%ebx))压到栈上，然后给动态链接器一个识别信息。此后，程序会跳转到第三个全局偏移表项(got_plus\_8 或者8(%ebx)) 处，这将会将程序流交给动态链接器。
7. 当动态链接器接收到控制权后，他将会进行出栈操作，查看重定位表项，找到对应的符号的值，将name1的整整的地址存储在全局偏移表项中，然后将控制权交给目的地址。
8. 过程链接表执行之后，程序的控制权将会直接交给name1函数，而且此后再也不会调用动态链接器来解析这个函数。也就是说，在.PLT1处的jmp指令将会直接跳转到name1处，而不是再次执行pushl指令。

LD_BIND_NOW环境变量可以改变动态链接器的行为。如果它的值非空的话，动态链接器在将控制权交给程序之前会执行PLT表项。也就是说，动态链接器在进程初始化过程中执行类型为R\_3862\_JMP_SLOT的重定位表项。否则的话，动态链接表会对过程链接表项进行延迟绑定，直到第一次执行对应的表项时，才会今次那个符号解析以及重定位。

注意

> 惰性绑定通常来说会提高应用程序的性能，因为没有使用的符号并不会增加动态链接的负载。然而，有以下两种情况将会使得惰性绑定出现未预期的情况。首先，对于一个共享目标文件的函数的初始引用一般来说会超过后续调用的时间，因为动态链接器需要拦截调用以便于去解析符号。一些应用并不能够忍受这种不可预测性。其次，如果发生了错误，并且动态链接器不能够解析符号。动态链接器将会终止程序。在惰性绑定的情况下，这种情况可能随时发生。当关闭了惰性绑定的话，动态链接器在进程初始化的过程中就不会出现相应的错误，因为这些都是在应用获得控制权之前执行的。



## Hash Table

















