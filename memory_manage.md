<h3><a href='http://darkc.at/about-data-structure-alignment/'>refer to</a></h3>

<ol>
  <h1><li>
    内存对齐（Data Structure Alignment）是什么</h1>
    <p>内存对齐，或者说字节对齐，是一个数据类型所能存放的内存地址的属性（<a href='http://msdn.microsoft.com/en-us/library/ms253949.aspx' target='_blank'>Alignment is a property of a memory address</a>）。</p>
    <p>这个属性是一个无符号整数，并且这个整数必须是2的N次方（1、2、4、8、……、1024、……）。
当我们说，一个数据类型的内存对齐为8时，意思就是指这个数据类型所定义出来的所有变量，其内存地址都是8的倍数。</p>
    <p>当一个基本数据类型（fundamental types）的对齐属性，和这个数据类型的大小相等时，这种对齐方式称作自然对齐（naturally aligned）。
比如，一个4字节大小的int型数据，默认情况下它的字节对齐也是4。</p>
  </li>
    
  <h1><li>
    为什么我们需要内存对齐</h1>
    <p>这是因为，并不是每一个硬件平台都能够随便访问任意位置的内存的。</p>
    <p>微软的<a href='http://msdn.microsoft.com/en-us/library/ms253949.aspx' target='_blank'>MSDN</a>里有这样一段话：</p>
    <blockquote>Many CPUs, such as those based on Alpha, IA-64, MIPS, and SuperH architectures, refuse to read misaligned data. When a program requests that one of these CPUs access data that is not aligned, the CPU enters an exception state and notifies the software that it cannot continue. On ARM, MIPS, and SH device platforms, for example, the operating system default is to give the application an exception notification when a misaligned access is requested.</blockquote>
    <p>大意是说，有不少平台的CPU，比如Alpha、IA-64、MIPS还有SuperH架构，若读取的数据是未对齐的（比如一个4字节的int在一个奇数内存地址上），将拒绝访问，或抛出硬件异常。</p>
    <p>另外，在<a href='http://en.wikipedia.org/wiki/Data_structure_alignment' target='_blank'>维基百科</a>里也记载着如下内容：</p>
    <blockquote>Data alignment means putting the data at a memory offset equal to some multiple of the word size, which increases the system's performance due to the way the CPU handles memory.</blockquote>
    <p>意思是，考虑到CPU处理内存的方式（32位的x86 CPU，一个时钟周期可以读取4个连续的内存单元，即4字节），使用字节对齐将会提高系统的性能（也就是CPU读取内存数据的效率。比如你一个int放在奇数内存位置上，想把这4个字节读出来，32位CPU就需要两次。但对齐之后一次就可以了）。</p>
  </li>
    
  <h1><li>
      内存对齐带来的数据结构大小变化</h1>
      <p>因为有了内存对齐，因此数据在内存里的存放就不是紧挨着的，而是可能会出现一些空隙（Data Structure Padding，也就是用于填充的空白内容）。因此对基本数据类型来说可能还好说，对于一个内部有多个基本类型的结构体（struct）或类而言，sizeof的结果往往和想象中不大一样。</p>
      <pre>
struct MyStruct  
{  
    char a;         // 1 byte  
    int b;          // 4 bytes  
    short c;        // 2 bytes  
    long long d;    // 8 bytes  
    char e;         // 1 byte  
};  
      </pre>
      <p>我们可以看到，MyStruct中有5个成员，如果直接相加的话大小应该是16，但在32位MSVC里它的sizeof结果是32。</p>
      <p>之所以结果出现偏差，为了保证这个结构体里的每个成员都应该在它对齐了的内存位置上，而在某些位置插入了Padding。</p>
      <p>下面我们尝试考虑内存对齐，来计算一下这个结构体的大小。首先，我们可以假设MyStruct的整体偏移从0x00开始，这样就可以暂时忽略MyStruct本身的对齐。这时，结构体的整体内存分布如下图所示：</p>
      <img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/align_1.png'>
      <p>我们可以看到，char和int之间；short和long long之间，为了保证成员各自的对齐属性，分别插入了一些Padding。因此整个结构体会被填充得看起来像这样：</p>
      <pre>
struct MyStruct  
{  
    char a;         // 1 byte  
    char pad_0[3];  // Padding 3  
    int b;          // 4 bytes  
    short c;        // 2 bytes  
    char pad_1[6];  // Padding 6  
    long long d;    // 8 bytes  
    char e;         // 1 byte  
    char pad_2[7];  // Padding 7  
};  
      </pre>
      <p>注意到上面加了Padding的示意结构体里，e的后面还跟了7个字节的填充。这是因为结构体的整体大小必须可被对齐值整除，所以“char e”的后面还会被继续填充7个字节好让结构体的整体大小是8的倍数32。</p>
      <p><b style="color:red;">注意:</b>字节对齐是单向增的，不会随着数据的自然对齐变小而变小。</p>
      <p>我们可以在gcc + 32位linux中尝试计算sizeof(MyStruct)，得到的结果是24。
这是因为gcc中的对齐规则和MSVC不一样，不同的平台下会使用不同的默认对齐值（<a href='http://gcc.gnu.org/onlinedocs/gcc/Variable-Attributes.html' target="_blank">The default alignment is fixed for a particular target ABI</a>）。在gcc + 32位linux中，大小超过4字节的基本类型仍然按4字节对齐。因此MyStruct的内存布局这时看起来应该像这个样子：</p>
      <img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/align_2.png'>
      <p>下面我们来确定这个结构体类型本身的内存对齐是多少。为了保证结构体内的每个成员都能够放在它自然对齐的位置上，对这个结构体本身来说最理想的内存对齐数值应该是结构体里内存对齐数值最大的成员的内存对齐数。</p>
      <p>也就是说，对于上面的MyStruct，结构体类型本身的内存对齐应该是8。并且，当我们强制对齐方式小于8时，比如设置MyStruct对齐为2，那么其内部成员的对齐也将被强制不能超过2。</p>
      <p>为什么？因为对于一个数据类型来说，其内部成员的位置应该是相对固定的。假如上面这个结构体整体按1或者2字节对齐，而成员却按照各自的方式自然对齐，就有可能出现成员的相对偏移量随内存位置而改变的问题。
比如说，我们可以画一下整个结构体按1字节对齐，并且结构体内的每个成员按自然位置对齐的内存布局：</p>
      <img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/align_3.png'>
      <p>上面的第一种情况，假设MyStruct的起始地址是0x01（因为结构体本身的偏移按1字节对齐），那么char和int之间将会被填充2个字节的Padding，以保证int的对齐还是4字节。如果第二次分配MyStruct的内存时起始地址变为0x03，由于int还是4字节对齐，则char和int之间将不会填充Padding（填充了反而不对齐了）。以此类推，若MyStruct按1字节对齐时不强制所有成员的对齐均不超过1的话，这个结构体里的相对偏移方式一共有4种。</p>
      <p>因此对于结构体来说，默认的对齐将等于其中对齐最大的成员的对齐值。并且，当我们限定结构体的内存对齐时，同时也限定了结构体内所有成员的内存对齐不能超过结构体本身的内存对齐。</p>
  </li>
  <h1><li>
      指定内存对齐</h1>
      <p>在C++98/03里，对内存对齐的操作在不同的编译器里可能有不同的方法。在MSVC中，一般使用#progma pack来指定内存对齐：</p>
      <pre>
#pragma pack(1) // 指定后面的内容内存对齐为1  
struct MyStruct  
{  
    char a;         // 1 byte  
    int b;          // 4 bytes  
    short c;        // 2 bytes  
    long long d;    // 8 bytes  
    char e;         // 1 byte  
};  
#pragma pack() // 还原默认的内存对齐  
      </pre>
      <p>这时，MyStruct由于按1字节对齐，其中的所有成员都将变为1字节对齐，因此sizeof(MyStruct)将等于16。还有另外一个简单的方法：</p>
      <pre>
__declspec(align(64)) struct MyStruct  
{  
    char a;         // 1 byte  
    int b;          // 4 bytes  
    short c;        // 2 bytes  
    long long d;    // 8 bytes  
    char e;         // 1 byte  
}; 
      </pre>
      <p><b>__declspec(align(64))</b>将指定内存对齐为64。比较坑的是，这种方法不能指定内存对齐小于默认对齐，也就是说它只能调大不能调小（<a href='https://docs.microsoft.com/en-us/cpp/cpp/align-cpp?redirectedfrom=MSDN&view=msvc-170' target="_blank">__declspec(align(#)) can only increase alignment restrictions</a>）。因此下面这样写会忽略掉declspec：</p>
      <p></p>
      <pre>
__declspec(align(1)) struct MyStruct // ...  
// warning C4359: 'MyStruct': Alignment specifier is less than actual alignment (8), and will be ignored.
      </pre>
      <p>微软的__declspec(align(#))，其#的内容可以是预编译宏，但不能是编译期数值：</p>
      <pre>
#define XX 32  
struct __declspec(align(XX)) MyStruct_1 {}; // OK  
   
template <size_t YY>  
struct __declspec(align(YY)) MyStruct_2 {}; // error C2059: syntax error: 'identifier'  
   
static const unsigned ZZ = 32;  
struct __declspec(align(ZZ)) MyStruct_3 {}; // error C2057: expected constant expression 
      </pre>
      <p>在<a href='http://www.microsoft.com/en-us/download/details.aspx?id=41151' target="_blank">Visual C++ Compiler November 2013 CTP</a>之后，微软终于支持编译期数值的写法了：</p>
      <pre>
template <size_t YY>  
struct __declspec(align(YY)) MyStruct_2 {}; // OK in 2013 CTP  
      </pre>
      <p> __declspec(align(#))最大支持对齐为8192（<a href="http://msdn.microsoft.com/en-us/library/83ythb65.aspx" target="_blank">Valid entries are integer powers of two from 1 to 8192</a>）。下面再来看gcc。gcc和MSVC一样，可以使用#pragma pack： </p>
      <pre>
#pragma pack(1)  
struct MyStruct  
{  
    // ...  
};  
#pragma pack()  
      </pre>
      <p>另外，也可以使用__attribute__((__aligned__((#))))：</p>
      <pre>
struct __attribute__((__aligned__((1)))) MyStruct_1  
{  
    // ...  
};  
   
struct MyStruct_2  
{  
    // ...  
} __attribute__((__aligned__((1))));   
      </pre>
      <p>这东西写上面写下面都是可以的，但是不能写在struct前面。和MSVC一样，__attribute__也只能把字节对齐改大，不能改小（<a href='http://gcc.gnu.org/onlinedocs/gcc-4.0.4/gcc/Type-Attributes.html' targt="_blank">The aligned attribute can only increase the alignment</a>）。比较坑的是当你试图改小的时候，gcc没有任何编译提示信息。gcc可以接受一个宏或编译期数值：</p>
      <pre>
#define XX 1  
struct __attribute__((__aligned__((XX)))) MyStruct_1 {}; // OK  
   
template <size_t YY>  
struct __attribute__((__aligned__((YY)))) MyStruct_2 {}; // OK  
   
static const unsigned ZZ = 1;  
struct __attribute__((__aligned__((ZZ)))) MyStruct_3 {};  
//                                        ^  
// error: requested alignment is not an integer constant  
      </pre>
      <p>gcc的__attribute__((__aligned__((#))))支持的上限受限于链接器（<a href='http://gcc.gnu.org/onlinedocs/gcc-4.0.4/gcc/Type-Attributes.html' target='_blank'>Note that the effectiveness of aligned attributes may be limited by inherent limitations in your linker</a>）。</p>
  </li>
  
  <h1><li>
      获得内存对齐</h1>
      <p>同样的，在C++98/03里，不同的编译器可能有不同的方法来获得一个类型的内存对齐。</p>
      <p>MSVC使用__alignof操作符获得内存对齐大小：</p>
      <pre>
MyStruct xx;  
std::cout << __alignof(xx) << std::endl;  
std::cout << __alignof(MyStruct) << std::endl;   
      </pre>
      <p>gcc则使用__alignof__：</p>
      <pre>
MyStruct xx;  
std::cout << __alignof__(xx) << std::endl;  
std::cout << __alignof__(MyStruct) << std::endl;  
      </pre>
      <p>需要注意的是，不论是__alignof还是__alignof__，对于对齐的计算都发生在编译期。因此像下面这样写：</p>
      <pre>
int a;  
char& c = reinterpret_cast<char&>(a);  
std::cout << __alignof__(c) << std::endl;  
      </pre>
      <p>得到的结果将是1。</p>
      <p>如果需要在运行时动态计算一个变量的内存对齐，比如根据一个void*指针指向的内存地址来判断这个地址的内存对齐是多少，我们可以用下面这个简单的方法：</p>
      <pre>
__declspec(align(128)) long a = 0;  
size_t x = reinterpret_cast<size_t>(&a);  
x &= ~(x - 1);  // 计算a的内存对齐大小  
std::cout << x << std::endl;
      </pre>
      <p>用这种方式得到的内存对齐大小可能比实际的大，因为它是切实的获得这个内存地址到底能被多大的2^N整除。</p>
  </li>
  
  <h1><li>
      堆内存的内存对齐</h1>
      <p>我们在讨论内存对齐的时候很容易忽略掉堆内存。我们经常会使用malloc分配内存，却不理会这块内存的对齐方式，仿佛堆内存不需要考虑内存对齐一样。</p>
      <p>实际上，malloc一般使用当前平台默认的最大内存对齐数对齐内存。比如MSVC在32位下一般是8字节对齐；64位下则是16字节（<a href='http://msdn.microsoft.com/en-us/library/6ewkz86d.aspx' target="_blank">In Visual C++, the fundamental alignment is the alignment that's required for a double, or 8 bytes. In code that targets 64-bit platforms, it’s 16 bytes</a>）。这样对于常规的数据都是没有问题的。</p>
      <p>但是如果我们自定义的内存对齐超出了这个范围，则是不能直接使用malloc来获取内存的。</p>
      <p></p>
      <p>当我们需要分配一块具有特定内存对齐的内存块时，在MSVC下应当使用_aligned_malloc；而在gcc下一般使用memalign等函数。</p>
      <p></p>
      <p>其实自己实现一个简易的aligned_malloc是很容易的：</p>
      <pre>
#include <assert.h>  
   
inline void* aligned_malloc(size_t size, size_t alignment)  
{  
    // 检查alignment是否是2^N  
    assert(!(alignment & (alignment - 1)));  
    // 计算出一个最大的offset，sizeof(void*)是为了存储原始指针地址  
    size_t offset = sizeof(void*) + (--alignment);  
   
    // 分配一块带offset的内存  
    char* p = static_cast<char*>(malloc(offset + size));  
    if (!p) return nullptr;  
   
    // 通过“& (~alignment)”把多计算的offset减掉  
    void* r = reinterpret_cast<void*>(reinterpret_cast<size_t>(p + offset) & (~alignment));  
    // 将r当做一个指向void*的指针，在r当前地址前面放入原始地址  
    static_cast<void**>(r)[-1] = p;  
    // 返回经过对齐的内存地址  
    return r;  
}  
   
inline void aligned_free(void* p)  
{  
    // 还原回原始地址，并free  
    free(static_cast<void**>(p)[-1]);  
}  
      </pre>
  </li>
  
  <h1><li>
      C++11中对内存对齐的操作</h1>
      <p>C++11标准里统一了内存对齐的相关操作。指定内存对齐使用alignas说明符：</p>
      <pre>
alignas(32) long long a = 0;  
   
#define XX 1  
struct alignas(XX) MyStruct_1 {}; // OK  
   
template <size_t YY = 1>  
struct alignas(YY) MyStruct_2 {}; // OK  
   
static const unsigned ZZ = 1;  
struct alignas(ZZ) MyStruct_3 {}; // OK  
      </pre>
      <p>注意到MyStruct_3编译是OK的。在C++11里，只要是一个编译期数值（包括static const）都支持alignas（the assignment-expression shall be an integral constant expression，参考ISO/IEC-14882:2011，7.6.2 Alignment specifier，第2款）。但是需要小心的是，目前微软的编译器（Visual C++ Compiler November 2013 CTP）在MyStruct_3的情况下仍然会报error C2057。</p>
      <p>另外，alignas同前面介绍的__declspec、__attribute__一样，只能改大不能改小（参考ISO/IEC-14882:2011，7.6.2 Alignment specifier，第5款）。如果需要改小，比如设置对齐为1的话，仍然需要使用#pragma pack。或者，可以使用C++11里#pragma的等价物_Pragma（微软暂不支持这个）：</p>
      <pre>
_Pragma("pack(1)")  
struct MyStruct  
{  
    char a;         // 1 byte  
    int b;          // 4 bytes  
    short c;        // 2 bytes  
    long long d;    // 8 bytes  
    char e;         // 1 byte  
};  
_Pragma("pack()")  
      </pre>
      <p>除了这些之外，alignas比__declspec、__attribute__强大的地方在于它还可以这样用：</p>
      <pre>
alignas(int) char c;  
      </pre>
      <p>这个char就按int的方式对齐了。获取内存对齐使用alignof操作符：</p>
      <pre>
MyStruct xx;  
std::cout << alignof(xx) << std::endl;  
std::cout << alignof(MyStruct) << std::endl;  
      </pre>
      <p>相关注意点和前面介绍的__alignof、__alignof__并无二致。除了alignas和alignof，C++11中还提供了几个有用的工具。</p>
      <p><a href='http://en.cppreference.com/w/cpp/types/alignment_of' target='_blank'>A. std::alignment_of</a></p>
      <p>功能是编译期计算类型的内存对齐。std里提供这个是为了补充alignof的功能。alignof只能返回一个size_t，而alignment_of则继承自std::integral_constant，因此拥有value_type、type、operator()等接口（或者说操作）。</p>
      <p><ahref='http://en.cppreference.com/w/cpp/types/aligned_storage' target='_blank'>B. std::aligned_storage</a></p>
      <p>这是个好东西。我们知道，很多时候需要分配一块单纯的内存块，比如new char[32]，之后再使用placement new在这块内存上构建对象：</p>
      <pre>
char xx[32];  
::new (xx) MyStruct;  
      </pre>
      <p>但是char[32]是1字节对齐的，xx很有可能并不在MyStruct指定的对齐位置上。这时调用placement new构造内存块，可能会引起效率问题或出错，这时我们应该使用std::aligned_storage来构造内存块：</p>
      <pre>
std::aligned_storage<sizeof(MyStruct), alignof(MyStruct)>::type xx;  
::new (&xx) MyStruct; 
      </pre>
      <p>需要注意的是，当使用堆内存的时候我们可能还是需要aligned_malloc。因为现在的编译器里new并不能在超出默认最大对齐后，还能保证内存的对齐是正确的。比如在MSVC 2013里，下面的代码：</p>
      <pre>
struct alignas(32) MyStruct  
{  
    char a;         // 1 byte  
    int b;          // 4 bytes  
    short c;        // 2 bytes  
    long long d;    // 8 bytes  
    char e;         // 1 byte  
};  
   
void* p = new MyStruct;  
// warning C4316: 'MyStruct' : object allocated on the heap may not be aligned 32  
      </pre>
      <p>将会得到一个编译警告。</p>
      <p><a href='http://en.cppreference.com/w/cpp/types/max_align_t' target='_blank'>C. std::max_align_t<a></p>
      <p>返回当前平台的最大默认内存对齐类型。malloc返回的内存，其对齐和max_align_t类型的对齐大小应当是一致的。</p>
      <p>我们可以通过下面这个方式获得当前平台的最大默认内存对齐数：</p>
      <pre>
std::cout << alignof(std::max_align_t) << std::endl;  
      </pre>
      <p><a href='http://en.cppreference.com/w/cpp/memory/align' target="_blank">D. std::align</a></p>
      <p>这货是一个函数，用来在一大块内存当中获取一个符合指定内存要求的地址。看下面这个例子：</p>
      <pre>
char buffer[] = "------------------------";  
void * pt = buffer;  
std::size_t space = sizeof(buffer) - 1;  
std::align(alignof(int), sizeof(char), pt, space);  
      </pre>
      <p>意思是，在buffer这个大内存块中，指定内存对齐为alignof(int)，找一块sizeof(char)大小的内存，并在找到这块内存后，将地址放入pt，将buffer从pt开始的长度放入space。</p>
      <p>关于这个函数的更多信息，可以参考<a href='http://www.cplusplus.com/reference/memory/align/' target="_blank">这里</a>。</p>
      <hr>
      <p>关于内存对齐，该说的就是这么多了。我们经常会看到内存对齐的应用，是在网络收发包中。一般用于发送的结构体，都是1字节对齐的，目的是统一收发双方（可能处于不同平台）之间的数据内存布局，以及减少不必要的流量消耗。</p>
      <p>C++11中为我们提供了不少有用的工具，可以让我们方便的操作内存对齐。但是在堆内存方面，我们很可能还是需要自己想办法。不过在平时的应用中，因为很少会手动指定内存对齐到大于系统默认的对齐数，所以倒也不比每次new/delete的时候都提心吊胆。</p>
      <hr>
  </li>
  <p>参考文章：</p>
  <ul>
    <li> <a href="http://en.wikipedia.org/wiki/Data_structure_alignment" target="_blank">Data structure alignment</a></li>
    <li> <a href="http://msdn.microsoft.com/en-us/library/ms253949.aspx" target="_blank">About Data Alignment</a></li>
    <li> <a href="http://msdn.microsoft.com/en-us/library/2e70t5y1.aspx" target="_blank">#pragma pack</a></li>
    <li> <a href="http://msdn.microsoft.com/en-us/library/83ythb65.aspx" target="_blank">align (C++)</a></li>
    <li> <a href="http://msdn.microsoft.com/en-us/library/45t0s5f4.aspx" target="_blank">__alignof Operator</a></li>
    <li> <a href="https://gcc.gnu.org/onlinedocs/gcc/Structure-Packing-Pragmas.html" target="_blank">6.57.8 Structure-Packing Pragmas</a></li>
    <li> <a href="https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Type-Attributes.html" target="_blank">5.32 Specifying Attributes of Types</a></li>
    <li> <a href="http://chuansu.iteye.com/blog/1487350" target="_blank">C/C++ Data alignment 及 struct size深入分析</a></li>
    <li> <a href="http://www.cnblogs.com/TenosDoIt/p/3590491.html" target="_blank">C++ 内存对齐</a></li>
    <li> <a href="http://www.cnblogs.com/lingjingqiu/p/3446457.html" target="_blank">结构/类对齐的声明方式</a></li>
    <li> <a href="http://www.cnblogs.com/alfredzzj/archive/2012/06/17/2552431.html" target="_blank">字节对齐（强制对齐以及自然对齐）</a></li>
    <li> <a href="http://blog.csdn.net/typhoonzb/article/details/4732520" target="_blank">malloc函数字节对齐很经典的问题</a></li>
    <li> <a href="http://blog.csdn.net/21aspnet/article/details/6729724" target="_blank">C语言字节对齐</a></li>
    <li> <a href="http://blog.csdn.net/xcl168/article/details/19269157" target="_blank">网络编程(9)内存对齐对跨平台通讯的影响</a></li>
    <li> <a href="http://stackoverflow.com/questions/16305311/usage-issue-of-stdalign" target="_blank">Usage Issue of std::align</a></li>
    <li> <a href="http://stackoverflow.com/questions/17378444/stdalign-and-stdaligned-storage-for-aligned-allocation-of-memory-blocks" target="_blank">std::align and std::aligned_storage for aligned allocation of memory blocks</a></li>
  </ul>
</ol>
  
