---
title: "Delphi源代码分析读书笔记"
featured_image: ""
description: "Delphi源代码分析读书笔记"
categories: "读书笔记"
tags: ["Delphi","程序设计"]
date: 2022-01-18T15:31:00+08:00
draft: false
---

## 第1章 最小化Delphi内核

## 第2章 基本数据类型的实现

+ 结构体变体类型，基本上没用过这种类型，以前对这个语法理解的迷迷糊糊的

~~~Delphi
TPerson = record
    FirstName, LastName: string[40];
    BirthDate: TDate;
    case Citizen: Boolean of
        True:
        (Birthplace: string[40]);
        False:
        (Country: string[20];
            EntryPort: string[20];
            EntryDate, ExitDate: TDate);
    end;
~~~

其中的Citizen相当于另外添加了一个Boolean字段，等价于

~~~Delphi
TPerson = record
    FirstName, LastName: string[40];
    BirthDate: TDate;
    Citizen: Boolean;
    case Boolean of
        True:
        (Birthplace: string[40]);
        False:
        (Country: string[20];
            EntryPort: string[20];
            EntryDate, ExitDate: TDate);
end;
~~~

可以在代码中使用Citizen，也可以不使用。Citizen完完全全的相当于一个独立字段，Delphi并不会自动关联或检查Citizen的值和实际存储的值，即是说可以给Citizen赋值为False, 同时给Birthplace赋值，或者给Citizen赋值为True,同时给Country、EntryPort等变量赋值。也可以只给Birthplace赋值，而完全不处理Citizen。这都是取决于程序的需要。当然在一般情况下，两者需要保持关联，赋值时保持一致，读取时根据类型来决定读哪些字段。实际第二段代码里的True、False是没有意义的，对程序员和编译器来说都没意义。第一段代码里的True、False对编译器来说也没有任何意义，但可以方便程序员理解代码。  
那怎么判断变体部分到底存的什么内容呢？简单来说无法判断。但是在实际使用过程中，可以根据上下文来确定取值。或者在有tag字段的时候根据tag值来判断，当然这只能依赖于写值时正确的保持了关联。

~~~Delphi
Int64Rec = packed record
    case Integer of
        0: (Lo, Hi: Cardinal);
        1: (Cardinals: array [0..1] of Cardinal);
        2: (Words: array [0..3] of Word);
        3: (Bytes: array [0..7] of Byte);
end;
~~~

~~~Delphi
function FileSeek(Handle: THandle; const Offset: Int64; Origin: Integer): Int64;
{$IFDEF MSWINDOWS}
begin
    Result := Offset;
    Int64Rec(Result).Lo := SetFilePointer(Handle, Int64Rec(Result).Lo,
        @Int64Rec(Result).Hi, Origin);
    if (Int64Rec(Result).Lo = $FFFFFFFF) and (GetLastError <> 0) then
        Int64Rec(Result).Hi := $FFFFFFFF;
end;
~~~

Int64Rec是在SysUtils中定义的类型，这个类型实际上提供了一些方便的方法来操作Int64变量的各个字节。Int64一共是8个字节，这8个字节分别可以按照8个Byte，4个Word，2个Cardinal来访问。Int64Rec主要是用于类型转换，因为在类型转换时可能需要单独分析各个字节的值。  
FileSeek的参数Offset是表示文件位置的偏移量，在Delphi中定义为Int64，但在Windows API中需要两个32位整数，低32位和高32位分开传参（这可能和早期C语言中没有Int64的类型有关？）。所以后面调用API时就转成Int64Rec，然后用Lo、Hi分别访问低32位和高32位。  
这个例子也解释了为什么可以不加tag，这里的用法实际上不是根据不同情况来保存不同的值，相反，这里保存的值一定是Int64，只是对Int64可以有多种不同的解释和使用方式。  
注意这里面的0、1、2、3实际是没有任何意义的，而且实际上这里还不能加tag，因为加了tag后相当于增加了一个字段，就和Int64结构不一致了。

+ 全局变量在应用程序的数据区分配，而局部变量在应用程序的栈上分配。因此，相对于定义的顺序，多个全局变量的地址是递增的，而局部变量递减。此外，由于每次调用函数时，栈顶可能发生变化，因此，局部变量的地址是变化的，而全局变量是不变的。全局变量在编译器就被决定，它存在于可执行模块（.EXE或.DLL等）的文件映像内部。因此，在载入文件的同时，全局变量的内存就被分配了。而局部变量是在例程被调用的时候才分配的。局部变量总是在栈上分配。

+ 动态分配的内存将在堆上进行分配。堆分配与栈分配不同。

+ 简单类型的变量可以用SizeOf()计算内存占用。  
字符串、动态数组、引用参数、无类型参数、过程、接口、对象等类型的变量实际是指针，这些变量由两个部分组成，指针本身和数据体。  
指针本身的内存大小可以用SizeOf()计算，一般是固定大小，32位程序是4字节，64位程序是8字节。  
“数据体”的大小取决于该变量的数据结构，通常有专用的或独立设计的函数来取其大小，如用Length()来取字符串或者数组的大小。  

+ 所有的变量名在汇编中都是一个整数，表示内存地址。  
简单变量，相当于汇编中的直接寻址，i表示变量的值，@i表示变量的地址。
指针变量，相当于汇编中的间接寻址，p或者p^表示数据体的值，@p表示变量的地址。
指针变量中保存的是数据体的首地址，可通过Integer(p)将地址强制类型转换为整数。

~~~Delphi
var
  s: string = '1234567890';
  i: Integer = 15;
  p: PInteger;
  a: array of Integer;

begin
  Writeln('s的值        : ', s);
  Writeln('s的地址      : ', IntToHex(Integer(@s), 8));
  Writeln('s指向的首地址: ', IntToHex(Integer(s), 8));
  Writeln;

  Writeln('i的值        : ', i);
  Writeln('i的地址      : ', IntToHex(Integer(@i), 8));
  Writeln;

  p := @i;
  Writeln('p的值        : ', p^);
  Writeln('p的地址      : ', IntToHex(Integer(@p), 8));
  Writeln('p指向的首地址: ', IntToHex(Integer(p), 8));
  Writeln;

  SetLength(a, 1);
  a[0] := 9;
  Writeln('a[0]的值     : ', a[0]);
  Writeln('a的地址      : ', IntToHex(Integer(@a), 8));
  Writeln('a指向的首地址: ', IntToHex(Integer(a), 8));
  Writeln('a指向的首地址: ', IntToHex(Integer(@a[0]), 8));
  Writeln;

  Readln;

end.
~~~

+ var, const，out参数都是引用参数，引用参数在实际传参时并不会将真正的值传递给例程，而是传递参数地址，参数实际是指针。

~~~delphi
function GetAddr(const aVar): string;
begin
  Result := IntToHex(Integer(@aVar), 8);
end;

function GetDataAddr(const aVar): string;
begin
  Result := IntToHex(Integer(aVar), 8);
end;
~~~

+ 真常量（非类型化的常量）作为立即数处理：将值直接编码到汇编指令中。因此它们不可能有内存占用，当然也不可能有具体的内存地址值。  
除了字符串和集合，Delphi中定义的常量都不能超过8字节（Int64）。否则必须声明成类型化的常量, 类型化的常量与全局变量在同一内存空间中被分配，且使用相同的分配规则。

+ 字符串常量与变量使用的是不同的内存空间。编译器总是将字符串常量与代码放在一起，并将它的首地址直接编码到指令中。  
对于字符串常量来说，并不需要类似于字符串变量的额外空间，因为字符串本身不会被修改，所需要的空间是固定的不会变化，也不需要存储长度（固定的）、引用计数（不需要释放）等。

+ 对于真常量和非类型化的字符串常量来说，取地址是没有意义的，它们实际上没有地址。

+ 字符串使用引用计数和写复制。

+ 集合类型与简单类型一样，没有动态分配的内存。集合中的每个元素占一个位，因为Delphi最大支持256个元素的集合，所以使用1~32个字节来存放各个位数据。集合类型的局部变量总在栈上分配。

+ 不论数组元素是什么类型，静态数组的局部变量总会在栈上分配。如果栈大小不够，将会导致异常。Delphi不会在编译器计算栈的耗用。

+ 动态数组变量只占用4个字节。动态数组在堆中动态分配内存。

+ 缺省情况下，记录会以**对齐**方式存储。记录总是直接分配内存。变体记录使用能够容纳可变部分最大长度的空间来存储。Delphi约定变体部分必须出现在记录定义的最末部分，这使得在计算和分配空间时，可以通过直接在记录后面附加空间的方式来实现。

+ 文件类型是基于记录类型来实现的，类型文件和无类型文件都是TFileRec，文本文件类型为TTextRec。

+ 指针总是占用4字节的内存（32位程序）。任何时候都可以将指针作为整型使用，并使用整型运算做地址运算。

+ 过程变量只占用4个字节（32位程序）的内存，它实际上是以指针形式存在的。  
过程变量只存储例程代码地址的入口地址，真实的例程代码只存在于应用程序的**代码段**中。尽管如此，Delphi采用一种与指针不同的语意来理解过程——因此，不能在指针与过程变量之间做强制转换。  
应用程序的代码通常是在编译器由编译程序生成的，一旦编译过程结束，例程代码也就确定了。代码段是只读的。

+ 动态数组的引用计数并非基于元素的，而是基于变量的，这与字符串的引用计数不一样：在字符串中，访问串中的一个字符，会导致引用计数发生变化；而在动态数组中，读写一个元素，并不会使引用关系发生变化。
字符串有写时复制的机制，动态数组没有。

+ 值参数的备份源自于例程的调用约定：值参数可以被改写，但是不会影响到传值的原始变量。因此，编译器需要为值参数制作一个备份，而无论代码中是否要写该值参数。这个复制操作在编译时就被决定了，例程被调用一次，就发生一次复制操作。

+ 使用const声明的值参数，编译器将向例程直接传入参数的地址，如果使用指针访问该地址，就可以对原值进行直接修改。

## 第3章 BASM精要

+ 

