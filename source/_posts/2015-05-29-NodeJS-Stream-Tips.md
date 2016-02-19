title: "Pipe function in gulp.js"
date: 2015-05-29 19:01:54
categories: 
- Tech
tags: 
- NodeJS
---
还记得linux里面的pipe函数吗？现在，在[NodeJS](http://nodejs.org)里面，也有一个pipe函数。这两个函数有着相同的理念。起的都是串联的作用。而在前端项目开发中，我选择了[gulp](http://gulpjs.com)工具，在里面，大量用到了pipe函数。这里，就自己的一些疑问进行了一些探索。<!--more-->
最近在使用[gulp](http://gulpjs.com)的时候，看到这样的代码：
```
 var minifyHTML = require('gulp-minify-html');
 var opts = {
  conditionals: true,
  spare:true
 };
 gulp.src('./static/html/*.html')
    .pipe(minifyHTML(opts))
    .pipe(gulp.dest('./dist/'));

```
 关于这段代码，就有一些问题：

 * pipe函数是干什么的？
 * minifyHTML()的返回值是什么类型？为什么是一个返回值不是直接传入minifyHTML这个函数？
 * gulp.dest()和minifyHTML()的返回值是同一类型的吗？


 先看看第一个问题。

 ### 1，pipe()函数是干什么的？
从上面的代码可以看出，pipe是gulp.src()函数的返回值上的函数。那么gulp.src()函数的返回值是什么？
根据gulpjs的文档：

{% blockquote @gulpdoc  https://github.com/gulpjs/gulp/blob/master/docs/API.md#gulpsrcglobs-options %}
Returns a stream of Vinyl files that can be piped to plugins
{% endblockquote %}

以及源码：

```
...
var vfs = require('vinyl-fs');
...
Gulp.prototype.src = vfs.src;
Gulp.prototype.dest = vfs.dest;

```
也就是说`gulp.src`其实就是`vinyl-fs.src`。
文档中提到Vinyl，什么是[Vinyl](https://github.com/wearefractal/vinyl-fs)?

简单来说，它是一个库，一个提供文件相关操作的库。什么样的文件操作呢？

比如文件读入，创建文件等。


这里有一个Stream的概念，这个Stream来自于[NodeJS](http://nodejs.org)。就是文件流。你可以把它想象成一条溪流，只是里面流淌着的不是水，而是数据。举个例子，这个“流”可以从文件流出，然后流入到另一个文件里面去。

那么`gulp.src()`返回的是一个Stream。NodeJS里面关于Stream的定义：

{% blockquote @NodeJS %}
A stream is an abstract interface implemented by various objects in Node. For example a request to an HTTP server is a stream, as is stdout. Streams are readable, writable, or both. All streams are instances of EventEmitter
{% endblockquote %}

**Stream是一个接口。并且可以分为Readable，writable或者同时两者都是（Duplex）。**
具体的定义：

Readable:

{% blockquote @NodeJS %}
The Readable stream interface is the abstraction for a source of data that you are reading from. In other words, data comes out of a Readable stream.
{% endblockquote %}

Writable:

{% blockquote @NodeJS %}
The Writable stream interface is an abstraction for a destination that you are writing data to.
{% endblockquote %}

Duplex:

{% blockquote @NodeJS %}
Duplex streams are streams that implement both the Readable and Writable interfaces.
{% endblockquote %}


其中，Readable接口有一个方法pipe:
{% codeblock %}
readable.pipe(destination[, options])
{% endcodeblock %}

[]里面的是可选的参数。这里我们只关注destination：

{% blockquote @NodeJS %}
destination | Writable Stream The destination for writing data
{% endblockquote %}

现在，我们知道pipe函数的调用者是一个Readable，参数是一个Writable。就是说，通过调用Pipe函数，我们把数据流从一个Readable的对象传给了一个Writable的对象。文档上如是说：

{% blockquote %}
This method pulls all the data out of a readable stream, and writes it to the supplied destination, automatically managing the flow so that the destination is not overwhelmed by a fast readable stream.
{% endblockquote %}

文档上的例子：

{% codeblock %}
var readable = getReadableStreamSomehow();
var writable = fs.createWriteStream('file.txt');
// All the data from readable goes into 'file.txt'
readable.pipe(writable);
{% endcodeblock %}

总而言之，pipe函数的作用就是传递数据。将数据从A传递到B。A必须是Readable，B必须是Writable. 也就是说，A必须实现Readable接口，B必须实现Writable接口。

**pipe函数的返回值是其参数，如果其参数是Duplex的话，就可以形成pipe链。**

{% blockquote @NodeJS %}
This function returns the destination stream, so you can set up pipe chains
{% endblockquote %}

回到最初的例子，gulp.src()返回的是一个Readable对象，而gulp.dest()返回的是一个Writable对象。至于实际上是不是Writable对象呢？且看下面。

### 2, minifyHTML()函数的返回值是什么类型？

[gulp-minify-html](https://github.com/jonathanepollack/gulp-minify-html)是用来minify html的一个gulp tool。

源码很简单，可以用来作为一个简单例子，其余的Gulp tool由此可见一斑：

```
var through = require('through2');
var gutil = require('gulp-util');
var Minimize = require('minimize');

module.exports = function(opt){

  function minimize (file, encoding, callback) {
    if (file.isNull()) {
      return callback(null, file);
    }

    if (file.isStream()) {
      return callback(new gutil.PluginError('gulp-minify-html', 'doesn\'t support Streams'));
    }

    var minimize = new Minimize(opt || {} );

    minimize.parse(file.contents.toString(), function (err, data) {
      if (err) {
        return callback(new gutil.PluginError('gulp-minify-html', err));
      }

      file.contents = new Buffer(data);
      callback(null, file);
    });
  }

  return through.obj(minimize);
}
```
首先，在调用minifyHtml()函数的时候，实际调用的是module.exports。也就是上面function(opt)这个函数。


这个函数返回了`through.obj(minimize)`。`through`对象来自于[through2](https://github.com/rvagg/through2).

我们来看看through函数的API DOC:

```
through2([ options, ] [ transformFunction ] [, flushFunction ])
```

through2实际上是NodeJS的streams.Transform的Wrapper。换句话说，**through2函数返回一个streams.Transform对象。而streams.Transform是一个Duplex流**。

上一节提到Duplex既是Readable的，又是Writable的。因为它是Writable的，所以可以作为pipe()函数的参数。

{% blockquote @NodeJS %}
A "transform" stream is a duplex stream where the output is causally connected in some way to the input, such as a zlib stream or a crypto stream.
{% endblockquote %}

而`through.obj`则相当于`through2({objectMode: true}, fn)`。所以我们搞清楚through函数就行了。

{% blockquote %}
Note that through2.obj(fn) is a convenience wrapper around through2({ objectMode: true }, fn).
{% endblockquote %}

三个参数都是可选的。第一个是一个对象，其余的都是函数。如果传入的第一参数不是对象，而是一个函数，那么就表示第一个参数options被省略掉了。由于through2函数是streams.Transform的wrapper，其中第二个参数`transformFunction`跟NodeJS的[stream.transform](https://nodejs.org/api/stream.html#stream_transform_transform_chunk_encoding_callback)函数一样：
{% blockquote %}
transform._transform(chunk, encoding, callback)#

chunk Buffer | String | The chunk to be transformed. Will always be a buffer unless the decodeStrings option was set to false.
encoding | String | If the chunk is a string, then this is the encoding type. (Ignore if decodeStrings chunk is a buffer.)
callback | Function | Call this function (optionally with an error argument and data) when you are done processing the supplied chunk.

{% endblockquote %}

简单来说transform表示对数据的一种处理，比如输入的数据流是1，2，3；对每个数据进行加1处理，则变成了2,3,4.

上面的_transform函数的参数。第一个表示传过来的数据，第二个表示编码，第三个是一个回调函数，处理完数据之后需要调用这个函数。

这个_transform函数没有返回值，通过在函数里面调用`this.push(data)`，把转换之后的数据写入到流里面，以便进行后续的处理。比如：
 ```
 var through2 = require('through2');
 var streamify = require('stream-array');

 streamify(['1','2','3'])
   .pipe(through2(function(chunk, encode, cb){
     this.push(chunk.toString() + 1);
     cb();
   }))
   .pipe(process.stdout)
 ```
可以看到，通过`this.push()`将数据流传给下一个Writable对象。

**至此，我们搞清楚了，minifyHtml()函数的返回值实际上是一个Duplex对象，是streams.Transform的一个实例。**

### 3，gulp.dest()和minify()的返回值是同一类型的吗？

根据第一节中的gulp的源码，`gulp.dest`实际上就是vinyl-fs的dest函数。Vinyl-fs中dest的文档表示“Returns a Readable/Writable stream”。而minify返回的是Duplex，等同于“Readable/Writable”。因此二者的返回值是同一类型的。

---------
In summary，在了解stream, readable, writable, duplex等概念后，再去看pipe函数，就一目了然了。
```
//readable.pipe(duplex)
gulp.src('./one.js')
    .pipe(gulp.dest('build/one.js'))

//duplex.pile(duplex)
gulp.src('./static/html/*.html')
   .pipe(minifyHTML(opts))
   .pipe(gulp.dest('./dist/'));
```
