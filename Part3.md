## 实现一个kv数据库-第三部分 Kyoto Cabinet和LevelDB的体系结构比较分析

@(翻译)[kvdb|Markdown]

####3.5 字符串

1. 在kv数据库中，有很多的字符串操作在运行着，字符串被复制，散列，压缩，传递和返回。所以，一个高效的字符串实现方式是非常重要的，因为大规模的使用一个设计小巧的字符串实现对整个系统的影响是巨大的。

2. 为了解决这个问题，LevelDB使用了一个专门的类叫做"Slice"，这个类拥有一个字符数组和一个关于这个数组大小的数字。这使得获得这个字符串长度的时间复杂度为O(1),~~不像std::string的size()为O(n)~~,不像C中的strlen()函数需要O(n)。在C++的std::string的size()同样也是O(1)，Slice存储了字符串的长度，并且字符串可以存储'\0'字符，这表示k和v都是真正的字符串，而不是以空结尾的字符串。最后的也是很重要的，Slice的拷贝方式是浅拷贝方式，不是深拷贝，意思是只是把指向字符数组的的指针和字符串长度拷贝了一份，不是像std::string完整的拷贝字符数组。(译者注:参见深拷贝)这避免了对与大的字符数组的拷贝。

3. 像LevelDB，Redis也使用了它自己实现的string的数据结构，目的也在于避免为O(n)的操作来检索字符串的大小。Kyoto Cabinet使用的是std::string作为他的sting实现。我的看法是实现一个字符串去适应kv数据库的需求是绝对有必要的，如果能够避免，为什么要拷贝字符串(译者注:此处应该是深拷贝)和分配内存？

####3.6 错误管理

1. 在所有的我看过的kv数据库的c++源码中，我还从没见过一个使用单一异常被用在全局错误管理系统中。在Kyoto Cabinet中，在kcthread.cc中的线程组件使用了异常机制，但是我认为与其说这是个架构上的选择, 不如说只是要操作多线程罢了。异常是危险的，并且应该尽可能的避免。

2. BerkeleyDB有着很好的c分格的方式来处理错误，错误信息和错误代码都集中在一个文件中，所有函数返回的错误代码都有一个名为“RET”的整型局部变量并且在最后处理和返回时被赋值。这种方法被使用在所有文件和模块中，非常亮，非常规范化的错误管理机制。在其中的一些函数中，少数需要向前跳跃执行的函数使用了goto语句，这是一种广泛应用在严谨的基于c的系统，比如linux内核，即使这个错误管理非常清晰，和干净，但是在c++程序中c风格的错误管理是没有太大意义的。

3. 在Kyoto Cabinet中，一个错误对象被存储在每个数据库对象中，例如HashDB，在数据库类中，如果当错误发生时，方法通过调用set_error()方法来设置错误对象，并非常像c风格的方式来返回ture或者false，像BerkeleyDB，在方法的最末端不返回局部变量，在错误发生的地方放置return语句。

4. LevelDB没有用任何的异常机制，而是用一种类叫做"Status",这个类拥有两个错误值和一个错误信息，这个对象由所有方法返回，使得错误状态既可以当场处理或传递给其他方法中较高的调用堆栈。"Status"的实现方式也是非常聪明，因为错误代码存储在字符串内部，我对这种选择的理解是，在多数情况下这个方法都返回"OK"，来表示没有出现任何错误。这这种情况下(没有任何错误)，这个消息字符串为NULL，并且状态对象的产生是非常轻量的。(Had the authors of LevelDB chosen to have one additional attribute to store the error code, this error code would have had to be filled even in the case of a Status of “OK”, which would have meant more space used on every method call. )曾经LevelDB的作者选择一个额外的属性来存储错误代码，即使没有任何错误的时候，错误代码也需要被赋值，这意味着在每个方法调用时需要使用更多的空间。所有的组件都使用这个"Status"类，而且不必要像Kyoto Cabinet通过一个集中的方法，如图3.1和3.2.

5. 在上面所介绍的的错误管理解决方案中，我个人比较喜欢LevelDB的解决方案，他避免了异常机制的使用，在我看来，它不是一个功能有限的单纯的c风格的错误管理方式，而且更是避免了与Kyoto Cabinet一样，造成和核心部件的不必要的藕合情况。

####3.7 内存管理

1. 无论是Kyoto Cabinet和LevelDB在核心部件中定义了内存管理，在Kyoto Cabinet中，内存管理包括跟踪磁盘数据库文件的连续的(原文:free memory)空闲存储块，并且当有项目需要存储时选择足够大的块。文件本身只是内存映射和mmap()函数，需要注意的是MongoDB也时用的内存映射文件。

2. 在LevelDB中实现了LSM Tree，在文件中可用空间是连续的因为他是存储磁盘中的在hash表。内存管理方式是当日志文件超过了一定大小，则对他进行压缩。

3. 其他的kv数据库，例如Redis，用malloc()的方式去分配内存，在Redis下，内存分配算法不是一种像dlmalloc或者ptmalloc3那样由操作系统提供的，而是jemalloc.(the memory allocation algorithm is not the one provided by the operating system like dlmalloc or ptmalloc3, but jemalloc)
详细的内存管理介绍将在的IKVS系列的后续文章中。

####3.8 数据存储

- Kyoto Cabinet, LevelDB, BerkeleyDB, MongoDB,Redis都是使用文件系统来存储数据，与此相反，Memcached是在内存中的存储数据。详细的数据存储介绍将在的IKVS系列的后续文章中。

###四、代码审查
- This section is a quick code review of Kyoto Cabinet and LevelDB. It is not thorough, and only contains elements that I judged remarkable when I was reading the source code.

####4.1 结构的声明和定义

- 如果在LevelDB中代码的组织是在.h头文件中声明，.cc实现文件中定义算是普通，那我的确在Kyoto Cabinet中找到了令我震惊的地方，很多类中，在.cc文件中没有任何定义，而是直接被定义在了头文件里。在其他文件中，一部分方法定义定义在.h文件里，另一部分方法在.cc文件中，当我了解到这样选择背后的原因的时候，我还是觉得没有遵循一个被大众认可的c++编程习惯从根本上还是错误的，他这个样子使我感到疑惑，而且让我为了了解实现方式的时候，需要去研究两个不同的文件，

####4.2 命名方式

- 首先，Kyoto Cabinet的代码相对与Tokyo Cabinet来说有了显著的改善，整体架构和命名规范都得到了很大的提升，尽管如此，我还是在Kyoto Cabinet中找到了很多让我疑惑的属性和方法的命名,比如embcomp，trhard，fmtver()，fpow()。这感觉就像一些c代码迷之进入了c++代码中。另一方面，在LevelDB中的命名方式是非常清晰的。除了一部分零时变量的命名，比mem,imm,in,但是是非常稀少的，他的代码任然具有可读性。

####4.3 代码重复

- 我已经看到了在Kyoto Cabinet中有很多的代码重复，用来整理文件的代码至少被重复了三次，在unix和win版本之间分支方法都出现了很多的代码重复现象。我还没在LevelDB中发现任何显著的代码块重复。我确定一定是有一些的，但我不得不去研究更深层次的去找到他，这证明了LevelDB中代码重复的问题比Kyoto Cabinet要小。
