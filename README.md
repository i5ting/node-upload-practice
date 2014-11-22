node-upload-practice
====================

## 上传说明

总体上来说，对于post文件上传这样的过程，主要有以下几个部分：

- 获取http请求报文肿的头部信息，我们可以从中获得是否为POST方法，实体主体的总大小，边界字符串等，这些对于实体主体数据的解析都是非常重要的
- 获取POST数据（实体主体）
- 对POST数据进行解析
- 将数据写入文件

### 获取http请求报文头部信息

利用nodejs中的 http.ServerRequest中获取

	request.method

用来标识请求类型

	request.headers

其中我们关心两个字段：

	content-type

包含了表单类型和边界字符串（下面会介绍）信息。

	content-length

post数据的长度


### 关于content-type

- get请求的headers中没有content-type这个字段
- post 的 content-type 有两种
	- application/x-www-form-urlencoded
		这种就是一般的文本表单用post传地数据，只要将得到的data用querystring解析下就可以了
	- multipart/form-data
		文件表单的传输，也是本文介绍的重点

### 获取POST数据

前面已经说过，post数据的传输是可能分包的，因此必然是异步的。post数据的接受过程如下：

```
var postData = '';
request.addListener("data", function(postDataChunk) {  // 有新的数据包到达就执行
  postData += postDataChunk;
  console.log("Received POST data chunk '"+
  postDataChunk + "'.");
});

request.addListener("end", function() {  // 数据传输完毕
  console.log('post data finish receiving: ' + postData );
});
```

注意，对于非文件post数据，上面以字符串接收是没问题的，但其实 postDataChunk 是一个 buffer 类型数据，在遇到二进制时，这样的接受方式存在问题。

### POST数据的解析

在解析POST数据之前，先介绍一下post数据的格式：

	multipart/form-data

类型的post数据


例如我们有表单如下

```
	<FORM action="http://server.com/cgi/handle"
	   enctype="multipart/form-data"
	   method="post">
	<P>
	What is your name? <INPUT type="text" name="submit-name"><BR>
	What files are you sending? <INPUT type="file" name="files"><BR>
	<INPUT type="submit" value="Send"> <INPUT type="reset">
	</FORM>
```
 
若用户在text字段中输入‘Neekey’，并且在file字段中选择文件‘text.txt’,那么服务器端收到的post数据如下：

```
--AaB03x
Content-Disposition: form-data; name="submit-name"

Neekey
--AaB03x
Content-Disposition: form-data; name="files"; filename="file1.txt"
Content-Type: text/plain

... contents of file1.txt ...
--AaB03x--
```

若file字段为空：

```
--AaB03x
Content-Disposition: form-data; name="submit-name"

Neekey
--AaB03x
Content-Disposition: form-data; name="files"; filename=""
Content-Type: text/plain

--AaB03x--
```

若将file 的 input修改为可以多个文件一起上传：

```
<FORM action="http://server.com/cgi/handle"
     enctype="multipart/form-data"
     method="post">
 <P>
 What is your name? <INPUT type="text" name="submit-name"><BR>
 What files are you sending? <INPUT type="file" name="files" multiple="multiple"><BR>
 <INPUT type="submit" value="Send"> <INPUT type="reset">
</FORM>
```

那么在text中输入‘Neekey’,并在file字段中选中两个文件’a.jpg’和’b.jpg’后：

```
--AaB03x
Content-Disposition: form-data; name="submit-name"

Neekey
--AaB03x
Content-Disposition: form-data; name="files"; filename="a.jpg"
Content-Type: image/jpeg

/* data of a.jpg */
--AaB03x
Content-Disposition: form-data; name="files"; filename="b.jpg"
Content-Type: image/jpeg

/* data of b.jpg */
--AaB03x--// 可以发现 两个文件数据部分，他们的name值是一样的
```

### 数据规则

简单总结下post数据的规则

1、 不同字段数据之间以边界字符串分隔：
	--boundary\r\n // 注意，如上面的headers的例子，分割字符串应该是 ------WebKitFormBoundaryuP1WvwP2LyvHpNCi\r\n

2、 每一行数据用”CR LF”（\r\n)分隔

3、 数据以 边界分割符 后面加上 –结尾，如：
 
	------WebKitFormBoundaryuP1WvwP2LyvHpNCi--\r\n
	
4、 每个字段数据的header信息（content-disposition/content-type）和字段数据以一个空行分隔：

	\r\n\r\n

更加详细的信息可以参考W3C的文档[Forms](http://www.w3.org/TR/html4/interact/forms.html#h-17.13.4.2)，不过文档中对于` multiple=“multiple”` 的文件表单的post数据格式使用了二级边界字符串，但是在实际测试中，multiple类型的表单和多个单文件表单上传数据的格式一致，有更加清楚的可以交流下：

```
If the user selected a second (image) file "file2.gif", the user agent might construct the parts as follows:

Content-Type: multipart/form-data; boundary=AaB03x

--AaB03x
Content-Disposition: form-data; name="submit-name"

Larry
--AaB03x
Content-Disposition: form-data; name="files"
Content-Type: multipart/mixed; boundary=BbC04y

--BbC04y
Content-Disposition: file; filename="file1.txt"
Content-Type: text/plain

... contents of file1.txt ...
--BbC04y
Content-Disposition: file; filename="file2.gif"
Content-Type: image/gif
Content-Transfer-Encoding: binary

...contents of file2.gif...
--BbC04y--
--AaB03x--
```


### 数据解析基本思路
- 必须使用buffer来进行post数据的解析	
	利用文章一开始的方法（data += chunk， data为字符串 ），可以利用字符串的操作，轻易地解析出各自端的信息，但是这样有两个问题：
	- 文件的写入需要buffer类型的数据
	- 二进制buffer转化为string，并做字符串操作后，起索引和字符串是不一致的（若原始数据就是字符串，一致），因此是先将不总的buffer数据的toString()复制给一个字符串，再利用字符串解析出个数据的start，end位置这样的方案也是不可取的。
- 利用边界字符串来分割各字段数据
- 每个字段数据中，使用空行（\r\n\r\n)来分割字段信息和字段数据
- 所有的数据都是以\r\n分割
- 利用上面的方法，我们以某种方式确定了数据在buffer中的start和end，利用buffer.splice( start, end ) 便可以进行文件写入了

### 文件写入

比较简单，使用 File System 模块（更好的做法是用stream)

    var fs = new require( 'fs' ).writeStream,
    file = new fs( filename );
    fs.write( buffer, function(){
        fs.end();
    });
		
## expressjs 4.*上传例子

https://github.com/i5ting/upload-cli

## expressjs3和4之间的差别

一定要注意3和4之间的差别，其实最大差别你可以到

https://github.com/expressjs

里找中间件,是架构上的变化

## multer

最简单最好用的上传中间件，`multer`,官方推荐

https://github.com/expressjs/multer


最简答的用法

```
var app = express();
var multer  = require('multer')

app.use(multer({ 
	dest: './uploads',
  rename: function (fieldname, filename) {
    return filename //.replace(/\W+/g, '-').toLowerCase() + Date.now()
  }
}))
```

## 更多Options

https://github.com/expressjs/multer#options

## 其他

- connect-busboy[github主页](https://github.com/mscdex/connect-busboy)
- node-formidable是比较流行的处理表单的nodejs模块。[github主页](https://github.com/felixge/node-formidable)

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## 版本历史

- v0.1.0 初始化版本

## 欢迎fork和反馈

- write by `i5ting` shiren1118@126.com

如有建议或意见，请在issue提问或邮件

## License

this repo is released under the [MIT
License](http://www.opensource.org/licenses/MIT).
