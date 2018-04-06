# fs两个练手demo

## 第一个demo

判断服务器上有没有upload目录，没有则创建。

这里一般用于图片上传。

    fs.stat('upload', (err, stats) => {
        if(err) {
            console.log(err);
            createUpload();
            return false;
        }
    });

    function createUpload() {
        fs.mkdir('upload', err => {
            if(err) {
                consoel.log(err);
                return false;
            }
            console.log('创建成功');
        })
    }

## 2. 第二个demo

找出src目录下的所有目录，并且打印出来

版本一： promise写法

    fs.readdir('src', (err, files) => {
        if(err) {
            console.log(err);
            return false;
        }
        let dirNames = [];
        let fileNames = [];
        files.forEach((file, i) => {
            isDirectory('src/', file).then(({type, name}) => {
                if(type === 'dir') {
                    dirNames.push(name)
                }else {
                    fileNames.push(name);
                }
                if(i === files.length-1) {
                    console.log('dirs', dirNames);
                    console.log('files', fileNames);
                }
            }).catch(err => {
                console.log(err);
            });
        });
    });

    function isDirectory(pre, name) {
        return new Promise((resolve, reject) => {
            fs.stat(pre + name, (err, stats) => {
                if(err) {
                    reject(err);
                    return false;
                }
                if(stats.isDirectory()) {
                    resolve({
                        type: 'dir',
                        name
                    });
                }else {
                    resolve({
                        type: 'file',
                        name
                    });
                }
            });
        })
    }

版本二： async await写法

    fs.readdir('src', (err, files) => {
        if(err) {
            console.log(err);
            return false;
        }
        let dirNames = [];
        let fileNames = [];
        files.forEach(async (file, i) => {
            try {
                const { type, name } = await isDirectory('src/', file);
                if(type === 'dir') {
                    dirNames.push(name)
                }else {
                    fileNames.push(name);
                }
                if(i === files.length-1) {
                    console.log('dirs', dirNames);
                    console.log('files', fileNames);
                }
            } catch (error) {
                consoel.log(error);
            }
        });
    });

完。