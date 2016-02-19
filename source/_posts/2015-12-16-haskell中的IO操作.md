title: "haskell中的IO操作"
date: 2015-12-16 10:08:34
categories: 
- Tech
tags: 
- haskell
---

IO操作，即输入输出操作，一般指同系统的command line和文件系统的操作。本文首先给出IO操作的常见需求，然后给出各个需求在Haskell中的解决方案。这些解决方案可能是某个函数，也可能是某段具有既定模式的代码。<!-- more -->

先来看看对于command line，有什么需求：
* 输出到command line
* 从command line 输入
* 调用程序时，同时接收参数
* 错误处理

对于文件操作，有什么需求：
* 读入文件
* 写文件
* 错误处理

咱们一个需求一个需求来：

## 输出到command line
提到输出到command line，自然就想到了大名鼎鼎的hello world程序了。先来一段hello world：创建一个文件叫helloworld.hs，然后输入以下内容：
```haskell
main = putStrLn "Hello World!"
```
`main`函数就是程序的入口函数。`putStrLn`是：
```haskelld
:t putStrLn
putStrLn :: String -> IO ()
```
`putStrLn`接收一个String，返回一个IO ()。那么这个IO ()是什么呢？IO是一个type constructor，它接收一个参数，然后返回一个类型。比如IO String。而“()”是一个空的元组(tuple)，是个类型。那么这个类型能干什么？什么都不能干。

haskell中函数有参数，同时也必须要有返回值。接收一个输入，返回一个特定的输出。整个函数的职责就是制造这个输出。而putStrLn函数的目的是向command line输出。所以，除了返回一个IO ()之外，putStrLn还做了别的事情。这种函数，我们称之为不纯的函数，反之，只返回输出的函数，称为纯函数。在函数式编程里面，函数往往都是纯函数。不纯的函数，做了除制造输出之外的事情，这称为副作用(side effects)。putStrLn就是一个具有副作用的非纯函数。

使用`ghc --make helloworld.hs`进行编译链接，使用`./helloworld`进行调用。

回到问题本身，输出到command line，可以有这些函数：
* putStrLn，将一个String输出到command line，并且换行
* putStr，将一个String输出入到command line，不换行
* putChar，将一个Char输出到command line
* print，接收一个实现了Show的类型，然后将show的结果输出到command line

一些例子：
```haskell
main = do
    putStrLn "hello "
    putStr "this is a good sign  ====  "
    putChar 'a'
    print [1,2,3]
```
## 从command line输入
先看一段代码：
```haskell
main = do
    putStrLn "What's your name?"
    line <- getLine
    putStrLn $ "your name: " ++ line
```
输入函数：
* `getLine`的类型`getLine :: IO String`，它不接收参数，然后返回一个IO String型。通过“<-”操作，我们可以拿到getLine获得(注意，不是返回)的String。
* `getContents`的类型`getContents :: IO String`，也是通过“<-”操作，获得输入的String，不过getLine是获取一行，而这个是获取所有输入。如果输入是来自控制台，那么键入一个字符就会处理一个字符，如果是来自文件就会拿到所有的输入。如何取得来自文件的输入呢？使用重定向：“./helloworld < haiku.txt”

## 调用程序时，同时接收参数
先上一段代码:
```haskell
import System.Environment

main = do
    args <- getArgs
    name <- getProgName
    print args
    print name
```
两个函数：
* getArgs，类型为：getArgs :: IO [String], 返回一个String数组，数组里面的值是程序接收的参数
* getProgName，类型为： getProgName :: IO String，返回一个String，为程序的名字。

## 读入文件
继续代码优先:
```haskell
import System.IO

main = do
    handle <- openFile "girlfriend.txt" ReadMode
    contents <- hGetContents handle
    putStr contents
    hClose handle
```
解释解释：
* 首先需要引入System.IO模块
* 使用openFile打开一个文件，openFile的类型`FilePath -> IOMode -> IO Handle`，它接收一个表示路径的String，以及文件的打开方式，有ReadMode, WriteMode, AppendMode, 及ReadWriteMode。返回一个IO的句柄(handle)。
* 使用hGetContents拿到文件输入，hGetContents的类型`hGetContents :: Handle -> IO String`
* 处理完之后使用hClose关闭文件句柄。

另一种方式是使用`withFile`：
```haskell
import System.IO
main = do
    withFile "girlfriend.txt" ReadMode (\handle -> do
        contents <- hGetContents handle
        putStr contents)
```
withFile的类型为`withFile :: FilePath -> IOMode -> (Handle -> IO r) -> IO r`，接收三个参数：一个文件路径的String，一个打开模式，以及一个处理函数。这里不需要使用hClose来关闭handle，withFile会自己完成关闭的操作。

我们也可用使用`bracket`：
```haskell
import Control.Exception
main = do
    bracket (openFile "girlfriend.txt" ReadMode)
        (\handle -> hClose handle)
        (\handle -> do
                    contents <- hGetContents handle
                    putStr contents)
```
`bracket`函数的类型`bracket :: IO a -> (a -> IO b) -> (a -> IO c) -> IO c`，第一参数是获取资源的一个IO；第二个参数是一个函数，用来释放资源；第三个参数是拿到资源，并进行一些操作。最后返回一个IO。

除了上面的方法，还可用使用`readFile`来读取文件内容：
```haskell
main = do
    contents <- readFile "girlfriend.txt"
    putStr contents
```
readFile的类型`readFile :: FilePath -> IO String`。接收一个表示路径的String，返回一个IO String。

所以总结一下，读取文件这里介绍了两种方式：
* 使用`openFile`函数，得到handle，然后对handle进行操作，比如hGetContents等
* 使用`readFile`函数，直接得到文件内容


## 写文件
看代码啦：
```haskell
import Data.Char

main = do
    contents <- readFile "girlfriend.txt"
    writeFile "girlfriendcaps.txt" (map toUpper contents)
```
使用`readFile`读入文件，使用`writeFile`将String写入到文件中。

writeFile的类型`writeFile :: FilePath -> String -> IO ()`，接收两个参数，一个文件路径，一个String。将String写入到文件中，如果文件没有创建的话，就新建一个文件，然后写入。
## 其他
command line跟文件其实类似，属于同一个抽象，一般称为stdin和stdout。

参考资料：
[1][《Learn You a Haskell for Great Good》](http://book.douban.com/subject/4934481/)
