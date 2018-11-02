---
title: 导出csv乱码问题分析
date: 2017-02-22 21:33:45
tags: [2017,Apache,csv]
category: [java]
---
# 现象
导出元数据表信息到csv文件中，出现部分电脑导出中文乱码。

# 导出实现
csv文件操作使用commons-csv组件，封装一系列文件操作及处理过程。更详细介绍可参见[官网](http://commons.apache.org/proper/commons-csv/)介绍。


<!--more-->

# 问题分析
## 分析一
初步探索是文件编码不对，导致乱码。那么就设置个万能编码UTF-8的编码头部。

![写入文件](https://github.com/alanzhang211/blog-image/raw/master//2017/02/csv/%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5.JPG)

然而，问题并没有解决，还是出现乱码。

然后只能google/百度。

发现了，可以规避问题。

## 分析二
在简体中文环境下，EXCEL打开的CSV文件默认是ANSI编码，如果CSV文件的编码方式为utf-8、Unicode等编码可能就会出现文件乱码的情况。

### 解决方式
+ 使用记事本另存为ANSI编码。具体过程参见[记事本另存为解决乱乱码问题](http://jingyan.baidu.com/article/ac6a9a5e4c681b2b653eacf1.html)。
+ 使用excle以“数据”->“自文本” 的方式打开。[解决Excel打开UTF-8编码CSV文件乱码的问题](http://www.gaohaipeng.com/2251.html)

**以上不做重点分析，供参考**

## 分析三
### 业务分析
想到元数据缓存的业务流程，第一次下载时，先将数据下载到服务器端。也就是write服务器，以csv文件的形式保存。下一次下载直接读文件就可以，不用再一次sql查询。

然后，分析一种是服务端的文件流出来。客户端的download的过程也是文件流写入的过程。所以，想到写出的过程是不是也是编码格式未限定。

### 部分源码

```
        File file = new File(path + fileName);
        if (!file.exists()) {
            write2CSV(mtQueryResultVo);
        }
        long startTime = System.currentTimeMillis();
        try {
            bis = new BufferedInputStream(new FileInputStream(file));
            out = new BufferedOutputStream(httpServletResponse.getOutputStream());
            byte[] buff = new byte[2048];
            while (true) {
                int bytesRead;
                if (-1 == (bytesRead = bis.read(buff, 0, buff.length))) {
                    break;
                }
                out.write(buff, 0, bytesRead);
            }
        } catch (Exception e) {
            logger.error("error=",e);
            throw e;
        } finally {
            try {
                if (bis != null) {
                    bis.close();
                }
                if (out != null) {
                    out.flush();
                    out.close();
                }
            } catch (IOException e) {
                logger.error("error=",e);
                throw e;
            }
        }
```
看到问题没？

```
bis = new BufferedInputStream(new FileInputStream(file));
out = new BufferedOutputStream(httpServletResponse.getOutputStream());
```
的过程没有指定编码。那么，问题来了。这里会导致依据客户端环境编码进行处理。进而，部分客户端下载出现乱码问题。

### 方案分析
对于java的I/O流处理，是以装饰模式设计的，也可以理解问管道模式。就是一层套一层。关于装饰者模式，自行温故。

这里简单说下java的I/O的处理。
以Reader为例（Writer与Reader对应的层级结构对应）

![I/O Reader](https://github.com/alanzhang211/blog-image/raw/master//2017/02/csv/IO%E7%B1%BB%E5%9B%BE.JPG)


Reader 类是 Java 的 I/O 中读字符的父类，而 InputStream 类是读字节的父类，InputStreamReader 类就是关联字节到字符的桥梁，它负责在 I/O 过程中处理读取字节到字符的转换，而具体字节到字符的解码实现它由 StreamDecoder 去实现，在 StreamDecoder 解码过程中必须由用户指定 Charset 编码格式。值得注意的是如果你没有指定 Charset，将使用本地环境中的默认字符集，

### 修改源码
```
 BufferedReader br = null;
        BufferedWriter wr = null;
        try {
            br =new BufferedReader(new InputStreamReader(new FileInputStream(file),"UTF-8"));
            wr = new BufferedWriter(new OutputStreamWriter(out, "GB2312"), 1024);
            String line = null;
            while ((line = br.readLine()) != null) {
                wr.write(line);
                wr.newLine();
            }
        } catch (Exception e) {
            logger.error("error=",e);
            throw e;
        } finally {
            try {
                if (br != null) {
                    br.close();
                }
                if (out != null) {
                    wr.flush();
                    wr.close();
                }
            } catch (IOException e) {
                logger.error("error=",e);
                throw e;
            }
        }
```
# 结束语
编码一值是个头痛的问题，主要是两个流向的流在转换对接中，编码保持一致。明确指定编码格式，避免数据依赖于客户端本机环境。
