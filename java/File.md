##### ASCII 二进制数字与符号的转换

一个字节一共有8位，可以组合成256种状态

ASCII 码一共规定了128个字符的编码，比如空格`SPACE`是32（二进制`00100000`），大写的字母`A`是65（二进制`01000001`）。这128个符号（包括32个不能打印出来的控制符号），只占用了一个字节的后面7位，最前面的一位统一规定为`0`

**NOTICE:** 一般而言，英语只需要128个符号就够了，但是其他语言却远远不够，所以欧洲对于ASCII码进行了扩展，变成了256位，也就是用上了最后那个0，**`不同欧洲国家的语言编码0-127位都是一样的，不同的是它们扩展的部分`**



##### 象形文字的难点

在英文中可以使用一个字节就能标识一个英文符号，但汉语不行，象形文字的编码难度更大，**`所以常用的编码方式GB2312中中文字符使用两个字节来标识，理论上可以翻译256*256个字符`**，也就是65536个汉字



##### Unicode对于编码的扩展与困境

Unicode是一个很大的集合，它可以容纳100多万个符号，最多使用4字节去标识字符

**困境：**

1. 如何才能区别 Unicode 和 ASCII,比如判断三个字节标识一个字符而不是三个字符呢？
2. 如果Unicode规定了每个字符用三个或者四个字节标识，那势必会造成大量的字节浪费



##### UTF-8

UTF-8 就是在互联网上使用最广的一种 Unicode 的实现方式！

UTF-8 就是在互联网上使用最广的一种 Unicode 的实现方式！

UTF-8 就是在互联网上使用最广的一种 Unicode 的实现方式！

##### 字符流

> 字符流就是在字节流的基础上，加上编码，形成的数据流，因为字节流在操作字符时可能有中文乱码，所以引申了字符流

- Reader
  
  1. BufferedReader
  
     > BufferedReader的包装只是让读取数据时有缓冲区，使读取效率更高，但是并不能解决乱码问题，如果需要指定编码，需要配合InputStreamReader使用
  
     | 方法                                      | 描述                                         |
     | ----------------------------------------- | -------------------------------------------- |
     | `int read()`                              | 读取单个字符。                               |
     | `int read(char[] cbuf, int off, int len)` | 取出len个字符放到数组cbuf的off位置           |
     | `String readLine()`                       | **读取一个文本行。**                         |
     | `long skip(long n)`                       | 跳过字符。                                   |
     | `boolean ready()`                         | 判断此流是否已准备好被读取。                 |
     | `void close()`                            | 关闭该流并释放与之关联的所有资源。           |
     | `void mark(int readAheadLimit)`           | 标记流中的当前位置。                         |
     | `boolean markSupported()`                 | 判断此流是否支持 mark() 操作（它一定支持）。 |
     | `void reset()`                            | 将流重置到最新的标记。                       |
  
     ```java
     # 支持逐个读取
     public class IBufferedReader {
         @Test
         public void printByFileReader() throws IOException {
         	// new FileReader使用项目编码方式去读取数据
             BufferedReader bufferedReader = new BufferedReader(new FileReader("G:\\Study\\src\\test\\java\\com\\kiy\\test\\1"));
             if(!bufferedReader.ready()){
                 System.out.println("文件流暂时无法读取!");
                 return;
             }
             int result =0;
             // 读取单个字符且返回int,如t返回值就是116，强转成char变成字母
             while ((result=bufferedReader.read())!=-1){
                 System.out.println(result);
                 System.out.println((char) result
                 );
             }
             bufferedReader.close();
         }
     }
     ```
  
     ```java
     # 支持一次读取指定个数的字符
     @Test
     public void printByFileReaderChars() throws IOException {
         BufferedReader bufferedReader = new BufferedReader(new FileReader("G:\\Study\\src\\test\\java\\com\\kiy\\test\\1"));
         if(!bufferedReader.ready()){
             System.out.println("文件流暂时无法读取!");
             return;
         }
         int size=0;
         char[] chars = new char[20];
         // 一次读取n个字符放到数组中，返回值为本次读取的个数
         while ((size=bufferedReader.read(chars,0,chars.length))!=-1){
             System.out.println(size);
             System.out.println(new String(chars,0,size));
         }
         bufferedReader.close();
     }
     ```
  
     ```java
      # 支持逐行读取并返回字符串
      @Test
      public void printByFileReaderLine() throws IOException {
          BufferedReader bufferedReader = new BufferedReader(new FileReader("G:\\Study\\src\\test\\java\\com\\kiy\\test\\1"));
          if(!bufferedReader.ready()){
              System.out.println("文件流暂时无法读取!");
              return;
          }
          int size = 0;
          String line;
          // 读取一行数据，返回的字符串将不包含换行符
          while((line=bufferedReader.readLine())!=null){
              System.out.println(line+"\n");
          }
          bufferedReader.close();
      }
     ```
  
     ```java
     # 配合InputStreamReader使用指定的编码方式来读取文件
      @Test
     public void printByInputStreamEncoding() throws IOException {
         BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(new FileInputStream("G:\\Study\\src\\test\\java\\com\\kiy\\test\\utf8"), "utf-8"));
         char[] chars = new char[20];
         int size;
         while ((size=bufferedReader.read(chars,0,chars.length))!=-1){
         	System.out.println(new String(chars,0,size));
         }
     }
     // notice:BufferedReader只负责读到它的内部缓冲区中，而解码的工作是InputStreamReader完成的
     ```
  
  2. InputStreamReader
  
     > InputStreamReader流，实现从字节流到字符流的转换
  
     1、FileReader类仅仅是InputStreamReader的简单衍生并未扩展任何功能
  
     2、FileReader类读取数据实质是InputStreamReader类在读取，而InputStreamReader读取数据实际是StreamDecoder类读取
  
     3、因此在使用字符输入流的时候实际是StreamDecoder类在发挥作用
  
     ```java
     InputStreamReader in = new InputStreamReader(new FileInputStream("D:\\bf\\Desktop\\test.txt"), "UTF-8");
     int n ;
     while((n = in.read()) != -1){
     	System.out.print((char)n); 
     }
     in.close();
     ```
  
  3. StringReader
  
     ```java
     @Test
     public void test1() throws IOException {
         String s = "gggg";
         StringReader stringReader = new StringReader(s);
         int c;
         while ((c=stringReader.read())!=-1){
         System.out.println(c);
         }
     }
     ```
  
  4. PipedReader
  
  5. CharArrayReader
  
  6. FilterReader
  
     > `FileReader`使用项目的编码来读取解析字符，不能指定编码，可能会出现编码问题
  
- Writer
  1. BufferedWriter
  
     | 方法                                 | 描述                                            |
     | ------------------------------------ | ----------------------------------------------- |
     | `BufferedWriter(Writer out)`         | 创建一个缓冲字符输出流,使用默认大小的输出缓冲区 |
     | `BufferedWriter(Writer out, int sz)` | 创建一个缓冲字符输出流,使用给定大小的输出缓冲区 |
  
     | 方法                                        | 描述                     |
     | ------------------------------------------- | ------------------------ |
     | `void write(int c)`                         | 写入单个字符。           |
     | `void write(char[] cbuf, int off, int len)` | 写入字符数组的某一部分。 |
     | `void write(String s, int off, int len)`    | 写入字符串的某一部分。   |
     | `void newLine()`                            | 写入一个行分隔符。       |
     | `void close()`                              | 关闭此流，但要先刷新它。 |
     | `void flush()`                              | 刷新该流的缓冲。         |
  
     ```java
     public static void main(String[] args) throws IOException
     {
         BufferedWriter writer=new BufferedWriter(new FileWriter("静夜思.txt"));
         char ch='床';
         //写入一个字符
         writer.write(ch);
         String next="前明月光,";
         char[] nexts=next.toCharArray();
         //写入一个字符数组
         writer.write(nexts,0,nexts.length);
         //写入换行符
         writer.newLine();//写入换行符
         String nextLine="疑是地上霜。";
         //写入一个字符串。
         writer.write(nextLine);
         //关闭流
         writer.close();
     }
     // 注意，这种写入不是以追加的形式
     ```
  
     ```java
     # 复制文本数据，使用不同编码
     # BufferedReader和BufferedWriter一样，只是让流有了一个缓冲区，但并不能决定字节转换成字符的编码格式
     static void copyByLineEncoding(String srcFile, String srcEncoding, String destFile,
             String destEncoding)
     {
         BufferedReader reader = null;
         BufferedWriter writer = null;
         try
         {
             reader = new BufferedReader(new InputStreamReader(new FileInputStream(srcFile), srcEncoding));
             writer = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(destFile), destEncoding));
             char[] charArray = new char[512];
             int size;
             while ((size = reader.read(charArray, 0, charArray.length)) != -1)
             {
                 writer.write(charArray, 0, size);
             }
         } catch (UnsupportedEncodingException | FileNotFoundException e)
         {
             e.printStackTrace();
         } catch (IOException e)
         {
             e.printStackTrace();
         } finally
         {
             if (writer != null)
             {
                 try
                 {
                     writer.close();
                 } catch (IOException e)
                 {
                     e.printStackTrace();
                 }
             }
             if (reader != null)
             {
                 try
                 {
                     reader.close();
                 } catch (IOException e)
                 {
                     e.printStackTrace();
                 }
             }
         }
     }
     ```
  
  2. OutputStreamWriter
  
  3. PrinterWriter
  
  4. StringWriter
  
  5. PipedWriter
  
  6. CharArrayWriter
  
  7. FilterWriter





































