# easy_re

## 题目简述

题目是一个把 Perl 打包进 Windows x64 EXE 的逆向题。程序运行时会初始化 Perl 运行环境，从 PE 资源或文件尾部定位 BFS/Perl 打包资源，经过异或、文件表解析、zlib 解压或再次异或后还原出内部 Perl 文件，最终 flag 藏在解包出的 `perl.pl` 中。解题重点是静态分析资源定位、打包文件结构和两类解密方式，而不是动态 OD 调试。

## 解题过程

如果不用od来做的话   是一个很有意思的题，这里给了一个纯ida 静态分析的做法

### 第一步：确定资源文件位置并解密

前置知识点：

将perl打包成exe文件，只要让exe运行时加载起来配置好perl的运行库，那就可以让exe运行perl代码,那么本道赛题的核心目的就应该是分析exe加载过程，找到perl代码，解出flag。

首先64位程序，ida打开逆向分析main函数

上面一部分的函数主要是获取文件环境，以及main函数参数，注释有init的地方是初始化，exe运行perl代码肯定需要配置好perl环境，所以初始化的地方是重要的，下面一部分是直接run执行以及程序的释放内存的处理代码。那么关键点就是初始化和运行部分了。

那么先看初始化的函数：

上面全是初始化的过程，但是在这个函数中有着一个paperl的输出，这样子应该就是在初始化perl的运行环境了，那么继续往下看

在这个sub_40F730函数中，为paperl创建了一个0x7C0的堆区空间，并且初始化为0，然后，根据报错信息，我们可以知道bfs节段无法被找到，那么就代表，sub_406BF0函数,就是在寻找bfs节区，我们进入到该函数

在这里，我们发现一个关键函数，就是FindResourceA函数，根据百度的理解，这个函数拥有查找资源的功能，那么这里如果有人曾经用过查找资源的软件：Resource Hacker，就可以直接使用该软件找到BFS相关的资源。

这里的FindResourceA，LoadResource，LockResource三个函数常常是组合在一起使用的，用来寻找到资源以及定位资源，数据结构如下：

最左边的指针就代表了资源文件的起始字符串magic字符串的位置点。这类思想，在固件解包分析中比较常用，虽然被应用到了windows64位程序上，但是定位资源文件原理是一样的。

>其中0x200000这个数据总是会被比较，是因为对于不同操作系统内容来说，有些可执行文件的载入是大端序，有些是小端序，所以常常设置标志位0x200000作为文件的标识，用来识别大端序和小端序

并且在查找资源后，有一段校验的函数代码，这里是更加核心的确认了资源文件的位置。

这个函数的主要功能就是找到，文件的开始部分，进行校验，然后找到文件头，最后将文件头之后的数据和偏移第8位的4个字节进行异或操作，那么这里就是第一重解密，资源文件会根据某一个字符串进行异或，并且加载在堆区中。

资源文件文件头结构如下：

0-4： 文件标识位置，文件头

4-8： 文件小端序大端序标识

8-12： 被异或的数据也是整个资源文件的长度

12-16： 文件头的长度

那么解密代码如下：

```plain
def decypt_perl_code_bin_xor(perl_code_bin):
    perl_code_bin_total_len = struct.unpack('<I',perl_code_bin[8:12])[0]
    # print "perl_code_bin_total_len is %s" % hex(perl_code_bin_total_len)
    perl_code_bin_header_len = struct.unpack('<H',perl_code_bin[12:14])[0]
    # print "perl_code_bin_header_len is %s" % hex(perl_code_bin_header_len)
    perl_code_bin_data_len = perl_code_bin_total_len - perl_code_bin_header_len
    # print "perl_code_bin_data_len is %s" % hex(perl_code_bin_data_len)
    perl_code_orig =  perl_code_bin[perl_code_bin_header_len:perl_code_bin_total_len]
    if perl_code_orig == "":
        print "decypt_perl_code_bin_xor: perl_code_orig error"
        return 0
    i = 0
    decypt_xor_data = ''
    while(i < perl_code_bin_data_len):
        t = struct.unpack('<I',perl_code_orig[i:i+4])[0] ^ struct.unpack('<I',perl_code_bin[8:12])[0]
        decypt_xor_data = decypt_xor_data + struct.pack('I', t)
        i = i + 4
    with open('bfs_file', 'ab') as fout:
      fout.write(decypt_xor_data)
    return decypt_xor_data

fd = open(filename, 'rb')
orig_data = fd.read()
fd.close()
if orig_data[0x138F50:0x138F54] == '\xBC\x55\x21\xAB':
    # print struct.unpack('<I', orig_data[0x138F50:0x138F54])[0]
    print '[+] Found End Magic'
else:
    print '[-] Not Found End Magic'
    return False
perl_code_bin_total_len = struct.unpack('<I', orig_data[0x138F58:0x138F5C])[0]
if orig_data[0x138F50-perl_code_bin_total_len:0x138F54-perl_code_bin_total_len] == '\x7F\x3A\x5B\xAA':
    print '[+] Found Begin Magic'
else:
    print '[-] Not Found Begin Magic'
    return False
xor_perl_code_bin = orig_data[0x138F50-perl_code_bin_total_len:0x138F50]
# print "bin len is %s" % hex(perl_code_bin_total_len)
if xor_perl_code_bin == "":
    print "error"
print '[+] Decypt 1st for perl_code_bin'
# print xor_perl_code_bin
perl_code_bin = decypt_perl_code_bin_xor(xor_perl_code_bin)
if perl_code_bin == "":
    print "error"
return 0
```
那么继续分析发现，导出的数据的头部正好是 `BFS`，这里就确认了大概perl的资源文件位置会在这里。

### 第二步： 分析数据结构并解密得到文件列表

继续往下分析,上面已经得到了部分的被加密的资源文件，并且它存放在堆区之中。回到sub_40F730的函数中，上面的bfs选定节段已经分析完了，没啥东西了，下面这个函数，看参数runlib，猜测是载入库的过程，那么开始看吧

进入到sub_40D1A0函数中：

根据注释，我们可以理解到，后面有个extract提取的过程，但是先不能着急，先把上面的函数分析完。。。

在sub_407290函数中，在这个函数中，会创建一个结构体，并且导出（结构体结构，后续分析），经过比较长时间的分析，我们会发现，我们之前导出的资源文件是这样子的数据结构

0-4： 文件头标识

4-8： 文件小端序大端序标识

8-12： 整个文件的长度

12-16： 结尾标志

16-20： 标志位控制程序流程

20-22： 这两个字节代表了文件头长度，也就是0x20个字节

22-24：标志位

24-28：文件列表长度

28-32：结尾标志

那么我们进入到while循环的sub_406FB0函数分析一波：

发现程序就经过异或判断，来判断数据是否已经真实载入内存了。所以我们异或测试一波，会发现文件列表会是这样子的结构体。

我们发现，从0x20开始的数据，会通过异或0xEA，将一个一个文件给剥离出来：

每个文件数据结构如下：

0-2： 文件名称长度为7

2-9： 文件名称（这里根0-2字节确定文件名称长度）

9-12： 被#填充（这个填充字节长度根据4个倍数来判断）

12-16：文件的结构体的标识位，代表真实文件。例如：第一个文件Carp.pm的压缩包的文件头（为什么是压缩包，这个后面讲解）

找到文件列表的代码如下：

```plain
def extract_files_list(files_tables, files_tables_len):
    print hex(files_tables_len)
    i = 0
    files_list = []
    while(i < files_tables_len):
        filename_len = struct.unpack('<H', files_tables[i:i+2])[0]
        # print hex(filename_len)
        j = 0
        filename = ''
        while(j != filename_len):
            filename += struct.pack('B', (ord(files_tables[i+2+j:i+2+j+1]) ^ 0xEA))
            j = j + 1
        k = 0
        # print filename
        if (filename_len+2) % 4 != 0:
            k = 4 - ((filename_len+2) % 4)
        # while(files_tables[i+2+filename_len+k] == '#'):
        #   k = k + 1
        filedata_offset = struct.unpack('<I', files_tables[i+2+filename_len+k:i+2+filename_len+k+4])[0]
        # print filename_len
        # print filename
        print filename_len, filename, hex(filedata_offset)
        files_list.append([filename_len,filename,filedata_offset])
        i = i + 2 + filename_len + k + 4
    return files_list
file_offset = struct.unpack('<H', perl_code_bin[20:22])[0]
    files_tables_end_offset = struct.unpack('<I', perl_code_bin[24:28])[0]
    files_tables = perl_code_bin[file_offset:files_tables_end_offset]
    files_list = extract_files_list(files_tables, files_tables_end_offset-32)
```
文件列表如下：长度 文件名 文件索引。列表中可以看到 `Capture.pm`、`Exporter.pm`、`AUTOLOAD.pm`、`Carp.pm`、`Config.pm`、`DynaLoader.pm`、`Fcntl.pm`、`perl.pl` 等 Perl 运行库和脚本文件。

### 第三步：根据文件列表并解压或者解密出文件

回到之前，这个程序会返回一个结构体到最后一个参数，然后会被sub_406D40函数调用。。。当我们分析完sub_406D40函数之后，我们可以知道：

其实就是真实文件的文件结构体：

从第一个例子开始分析，也就是从0x3B4位置开始分析,我们可以得知：

0 0x00 -->  file_len   // 原始文件长度

1 0x04 -->  compress_len  // 如果 decypt_method==0x02，则 compress_len = file_len

2 0x08 -->  decypt_method  // 0x03 --> 解压，0x02 --> 只异或 0xEA

3 0x0c -->  data

结合上面两个图，我们发现在sub_406D40函数中，会根据结构体的第三个参数，来判定是解压缩还是异或，为什么我知道是zlib解压缩呢？因为之前我用c写过zlib的代码，第二个参数是很标准的1.2.8代表，是zlib的inflateInit_函数的第二个参数。

ok那么这一步，就是将文件解压或者用异或解密出来的代码：

```plain
def extract_files(input_file,files_list, perl_code_bin):
    i = 0
    while(i < len(files_list)):
        filename = files_list[i][1]
        filedata_offset = files_list[i][2]
        filedata_dec_len = struct.unpack('<I', perl_code_bin[filedata_offset:filedata_offset+4])[0]
        filedata_enc_len = struct.unpack('<I', perl_code_bin[filedata_offset+4:filedata_offset+8])[0]
        filedata_dec_method = struct.unpack('<I', perl_code_bin[filedata_offset+8:filedata_offset+12])[0]
        if (filedata_dec_len == filedata_enc_len) and (filedata_dec_method == 0x02):
            print '[+] XOR  -->  {}'.format(filename)
            j = 0
            filedata = ''
            while (j != filedata_dec_len):
                filedata += struct.pack('B', (ord(perl_code_bin[filedata_offset+12+j:filedata_offset+12+j+1]) ^ 0xEA))
                j = j + 1
            filename = filename.replace('*','_').replace('<','-')
            if not os.path.exists('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe")):
                os.makedirs('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe"))
            if '/' in filename and not os.path.exists('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0])):
                os.makedirs('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0]))
            fd = open('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\')), 'wb+')
            fd.write(filedata)
            fd.close()
            i = i + 1
        elif (filedata_dec_len != filedata_enc_len) and (filedata_dec_method == 0x03):
            print '[+] ZLIB  -->  {}'.format(filename)
            enc_filedata = perl_code_bin[filedata_offset+12:filedata_offset+12+filedata_enc_len]
            dec_filedata = zlib.decompress(enc_filedata)
            j = 0
            filedata = ''
            while (j != filedata_dec_len):
                filedata += struct.pack('B', (ord(dec_filedata[j:j+1]) ^ 0xEA))
                j = j + 1
            filename = filename.replace('*','_').replace('<','-')
            if not os.path.exists('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe")):
                os.makedirs('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe"))
            if '/' in filename and not os.path.exists('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0])):
                os.makedirs('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0]))
            fd = open('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\')), 'wb+')
            fd.write(filedata)
            fd.close()
            i = i + 1
        else:
            print '[-] Unknow  -->  {}'.format(filename)
            i = i + 1

    return True
ret = extract_files(filename, files_list, perl_code_bin)
```
但是大家可以看看为什么我代码中在解压之后，还会异或一次0xEA呢？
>如果大家没有异或0xEA的话， 其实解压缩出来还是乱码。。。

这一段异或，其实是在main函数的run的部分（我在最上面的截图里面有注释），因为run部分要讲解起来很麻烦，太多了，这里，建议可以直接脑洞异或0xEA，解压缩出来的数据很多都是0xEA，所以可以作为脑洞。

### 完整exp

```plain
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import os
import struct
import sys
import zlib
import binascii
def decypt_perl_code_bin_xor(perl_code_bin):
    perl_code_bin_total_len = struct.unpack('<I',perl_code_bin[8:12])[0]
    # print "perl_code_bin_total_len is %s" % hex(perl_code_bin_total_len)
    perl_code_bin_header_len = struct.unpack('<H',perl_code_bin[12:14])[0]
    # print "perl_code_bin_header_len is %s" % hex(perl_code_bin_header_len)
    perl_code_bin_data_len = perl_code_bin_total_len - perl_code_bin_header_len
    # print "perl_code_bin_data_len is %s" % hex(perl_code_bin_data_len)
    perl_code_orig =  perl_code_bin[perl_code_bin_header_len:perl_code_bin_total_len]
    if perl_code_orig == "":
        print "decypt_perl_code_bin_xor: perl_code_orig error"
        return 0
    i = 0
    decypt_xor_data = ''
    while(i < perl_code_bin_data_len):
        t = struct.unpack('<I',perl_code_orig[i:i+4])[0] ^ struct.unpack('<I',perl_code_bin[8:12])[0]
        decypt_xor_data = decypt_xor_data + struct.pack('I', t)
        i = i + 4
    # with open('bfs_file', 'ab') as fout:
    #   fout.write(decypt_xor_data)
    return decypt_xor_data

def extract_files_list(files_tables, files_tables_len):
    print hex(files_tables_len)
    i = 0
    files_list = []
    while(i < files_tables_len):
        filename_len = struct.unpack('<H', files_tables[i:i+2])[0]
        # print hex(filename_len)
        j = 0
        filename = ''
        while(j != filename_len):
            filename += struct.pack('B', (ord(files_tables[i+2+j:i+2+j+1]) ^ 0xEA))
            j = j + 1
        k = 0
        # print filename
        if (filename_len+2) % 4 != 0:
            k = 4 - ((filename_len+2) % 4)
        # while(files_tables[i+2+filename_len+k] == '#'):
        #   k = k + 1
        filedata_offset = struct.unpack('<I', files_tables[i+2+filename_len+k:i+2+filename_len+k+4])[0]
        # print filename_len
        # print filename
        print filename_len, filename, hex(filedata_offset)
        files_list.append([filename_len,filename,filedata_offset])
        i = i + 2 + filename_len + k + 4
    return files_list

def extract_files(input_file,files_list, perl_code_bin):
    i = 0
    while(i < len(files_list)):
        filename = files_list[i][1]
        filedata_offset = files_list[i][2]
        filedata_dec_len = struct.unpack('<I', perl_code_bin[filedata_offset:filedata_offset+4])[0]
        filedata_enc_len = struct.unpack('<I', perl_code_bin[filedata_offset+4:filedata_offset+8])[0]
        filedata_dec_method = struct.unpack('<I', perl_code_bin[filedata_offset+8:filedata_offset+12])[0]
        if (filedata_dec_len == filedata_enc_len) and (filedata_dec_method == 0x02):
            print '[+] XOR  -->  {}'.format(filename)
            j = 0
            filedata = ''
            while (j != filedata_dec_len):
                filedata += struct.pack('B', (ord(perl_code_bin[filedata_offset+12+j:filedata_offset+12+j+1]) ^ 0xEA))
                j = j + 1
            filename = filename.replace('*','_').replace('<','-')
            if not os.path.exists('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe")):
                os.makedirs('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe"))
            if '/' in filename and not os.path.exists('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0])):
                os.makedirs('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0]))
            fd = open('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\')), 'wb+')
            fd.write(filedata)
            fd.close()
            i = i + 1
        elif (filedata_dec_len != filedata_enc_len) and (filedata_dec_method == 0x03):
            print '[+] ZLIB  -->  {}'.format(filename)
            enc_filedata = perl_code_bin[filedata_offset+12:filedata_offset+12+filedata_enc_len]
            dec_filedata = zlib.decompress(enc_filedata)
            j = 0
            filedata = ''
            while (j != filedata_dec_len):
                filedata += struct.pack('B', (ord(dec_filedata[j:j+1]) ^ 0xEA))
                j = j + 1
            filename = filename.replace('*','_').replace('<','-')
            if not os.path.exists('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe")):
                os.makedirs('.\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe"))
            if '/' in filename and not os.path.exists('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0])):
                os.makedirs('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\').rsplit('\\',1)[0]))
            fd = open('.\\{}\\{}'.format(os.path.splitext(os.path.basename(input_file))[0] + "_exe", filename.replace('/','\\')), 'wb+')
            fd.write(filedata)
            fd.close()
            i = i + 1
        else:
            print '[-] Unknow  -->  {}'.format(filename)
            i = i + 1

    return True

def extract_perl_bin(filename):
    fd = open(filename, 'rb')
    orig_data = fd.read()
    fd.close()
    if orig_data[0x138F50:0x138F54] == '\xBC\x55\x21\xAB':
        # print struct.unpack('<I', orig_data[0x138F50:0x138F54])[0]
        print '[+] Found End Magic'
    else:
        print '[-] Not Found End Magic'
        return False
    perl_code_bin_total_len = struct.unpack('<I', orig_data[0x138F58:0x138F5C])[0]
    if orig_data[0x138F50-perl_code_bin_total_len:0x138F54-perl_code_bin_total_len] == '\x7F\x3A\x5B\xAA':
        print '[+] Found Begin Magic'
    else:
        print '[-] Not Found Begin Magic'
        return False
    xor_perl_code_bin = orig_data[0x138F50-perl_code_bin_total_len:0x138F50]
    # print "bin len is %s" % hex(perl_code_bin_total_len)
    if xor_perl_code_bin == "":
        print "error"
    print '[+] Decypt 1st for perl_code_bin'
    # print xor_perl_code_bin
    perl_code_bin = decypt_perl_code_bin_xor(xor_perl_code_bin)
    if perl_code_bin == "":
        print "error"
        return 0
    file_offset = struct.unpack('<H', perl_code_bin[20:22])[0]
    files_tables_end_offset = struct.unpack('<I', perl_code_bin[24:28])[0]
    files_tables = perl_code_bin[file_offset:files_tables_end_offset]
    files_list = extract_files_list(files_tables, files_tables_end_offset-32)
    ret = extract_files(filename, files_list, perl_code_bin)

    return ret

if __name__ == '__main__':
    if len(sys.argv) == 2:
        print '----------------------------------------------------------------'
        print extract_perl_bin(sys.argv[1])
        print '----------------------------------------------------------------'
    else:
    print 'need one argv which is filename'
```
最后会解出目录，里面包含 `Capture.pm`、`Exporter.pm`、`AUTOLOAD.pm`、`Carp.pm`、`Config.pm`、`DynaLoader.pm`、`Fcntl.pm`、`perl.pl` 等文件。

而里面的perl.pl就是最后的perl文件

`perl.pl` 中直接保存了 flag 校验逻辑：

```perl
$flag = "WMCTF{I_WAnt_dynam1c_Flag}";
print "please input the flag:";
$line = <STDIN>;
chomp($line);
if ($line eq $flag) {
    print "congratulation!"
} else {
    print "no,wrong"
}
```

flag就隐藏在此处。

## 方法总结

- 核心技巧：识别 Perl 打包 EXE 的初始化流程，通过 `FindResourceA` / `LoadResource` / `LockResource` 和文件尾 magic 定位 BFS 资源，按文件表结构批量还原内部文件。
- 识别信号：Windows 程序中出现 Perl 运行库初始化、BFS 节段、资源查找 API、文件头长度/大小端标志/文件列表等结构时，应优先按“打包脚本资源”而不是普通校验函数分析。
- 复用要点：文件数据有两类处理方式：`decypt_method == 0x02` 时按字节异或 `0xEA`，`decypt_method == 0x03` 时先 zlib 解压再异或；长期 WP 保留文件头、文件表和解包脚本即可复现。
