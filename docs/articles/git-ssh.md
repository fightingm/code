# git 配置ssh

## 1. 生成密钥对

先查看本地有没有密钥对

    $ cd ~/.ssh
    $ ls

如果有id_rsa和id_rsa.pub(或者是id_dsa和id_dsa.pub之类成对的文件)，可以直接进入第二步。

假如没有这些文件，可以用 ssh-keygen 来创建。

    $ ssh-keygen -t rsa -C "your_email@youremail.com"

    Creates a new ssh key using the provided email # Generating public/private rsa key pair.

    Enter file in which to save the key (/home/you/.ssh/id_rsa):

直接按Enter就行。当然，也可以自定义名称和设置密码。

完了之后，大概是这样

    Your public key has been saved in /home/you/.ssh/id_rsa.pub.
    The key fingerprint is: # 01:0f:f4:3b:ca:85:d6:17:a1:7d:f0:68:9d:f0:a2:db your_email@youremail.com

这就成功生成了密钥对就生成了。

## 2. 添加公钥到你的远程仓库

这里以github为例。其他比如gitlab是类似的方法。

    $ cat ~/.ssh/id_rsa.pub
    ssh-rsa *******

登陆你的`github`帐户。点击你的头像，然后 `Settings` -> 左栏点击 `SSH and GPG keys` -> 点击 `New SSH key`

然后你复制上面的公钥内容，粘贴进“Key”输入框中。 然后在"Title"输入框自己起一个名字

点击 `Add SSH key`。

完成以后，验证下这个key是不是正常工作：

    $ ssh -T git@github.com
    Attempts to ssh to github

如果，看到：

    Hi xxx! You've successfully authenticated, but GitHub does not # provide shell access.

恭喜你，你的设置已经成功了。

完。