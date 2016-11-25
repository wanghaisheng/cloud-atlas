> 一直没有系统学习过`sed`，这里是一些例子的汇总整理，快刀斩乱麻......

样例文本`file`内容如下

```
Cygwin
Unix
Linux
Solaris
AIX
```

* 删除文件中指定行

例如删除第1行

```bash
sed '1d' file
```

删除第3行

```bash
sed '3d' file
```

> 上述`sed`命令是直接将`sed`修改后的内容出处到标准输出（屏幕）上，而源文件不变。要更新源文件，需要加上`-i`参数。例如：

```bash
sed -i '3d' file
```

> `sed`是将源文件复制后修改，所以`-i`参数实际是修改后覆盖源文件

删除最后一行

```bash
sed '$d' file
```

删除第2行到第四行

```bash
sed '2,4d' file
```

保留第2行到第4行，删除其余行（反转选择就是加上`!`符号）

```bash
sed '2,4!d' file
```

这里输出就是

```bash
Unix
Linux
Solaris
```

删除第1行和最后一行（注意：每一行之间用`;`分隔）

```bash
sed '1d;$d' file
```

> 同上，如果要删除第2行和第4行，就使用`'2d;3d'`

* 匹配并删除

删除带有一个特定字符开头的行，例如，`L`开头的行

```bash
sed '/^L/d' file
```

这里输出就是 

```
Cygwin
Unix
Solaris
AIX
```

删除`x`结尾的行

```bash
sed '/x$/d' file
```

还支持正则，例如，要删除`x`或`X`结尾的行

```bash
sed '/[xX]$/d' file
```

删除文件中的空白行

```bash
sed '/^$/d' file
```

> `^`表示开头，`$`表示结尾，由于中间没有任何字符，就是空白行

删除空白行或者包含一些空格的行

```bash
sed '/^ *$/d' file
```

> 这里`*`表示0个或者多个重复的前一个字符，这里前一个字符是空格，所以这里` *`就表示有一个空格或者多个空格。此时会删除完全空白行，以及包含一个或多个空格的空白行。

删除全部是大写字符的行（这里会删除`AIX`行）

```bash
sed '/^[A-Z]*$/d' file
```

> `[A-Z]`表示全部大写字母，后面跟了`*`表示前面的`[A-Z]`字符重复（任意大写字符），也就是表示整行都是大写字符，所以就删除了`AIX`

删除**没有**包含字符串`Unix`的行

```bash
sed '/Unix/!d' file
```

删除包含`Unix`或`Linux`的行（或的符号是`|`，注意需要加转义符）

```bash
sed '/Unix\|Linux/d' file
```

删除从第一行到匹配上`Linux`的行

```bash
sed '1,/Linux/d' file
```

输出的结果是

```bash
Solaris
AIX
```

删除从匹配`Linux`行到文档最后行

```bash
sed '/Linux/,$d' file
```

如果最后一行包含了`AIX`，则删除最后一行（如果最后一行是其他内容则不删除）

```bash
sed '${/AIX/d;}' file
```

输出内容是

```
Cygwin
Unix
Linux
Solaris
```

> 这里`$`表示最后一行，只在这行包含了`AIX`才删除，这里引入了`if`判断（也就是`{}`表示`if`条件）

如果最后一行包含了`AIX`或者`HPUX`则删除最后一行

```bash
sed '${/AIX\|HPUX/d;} file
```

只在第1行到第4行之间出现`Solaris`字符串匹配情况下删除这行：

```bash
sed '1,4{/Solaris/d;}' file
```

如果一行中包含了`Unix`关键字，则删除这行和下面一行

```bash
sed '/Unix/{N;d;}' file
```

> 这里`N`命令读取匹配行的下一行，而`d`命令则删除匹配行和下一行

上述命令输出是

```
Cygwin
Solaris
AIX
```

只删除包含了`Unix`的下一行（但`Unix`行不删除）

```bash
sed '/Unix/{N;s/\n.*//;}' file
```

输出内容是

```
Cygwin
Unix
Solaris
AIX
```

[sed - 25 examples to delete a line or pattern in a file](http://unix-school.blogspot.com/2012/06/sed-25-examples-to-delete-line-or.html)原文还有3个例子较为复杂：

删除匹配行和匹配行之前的行

```bash
sed -n '/Linux/{s/.*//;x;d;};x;p;${x;p;}' file | sed '/^$/d'
```

删除包含`Linux`的行之前的行，但不删除'Linux'行：

```bash
sed -n '/Linux/{x;d;};1h;1!{x;p;};${x;p;}' file
```

删除包含`Linux`行以及之前的行和之后好的行

```bash
sed -n '/Linux/{N;s/.*//;x;d;};x;p;${x;p;}' file | sed '/^$/d'
```

* 合并重复的空格成一个空格

在使用`cut`工具来截取`ps`出来的进程的`pid`，会遇到一个问题，就是每列之间的空格数量是不一定的，这样虽然可以通过`awk`来截取，但是蹪于`cut`命令就不行了。解决的方法是将多个空格合并成一个空格，`sed`命令提供了这个功能：

```bash
ps axu | grep '[j]boss' | sed 's/\s\+/ /g' | cut -d' ' -f2
```

> 这里使用的是GNU sed，这个sed提供了`\s`表示空格（扩展），`\+`表示多个空格

或者

```bash
ps axu | grep '[j]boss' | sed 's/\s\s*/ /g' | cut -d' ' -f2
```

> 不过，在OS X中，需要传递`-E`参数来激活sed扩展正则表达式，然后使用`[[:space:]]`来代替`\s`，也就是

```bash
ps axu | grep '[j]boss' | sed -E 's/[[:space:]]+/ /g' | cut -d' ' -f2
```

> 请参考[cut使用举例](../utilities/cut_examples.md)

* 替换换行符（`\n`）: 读取整个文件替换换行符成一个空格

```bash
sed ':a;N;$!ba;s/\n/ /g'
```

> 首先创建一个label `:a`
>
> 通过`N`将当前行和下一行通过空格连接
>
> 如果到了最后一行，则创建label `$!ba` （这里`$!`表示不在最后一行做这个操作）
>
> 最后使用在模式坑拜拜中的空格来替换每个换行符

**上述方法实在太拗口**，其实可以使用`tr`工具来实现这个任务

```bash
tr '\n' ' ' < input_filename
```

或者

```bash
tr --delete '\n' < input.txt > output.txt
```

* 在匹配行的最后添加内容

```
sed -i '/kernel \/boot/s/$/ clocksource_failover=acpi_pm/' /boot/grub/grub.cfg
```

参考 [How to append a string at end of a specific line in a file in bash](http://stackoverflow.com/questions/22159044/how-to-append-a-string-at-end-of-a-specific-line-in-a-file-in-bash)

# 参考

* [sed - 25 examples to delete a line or pattern in a file](http://unix-school.blogspot.com/2012/06/sed-25-examples-to-delete-line-or.html)
* [linux cut help - how to specify more spaces for the delimiter?](http://stackoverflow.com/questions/7142735/linux-cut-help-how-to-specify-more-spaces-for-the-delimiter)
* [How can I replace a newline (\n) using sed?](http://stackoverflow.com/questions/1251999/how-can-i-replace-a-newline-n-using-sed)