---
title: 一些有意思的细节
updated: 2017-07-15 12:05
---
编程中有很多有意思的细节，看到了，就记在这里。


### `|` 简化判断

一堆数按位或，只要有多于一个数为负，则结果为负。
```Java
public void write(byte b[], int off, int len) throws IOException {
   if ((off | len | (b.length - (len + off)) | (off + len)) < 0)
       throw new IndexOutOfBoundsException();

   for (int i = 0 ; i < len ; i++) {
       write(b[off + i]);
   }
}
```
*from*: `FilterOutputStream`




