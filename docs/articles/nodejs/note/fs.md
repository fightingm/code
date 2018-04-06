# fs模块

## 1. fs.stat

fs.stat 用来判断是文件还是文件夹，第一个参数是一个路径，第二个参数是一个回调函数，如果路径存在，回调函数的第二个参数包含两个方法isFile()用来判断该路径是不是文件，isDirectory()用来判断该路径是不是一个文件夹。

    fs.stat('hello.js', (err, stats) => {
        if(err) {
            console.log(err);
            return false;
        };
        if(stats.isFile()) {
            console.log('是一个文件')
        }
        if(stats.isDirectory()) {
            console.log('是一个文件夹')
        }
    });

## 2. fs.mkdir

fs.mkdir 用来创建一个文件夹，第一个参数是目录名称，第二个参数是一个回调函数，如果当前目录下已存在该文件夹，回调函数的第一个参数就是一个错误。

    fs.mkdir('css', err => {
        if(err) {
            console.log(err);
            return false;
        };
        console.log('创建目录成功');
    });

## 3. fs.writeFile()

fs.writeFile() 用来创建写入文件，第一个参数是文件名,第二个参数是写入的内容，可以是String或者Buffer类型,第三个参数是一些写入参数，包括encoding(编码方式，默认utf8)、mode(读写权限，默认438)、flag(不知道是啥，默认'w'),当这个参数为一个String时，表示字符编码方式，第四个参数是回调函数。

    fs.writeFile('1.md', 'test md', err => {
        if(err) {
            console.log(err);
            return false;
        };
        console.log('写入成功');
    });
这里如果不存在这个文件，就会创建并写入，存在就会覆盖文件的内容。

## 4. fs.appendFile()

fs.appendFile 用来追加文件内容，如果文件不存在，创建并写入， 如果存在则追加。可以用来记录日志。

    fs.appendFile('2.md', 'test\n', err => {
        if(err) {
            console.log(err);
            return false;
        }
        console.log('追加文件成功');
    });

## 5. fs.readFile()

fs.readFile 读取文件内容，默认输出的是buffer类型，可以用toString方法转换成string类型。

    fs.readFile('1.md', (err, data) => {
        if(err) {
            console.log(err);
            return false;
        }
        console.log(data.toString());
    });

## 6. fs.readdir()

fs.readdir 用来读取文件夹,只会读取到当前文件夹下的第一层的文件和目录。

    fs.readdir('hello', (err,files) => {
        if(err) {
            console.log(err);
            return false;
        }
        console.log(files);
    });

## 7. fs.rename()

fs.rename用来修改文件或者目录的名称，第一个参数必须是一个存在的目录或者文件，第二个参数如果文件或者目录不存在。会创建并命名。下面这段代码相当于剪切操作。

    fs.rename('3.md', 'hello/3.md', err => {
        if(err) {
            console.log(err);
            return false;
        }
        console.log('名称修改成功');
    });

## 8. fs.rmdir()

fs.rmdir用来删除文件夹,被删除的文件夹里面不能有文件，否则会删除失败。

    fs.rmdir('hello/test', err => {
        if(err) {
            console.log(err);
            return false;
        }
        console.log('删除目录成功');
    });

## 9. fs.unlink()

fs.unlink用来删除文件

    fs.unlink('hello.js', err => {
        if(err) {
            console.log(err);
            return false;
        }
        console.log('删除文件成功');
    });

## 10. fs.createReadStream()

fs.createReadStream用来创建读取流,适用于文件内容较大的文件。

    const readStream = fs.createReadStream('1.md');
    let data = '';
    let count = 0;

    // 正在读取
    readStream.on('data', chunk => {
        data += chunk;
        count ++;
    });

    // 读取完成
    readStream.on('end', () => {
        console.log(data);
        console.log(count);
    });
    // 读取失败
    readStream.on('error', err => {
        console.log(err);
    });

当文件内容较少的时候会看到控制台打印的count始终为1，说明是一次读取完成的。当文件内容较多时count会大于1，说明是多次读取的。

## 11. fs.createWriteStrm()

fs.createWriteStrm用来创建写入流,当写入内容较小时使用writeFile或者appendFile。

    // 创建一个写入流，写入到2,md
    const writeStream = fs.createWriteStream('2.md');

    // 写入
    writeStream.write('test md', 'utf8');
    // 停止写入 ，在这之后再调用write方法会报错
    writeStream.end();

    // 写入完成
    writeStream.on('finish', () => {
        console.log('写入完成');
    });
    // 写入失败
    writeStream.on('error', err => {
        console.log(err);
    });

## 12. pipe

pipe顾名思义是一个管道，用于一边读取一边写入。

    fs.createReadStream('2.md').pipe(fs.createWriteStream('4.md'));

这就把2.md文件的内容以管道的形式写到了4.md文件中。

相当下面这种方式：

    const readStream = fs.createReadStream('2.md');
    const writeStream = fs.createWriteStream('4.md');
    readStream.on('data', chunk => {
        writeStream.write(chunk, 'utf8');
    });
    readStream.on('end', () => {
        writeStream.end();
    });


完。