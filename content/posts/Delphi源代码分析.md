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
FileSeek的参数Offset是表示文件位置的偏移量，在Delphi中定义为Int64，但在Windows API中需要两个32位整数，低32位和高32位分开传参。（这可能和早期C语言中没有Int64的类型有关？）所以后面调用API时就转成Int64Rec，然后用Lo、Hi分别访问低32位和高32位。
这个例子也解释了为什么可以不加tag，这里的用法实际上不是根据不同情况来保存不同的值，相反，这里保存的值一定是Int64，只是对Int64可以有多种不同的解释和使用方式。  
注意这里面的0、1、2、3实际是没有任何意义的，而且实际上这里还不能加tag，因为加了tag后相当于增加了一个字段，就和Int64结构不一致了。

+
