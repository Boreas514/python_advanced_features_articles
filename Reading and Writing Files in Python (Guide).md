# 在python中读写文件（指南）
[原文链接](https://realpython.com/read-write-files-python/#what-is-a-file)

<!-- TOC -->

- [在python中读写文件（指南）](#在python中读写文件指南)
    - [文件是什么？](#文件是什么)
        - [文件路径](#文件路径)
        - [行尾结束符号](#行尾结束符号)
        - [字符编码](#字符编码)
    - [在python中打开和关闭文件](#在python中打开和关闭文件)
        - [文本文件类型](#文本文件类型)
        - [缓冲二进制文件类型](#缓冲二进制文件类型)
        - [原始文件类型](#原始文件类型)
    - [读写打开的文件](#读写打开的文件)
        - [在文件中迭代每一行](#在文件中迭代每一行)
        - [处理二进制字节](#处理二进制字节)
        - [一个完整的例子：`dos2unix.py`](#一个完整的例子dos2unixpy)
    - [提示和技巧](#提示和技巧)
        - [`__file__`](#__file__)
        - [追加写入](#追加写入)
        - [同时操作两个文件](#同时操作两个文件)
        - [创建自定义的上下文管理器](#创建自定义的上下文管理器)
    - [不要重复造轮子](#不要重复造轮子)
    - [你已经是出色的文件操作大师了！](#你已经是出色的文件操作大师了)

<!-- /TOC -->

使用python读写文件是一个常见的任务。不管是写一个文本文档，读取一个复杂的服务器日志亦或是分析原始二进制文件，这些情况都需要读写文件。

**在这篇教程中你会学到：**

* 什么构成了文件以及为什么这在python中很重要
* python读写文件基础
* 读写文件的一些基础场景

这篇教程主要面向想要进阶的新手玩家，但是这里也有一些高级开发者可以用到的小tips。

## 文件是什么？

在我们开始使用python操作文件之前，理解一个文件到底是什么以及现代操作系统如何处理它们非常重要。

本质上讲，一个文件是[用来存储数据](https://en.wikipedia.org/wiki/Computer_file)的连续的一组字节。这些数据使用一种特定的格式组织起来，不管是简单的文本文档还是复杂的可执行程序。最后这些字节文件为了更便于电脑处理，转化为二进制 `1` 或 `0`。

大多数现代文件系统上的文件有三个主要组成部分：

1. **Header**:文件的元数据（文件名，大小，类型等等）
2. **Data**:创建者或编辑者写入的文件主体内容
3. **End of file(EOF)**:表明文件结束的特定字符

![](https://files.realpython.com/media/FileFormat.02335d06829d.png)

这些数据的组织格式一般使用扩展名来表示。举个例子，一个有`.gif`扩展名的文件很可能是按照[Graphics Interchange Format](https://en.wikipedia.org/wiki/GIF)格式组织的。[这里](https://en.wikipedia.org/wiki/List_of_filename_extensions)有成百上千的扩展名。对于该教程，你只会涉及处理`.txt`和`.csv`文件。

### 文件路径

当你在操作系统上访问一个文件时，你需要一个文件路径。文件路径是表示文件所在位置的一段字符串。它可以被分解成三个主要部分：

1. **Folder Path**:文件夹所在位置，Unix中使用`/`分割，Windows中使用`\`分割
2. **File Name**:文件的名字
3. **Extension**:文件的扩展名，使用`.`分割

举个例子，假设你有如下的一个文件结构：
```
/
│
├── path/
|   │
│   ├── to/
│   │   └── cats.gif
│   │
│   └── dog_breeds.txt
|
└── animals.csv
```
假设说你想访问`cats.gif`，你当前的目录在 `/`，你需要穿过`path`和`to`文件夹，最终到达`cats.gif`文件。文件夹路径是`path/to/`。文件名字是`cats`。文件扩展名是`.gif`。所以完整路径是`path/to/cats.gif`。

现在假设你当前所在位置是`to`文件夹。你可以只是用文件名+扩展名`cats.gif`来代替绝对路径`path/to/cats.gif`。
```
/
│
├── path/
|   │
|   ├── to/  ← 你当前工作路径(cwd)
|   │   └── cats.gif  ← 访问这个文件
|   │
|   └── dog_breeds.txt
|
└── animals.csv
```
但要是想访问`dog_breeds.txt`呢？如何不使用绝对路径访问呢？你可以使用特殊字符 `..`来移动到上一级目录。所以你可以使用`../dog_breeds.txt`在`to`目录中访问`dog_breeds.txt`：
```
/
│
├── path/  ← Referencing this parent folder
|   │
|   ├── to/  ← Current working directory (cwd)
|   │   └── cats.gif
|   │
|   └── dog_breeds.txt  ← Accessing this file
|
└── animals.csv
```
`..`可以链式使用来返回多级目录之上。举个例子，在`to`文件夹中访问`animals.csv`，你可以使用`../../animals.csv`。

### 行尾结束符号

读写文件时总出现的一个问题是如何表示新一行内容开始或者一行内容的结束。换行符的历史可以追溯到摩斯密码时代，[在传输结束或一行结束之后发送一个特定的符号。](https://en.wikipedia.org/wiki/Prosigns_for_Morse_code#Official_International_Morse_code_procedure_signs)

然后，两个国际组织ISO和ASA推出了[标准化电传打字机。](https://en.wikipedia.org/wiki/Newline#History)ASA标准规定应该使用回车(`CR`或`\r`)和换行(`LF`或`\n`)连起来(`CR+LF`或`\r\n`)。然而ISO标准既允许使用`CR+LF`也允许只使用`LF`换行。

[Windows使用CR+LF](https://unix.stackexchange.com/a/411830)换行，Unix和比较新的Mac只使用`LF`字符。当你在一个和文件源不一样的平台上操作这个文件时就会有一些小问题。这有一个例子。我们假设文件`dog_breedx.txt`是创建在Win平台上：
```
Pug\r\n
Jack Russel Terrier\r\n
English Springer Spaniel\r\n
German Shepherd\r\n
Staffordshire Bull Terrier\r\n
Cavalier King Charles Spaniel\r\n
Golden Retriever\r\n
West Highland White Terrier\r\n
Boxer\r\n
Border Terrier\r\n
```
在Unix设备上的输出会有所不同：
```
Pug\r
\n
Jack Russel Terrier\r
\n
English Springer Spaniel\r
\n
German Shepherd\r
\n
Staffordshire Bull Terrier\r
\n
Cavalier King Charles Spaniel\r
\n
Golden Retriever\r
\n
West Highland White Terrier\r
\n
Boxer\r
\n
Border Terrier\r
\n
```
这会使遍历每行有问题，你应该把这样的情况考虑进去。

### 字符编码

另一个你可能遇到的问题是字节编码。一种编码是从字节数据到人类可读字符的一种转换。这通常是指定一个数值来表示一个字符。两种常见的编码方式是[ASCII](https://www.ascii-code.com/)和[UNICODE](https://unicode.org/)。[ASCII只可以存储128个字符](https://en.wikipedia.org/wiki/ASCII)，[Unicode可容纳1,114,112个字符]。

ASCII其实是Unicode（UTF-8）的一个子集，这意味着ASCII和Unicode共享相同字符的数值。需要注意的是使用错误的字符编码解析文件会导致严重错误。举个例子，加入一个文件使用UTF-8编码创建，然后你尝试使用ASCII解析它，如果文件中有128个值之外的字符，就会抛出一个错误。

## 在python中打开和关闭文件

当你想要操作文件时第一步是打开文件。这个使用[内置函数open()](https://docs.python.org/3/library/functions.html#open)完成。`open()`需要一个叫文件路径的参数。`open()`函数有一个返回值，[文件对象](https://docs.python.org/3/glossary.html#term-file-object)：
```python
file = open('dog_breeds.txt')
```
在你打开一个文件之后，你要学着如何关闭它。
```
警告：你必须永远确保打开的文件在合适的时机关闭。
```
记住！关闭文件永远是你的重大职责！大多数情况下，中止应用或脚本后你的文件最终会被关闭。然而不能保证这个情况何时会发生。这可能会导致未预期的行为比如资源泄露。同时这也是Pythonic的最佳实践，确保你的代码以定义良好的方式运行并减少任何未预期的行为。

当你操作文件时，有两种方法可以确保即使抛出了异常的情况下文件也会被恰当的关闭。第一种方法使用`try-finally`代码块：
```python
reader = open('dog_breeds.txt')
try:
    # Further file processing goes here
finally:
    reader.close()
```
第二种方法是使用`with`陈述：
```python
with open('dog_breeds.txt') as reader:
    # Further file processing goes here
```
`with`语句可以当程序流离开`with`代码块时自动关闭文件，即使代码块中出现了错误。我本人强烈建议你尽可能多的使用`with`，它让代码更干净并且你可以轻松的处理任何意外的错误。

同时你会用到第二个位置型参数，`mode`。这个参数是一个包含多个字符的字符串来表示你希望怎样打开这个文件。默认和最常用的是`r`，这代表通过只读方式打开这个文件：
```python
with open('dog_breeds.txt', 'r') as reader:
    # Further file processing goes here
```
其他的模式参考[在线文档](https://docs.python.org/3/library/functions.html#open)，但是最常用的几个选项如下：
```
Character	Meaning
'r'	        只读模式打开（默认）
'w'	        写入模式打开，覆盖之前的内容
'rb' or 'wb'	二进制模式打开（使用字节读/写）
```
现在我们回头看看文件对象是什么。一个文件对象是：

“an object exposing a file-oriented API (with methods such as read() or write()) to an underlying resource.” ([Source](https://docs.python.org/3/glossary.html#term-file-object))

暴露出面向文件API（使用`read()`或`write()`方法）来访问底层资源的一个对象。

文件对象有三种不同类别：

* Text files            文本文件
* Buffered binary files 缓冲二进制文件
* Raw binary files      原始二进制文件

每个文件类型都在`io`模块中被定义好了。下面是一个关于该部分内容的大纲。

### 文本文件类型

文本文件是经常要接触到的类型。这又几个如何打开文件的例子：
```python
open('abc.txt')

open('abc.txt', 'r')

open('abc.txt', 'w')
```
`open()`会返回一个`TextIOWrapper`文件对象：
```python
>>> file = open('dog_breeds.txt')
>>> type(file)
<class '_io.TextIOWrapper'>
```
这是`open()`返回的默认对象。

### 缓冲二进制文件类型

缓冲二进制文件类型用来读写二进制文件。下面是打开这类文件的例子：
```python
open('abc.txt', 'rb')

open('abc.txt', 'wb')
```
这时`open()`返回`BufferReader`或`BufferWriter`文件对象：
```python
>>> file = open('dog_breeds.txt', 'rb')
>>> type(file)
<class '_io.BufferedReader'>
>>> file = open('dog_breeds.txt', 'wb')
>>> type(file)
<class '_io.BufferedWriter'>
```

### 原始文件类型
一个原始文件类型是：

“通常作为二进制和文本流的底层构筑块。”（[资料](https://docs.python.org/3.7/library/io.html#raw-i-o)）

因此通常不使用这个东西。

以下是如何这样打开文件的例子：
```python
open('abc.txt', 'rb', buffering=0)
```
操作这种文件时`open()`会返回一个`FileIO`文件对象：
```python
>>> file = open('dog_breeds.txt', 'rb', buffering=0)
>>> type(file)
<class '_io.FileIO'>
```

## 读写打开的文件

一旦你打开了一个文件后你就要读写这个文件了。首先来读取一个文件。这里有很多种方法可以在一个文件对象上调用，帮助你读取文件：

| 方法 | 该方法的功能 |
|--|--|
| `.read(size=-1)` | 从文件中读取`size`个字节长的数据。如果不传参数或参数为`None`或`-1`，则读取整个文件 |
| `.readline(size=-1)` | 从一行中读取最大`size`个字符。This continues to the end of the line and then wraps back around.如果不传参数或参数为`None`或`-1`，则会读取整行。（或该行剩下的部分） |
| `.readlines()` | 这个方法读取文件对象剩余的全部行然后作为一个字典返回。|

还是用`dog_breeds.txt`这个文件举例，让我们通过一些例子来使用这些方法。下面是使用`.read()`读取整个文件的例子：
```python
>>> with open('dog_breeds.txt', 'r') as reader:
>>>     # 读取 & 打印整个文件
>>>     print(reader.read())
Pug
Jack Russel Terrier
English Springer Spaniel
German Shepherd
Staffordshire Bull Terrier
Cavalier King Charles Spaniel
Golden Retriever
West Highland White Terrier
Boxer
Border Terrier
```
使用`.readline()`每次读取五个字节的例子：
```python
>>> with open('dog_breeds.txt', 'r') as reader:
>>>     # Read & print the first 5 characters of the line 5 times
>>>     print(reader.readline(5))
>>>     # Notice that line is greater than the 5 chars and continues
>>>     # down the line, reading 5 chars each time until the end of the
>>>     # line and then "wraps" around
>>>     print(reader.readline(5))
>>>     print(reader.readline(5))
>>>     print(reader.readline(5))
>>>     print(reader.readline(5))
Pug

Jack
Russe
ll Te
rrier
```
使用`.readlines()`读取整个文件到一个list的例子：
```python
>>> f = open('dog_breeds.txt')
>>> f.readlines()  # Returns a list object
['Pug\n', 'Jack Russel Terrier\n', 'English Springer Spaniel\n', 'German Shepherd\n', 'Staffordshire Bull Terrier\n', 'Cavalier King Charles Spaniel\n', 'Golden Retriever\n', 'West Highland White Terrier\n', 'Boxer\n', 'Border Terrier\n']
```
上述例子也可以在文件对象外使用`list()`来完成：
```python
>>> f = open('dog_breeds.txt')
>>> list(f)
['Pug\n', 'Jack Russel Terrier\n', 'English Springer Spaniel\n', 'German Shepherd\n', 'Staffordshire Bull Terrier\n', 'Cavalier King Charles Spaniel\n', 'Golden Retriever\n', 'West Highland White Terrier\n', 'Boxer\n', 'Border Terrier\n']
```

### 在文件中迭代每一行

读取文件的一个基本操作就是迭代每一行，下面是使用`.readline()`实现迭代的一个例子：
```python
>>> with open('dog_breeds.txt', 'r') as reader:
>>>     # Read and print the entire file line by line
>>>     line = reader.readline()
>>>     while line != '':  # The EOF char is an empty string
>>>         print(line, end='')
>>>         line = reader.readline()
Pug
Jack Russel Terrier
English Springer Spaniel
German Shepherd
Staffordshire Bull Terrier
Cavalier King Charles Spaniel
Golden Retriever
West Highland White Terrier
Boxer
Border Terrier
```
另一种迭代每一行的方法是使用`.readlines()`方法，`.readlines()`返回一个一个元素是文件一行的list：
```python
>>> with open('dog_breeds.txt', 'r') as reader:
>>>     for line in reader.readlines():
>>>         print(line, end='')
Pug
Jack Russell Terrier
English Springer Spaniel
German Shepherd
Staffordshire Bull Terrier
Cavalier King Charles Spaniel
Golden Retriever
West Highland White Terrier
Boxer
Border Terrier
```
我们还可以通过遍历文件对象本身来简化上面的例子：
```python
>>> with open('dog_breeds.txt', 'r') as reader:
>>>     # Read and print the entire file line by line
>>>     for line in reader:
>>>         print(line, end='')
Pug
Jack Russel Terrier
English Springer Spaniel
German Shepherd
Staffordshire Bull Terrier
Cavalier King Charles Spaniel
Golden Retriever
West Highland White Terrier
Boxer
Border Terrier
```
最后一种实现是更Pythonic的，它更快一级更加高效的使用内存。所以建议你以后的代码用这种方式来实现。

**注意：**上述的一些例子包含`print('some text', end='')`。`end=''`是用来让python只打印从文件里独到的内容，防止python打印额外的换行符换行。

现在让我们进入到写文件。就像读文件一样，文件对象有几种方法用来写文件：

| 方法 | 该方法的功能 |
| -- | -- |
| `.write(string)` | 写入字符串到文件 |
| `.writelines(seq)` | 写入一个序列到文件中。不会自动向序列每一项添加换行符，你要自己添加适当的换行符。 |  

这里有几个快速使用`.write()`和`.writelines()`的例子：
```python
with open('dog_breeds.txt', 'r') as reader:
    # Note: readlines doesn't trim the line endings
    dog_breeds = reader.readlines()

with open('dog_breeds_reversed.txt', 'w') as writer:
    # Alternatively you could use
    # writer.writelines(reversed(dog_breeds))

    # Write the dog breeds to the file in reversed order
    for breed in reversed(dog_breeds):
        writer.write(breed)
```
### 处理二进制字节
有时你可能需要处理[字节字符串](https://docs.python.org/3.7/glossary.html#term-bytes-like-object)。通过在`mode`参数上加上`'b'`字符来实现这个操作。所有应用在文件对象上的方法也在二进制字符上适用。
但是需要注意的是每个方法期待和返回都是一个字节对象：
```python
>>> with open(`dog_breeds.txt`, 'rb') as reader:
>>>     print(reader.readline())
b'Pug\n'
```
可以看出来用`b`参数打开一个文本文档不是那么好玩。现在我们有一个狗的图片：
![](https://files.realpython.com/media/jack_russell.92348cb14537.png)

你可以真的在python中打开这个文件检查里面的内容！因为[.png文件格式](https://en.wikipedia.org/wiki/Portable_Network_Graphics)有明确定义，文件头是如下的八位字节：

| 值 | 含义 |
|--|--|
| 0x89 | 一个用来表示这是一个PNG开始的神奇数字 |
| 0x50 0x4E 0x47  | PNG的ASCII |
| 0x0D 0x0A | 一个DOS风格的换行符\r\n |
| 0x1A | 一个DOS风格的EOF字符 |
| 0x0A | 一个Unix风格的换行符\n |
果然，当你打开文件单独读这些字节时，你可以看到他们确实组成了`.png`的文件头：
```python
>>> with open('jack_russell.png', 'rb') as byte_reader:
>>>     print(byte_reader.read(1))
>>>     print(byte_reader.read(3))
>>>     print(byte_reader.read(2))
>>>     print(byte_reader.read(1))
>>>     print(byte_reader.read(1))
b'\x89'
b'PNG'
b'\r\n'
b'\x1a'
b'\n'
```
### 一个完整的例子：`dos2unix.py`

现在我们把全部知识汇总起来实现一个完整的读写文件的例子。这个例子很像工具[dos2unix](https://en.wikipedia.org/wiki/Unix2dos)，它把文件中行尾的`\r\n`转换为`\n`。

这个工具分为三部分组成，第一部分是`str2unix()`，该部分会把`\\r\\n`转换为`\\n`。第二部分是`dos2unix()`，它把字符串中包含的`\r\n`转换为`\n`。最后是入口区块`__main__`，它只在文件作为脚本执行时调用。可以把它看作是其他编程语言中的主函数。
```python
"""
A simple script and library to convert files or strings from dos like
line endings with Unix like line endings.
"""

import argparse
import os


def str2unix(input_str: str) -> str:
    r"""\
    Converts the string from \r\n line endings to \n

    Parameters
    ----------
    input_str
        The string whose line endings will be converted

    Returns
    -------
        The converted string
    """
    r_str = input_str.replace('\r\n', '\n')
    return r_str


def dos2unix(source_file: str, dest_file: str):
    """\
    Coverts a file that contains Dos like line endings into Unix like

    Parameters
    ----------
    source_file
        The path to the source file to be converted
    dest_file
        The path to the converted file for output
    """
    # NOTE: Could add file existence checking and file overwriting
    # protection
    with open(source_file, 'r') as reader:
        dos_content = reader.read()

    unix_content = str2unix(dos_content)

    with open(dest_file, 'w') as writer:
        writer.write(unix_content)


if __name__ == "__main__":
    # Create our Argument parser and set its description
    parser = argparse.ArgumentParser(
        description="Script that converts a DOS like file to an Unix like file",
    )

    # Add the arguments:
    #   - source_file: the source file we want to convert
    #   - dest_file: the destination where the output should go

    # Note: the use of the argument type of argparse.FileType could
    # streamline some things
    parser.add_argument(
        'source_file',
        help='The location of the source '
    )

    parser.add_argument(
        '--dest_file',
        help='Location of dest file (default: source_file appended with `_unix`',
        default=None
    )

    # Parse the args (argparse automatically grabs the values from
    # sys.argv)
    args = parser.parse_args()

    s_file = args.source_file
    d_file = args.dest_file

    # If the destination file wasn't passed, then assume we want to
    # create a new file based on the old one
    if d_file is None:
        file_path, file_extension = os.path.splitext(s_file)
        d_file = f'{file_path}_unix{file_extension}'

    dos2unix(s_file, d_file)
```

## 提示和技巧

现在你已经掌握了读写文件的基础，这里有一些提示和技巧助你的技术更加精进。

### `__file__`

（译者：本部分建议自己试一下，我自己尝试没得到和文章一样的结果，我定义两个文件a.py和b.py，然后在a.py中import b.py，不管我把b.py放到哪，b.py中的__file__都打印该文件在系统中的绝对路径，如能解惑可以发一个issue）
`__file__`是模块的一个[特殊属性](https://docs.python.org/3/reference/datamodel.html)，就像`__name__`一样。它是：
被加载模块的文件路径，如果它是被一个文件加载的。

**注意：**`__file__`返回关于Python脚本最初被调用路径的相对路径。如果你需要完整系统路径，你可以用`os.getcwd()`获取运行时代码所在的当前工作路径。

以下是几个例子。我一部分的工作是对硬件设备进行多项测试。对于每项测试我都用Python写一个测试脚本。这些脚本在执行时会使用`__file__`属性打印它们的工作状态。下面是文件结构：
```
project/
|
├── tests/
|   ├── test_commanding.py
|   ├── test_power.py
|   ├── test_wireHousing.py
|   └── test_leds.py
|
└── main.py
```

运行`main.py`产生一下输出：
```
>>> python main.py
tests/test_commanding.py Started:
tests/test_commanding.py Passed!
tests/test_power.py Started:
tests/test_power.py Passed!
tests/test_wireHousing.py Started:
tests/test_wireHousing.py Failed!
tests/test_leds.py Started:
tests/test_leds.py Passed!
```

这样我就可以通过`__file__`特殊属性动态地获取全部测试状态。

### 追加写入

有时你可能需要向一个文件末尾追加写入一些东西。通过在`mode`参数上使用`a`字符串来实现：
```python
with open('dog_breeds.txt', 'a') as a_writer:
    a_writer.write('\nBeagle')
```

现在检查`dog_breeds.txt`，你能看到文件开头没变，`Beagle`已经被写入文件末尾：
```python
>>> with open('dog_breeds.txt', 'r') as reader:
>>>     print(reader.read())
Pug
Jack Russell Terrier
English Springer Spaniel
German Shepherd
Staffordshire Bull Terrier
Cavalier King Charles Spaniel
Golden Retriever
West Highland White Terrier
Boxer
Border Terrier
Beagle
```

### 同时操作两个文件

有时你可能想同时读一个文件，并将内容写入另外一个文件。如果你想用上面写入文件的例子，其实可以合并一下：
```python
d_path = 'dog_breeds.txt'
d_r_path = 'dog_breeds_reversed.txt'
with open(d_path, 'r') as reader, open(d_r_path, 'w') as writer:
    dog_breeds = reader.readlines()
    writer.writelines(reversed(dog_breeds))
```

### 创建自定义的上下文管理器

有时你可能需要将文件对象做成一个自定义的类来更好地操纵它。当你这样做时记得添加给类添加几个魔术方法，要么`with`语句会不可用：`__enter__`和`__exit__`。加入这两个方法之后，你创建的类被叫做[上下文管理器(context manager)](https://docs.python.org/3/library/stdtypes.html#typecontextmanager)

当`with`陈述被调用时进入`__enter__()`方法，当程序退出`with`陈述代码块时`__exit__()`被调用。

这有一个模板，你可以用来生成自己的定制类：
```python
class my_file_reader():
    def __init__(self, file_path):
        self.__path = file_path
        self.__file_object = None

    def __enter__(self):
        self.__file_object = open(self.__path)
        return self

    def __exit__(self, type, val, tb):
        self.__file_object.close()

    # Additional methods implemented below
```

现在你的自定义类已经是上下文管理器了，你可以像使用内置函数`open()`一样使用这个类：
```python
with my_file_reader('dog_breeds.txt') as reader:
    # Perform custom class operations
    pass
```

下面是一个例子。还记得那条狗的照片吗？你想打开其他的`.png`文件但是不想每次打开时手动解析一下文件头。这有一个例子告诉你怎么做，这个例子还使用了自定义的迭代器，如果你对迭代器不熟，参考这篇文章[Python迭代器](https://dbader.org/blog/python-iterators)：
```python
class PngReader():
    # Every .png file contains this in the header.  Use it to verify
    # the file is indeed a .png.
    _expected_magic = b'\x89PNG\r\n\x1a\n'

    def __init__(self, file_path):
        # Ensure the file has the right extension
        if not file_path.endswith('.png'):
            raise NameError("File must be a '.png' extension")
        self.__path = file_path
        self.__file_object = None

    def __enter__(self):
        self.__file_object = open(self.__path, 'rb')

        magic = self.__file_object.read(8)
        if magic != self._expected_magic:
            raise TypeError("The File is not a properly formatted .png file!")

        return self

    def __exit__(self, type, val, tb):
        self.__file_object.close()

    def __iter__(self):
        # This and __next__() are used to create a custom iterator
        # See https://dbader.org/blog/python-iterators
        return self

    def __next__(self):
        # Read the file in "Chunks"
        # See https://en.wikipedia.org/wiki/Portable_Network_Graphics#%22Chunks%22_within_the_file

        initial_data = self.__file_object.read(4)

        # The file hasn't been opened or reached EOF.  This means we
        # can't go any further so stop the iteration by raising the
        # StopIteration.
        if self.__file_object is None or initial_data == b'':
            raise StopIteration
        else:
            # Each chunk has a len, type, data (based on len) and crc
            # Grab these values and return them as a tuple
            chunk_len = int.from_bytes(initial_data, byteorder='big')
            chunk_type = self.__file_object.read(4)
            chunk_data = self.__file_object.read(chunk_len)
            chunk_crc = self.__file_object.read(4)
            return chunk_len, chunk_type, chunk_data, chunk_crc
```
现在你可以使用自定义的上下文管理器打开和解析`.png`文件了：
```python
>>> with PngReader('jack_russell.png') as reader:
>>>     for l, t, d, c in reader:
>>>         print(f"{l:05}, {t}, {c}")
00013, b'IHDR', b'v\x121k'
00001, b'sRGB', b'\xae\xce\x1c\xe9'
00009, b'pHYs', b'(<]\x19'
00345, b'iTXt', b"L\xc2'Y"
16384, b'IDAT', b'i\x99\x0c('
16384, b'IDAT', b'\xb3\xfa\x9a$'
16384, b'IDAT', b'\xff\xbf\xd1\n'
16384, b'IDAT', b'\xc3\x9c\xb1}'
16384, b'IDAT', b'\xe3\x02\xba\x91'
16384, b'IDAT', b'\xa0\xa99='
16384, b'IDAT', b'\xf4\x8b.\x92'
16384, b'IDAT', b'\x17i\xfc\xde'
16384, b'IDAT', b'\x8fb\x0e\xe4'
16384, b'IDAT', b')3={'
01040, b'IDAT', b'\xd6\xb8\xc1\x9f'
00000, b'IEND', b'\xaeB`\x82'
```

## 不要重复造轮子

以下是几个你操作文件时的常见情况，这些情况大多可以靠其他库处理。有两种常见的文件类型`.csv`和`.json`，*Real Python*已经给出了处理他们的绝妙指南：
* [Python读写CSV文件](https://realpython.com/python-csv/)
* [Python操作JSON数据](https://realpython.com/python-json/)

此外，还有几个能帮到你的内置库：
* `wave:`读写WAV文件（音频）
* `aifc:`读写AIFF和AIFC文件（音频）
* `sunau:`读写Sun AU文件
* `tarfile:`读写tar文件
* `zipfile:`操作ZIP文件
* `configparser:`简单创建和解析配置文件
* `xml.etree.ElementTree:`创建和读取XML格式文件
* `msilib:`读写微软安装文件
* `plistlib:`生成和解析Mac OS X .plist文件

此外，PyPI上还有更多的第三方工具，以下是一些热门的：
* `PyPDF2:`PDF工具集
* `xlwings:`读写Excel文件
* `Pillow:`读取和操纵图片

## 你已经是出色的文件操作大师了！

你做到了!现在你知道了如何使用Python处理文件，包括一些高级技术。现在用Python处理文件应该比以前更容易了，而且当你开始这样做时，会有一种成就感。

在本教程中，你已经学会了：
* 什么是文件
* 如何正确地打开和关闭文件
* 如何读写文件
* 处理文件时的一些高级技术
* 一些用于处理常见文件类型的库
