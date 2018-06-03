# 关于v-for中key的思考

在vue中我们用 v-for 指令根据一组数组的选项列表进行渲染，而在渲染的时候我们一般需要为每一项指定一个唯一的key值，在vue的官方文档中是这样描述的：

当 Vue.js 用 v-for 正在更新已渲染过的元素列表时，它默认用“就地复用”策略。如果数据项的顺序被改变，Vue 将不会移动 DOM 元素来匹配数据项的顺序， 而是简单复用此处每个元素，并且确保它在特定索引下显示已被渲染过的每个元素。

这个默认的模式是高效的，但是只适用于不依赖子组件状态或临时 DOM 状态 (例如：表单输入值) 的列表渲染输出。

今天我们要讨论的就是这种不适用的场景。

## 使用index作为key出现的问题

在很多情况下我们的数据并没有唯一的id值，这时我们一般会选择使用index作为key值。大多数情况下这不会有什么问题，如果不需要对这个数据进行排序或者增删操作的时候。
但是有的时候会出现问题，比如下面这个例子：

<iframe width="100%" height="300" src="//jsrun.net/RuZKp/embedded/all/light/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

首先在姓名，年龄，地址后面的输入框中输入一些信息，然后点击下面的按钮删除第一项，这时候你会发现，虽然第一项变成了年龄，但是年龄后面的输入内容却变成了原来姓名的输入内容，地址后面的输入内容变成了原来年龄的输入内容。

## 使用id作为key

这个时候我们给每一个数据添加一个id，然后使用id作为key，代码如下：
<iframe width="100%" height="300" src="//jsrun.net/juZKp/embedded/all/light/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

## 使用index作为key的正确渲染

如果我们的数据没有id，我们怎么样才能让上面的例子能够按照我们希望的逻辑渲染呢？

我们来分析一下第一种方式的代码逻辑，首先我们渲染完成之后，姓名对应的key是0，年龄对应的key是1，地址对应的key是2。
这时候我们删除姓名所对应的数据，然后年龄所对应的index就变成了0，地址所对应的key变成了1。然后走到了虚拟dom，通过虚拟dom的比较，vue认为key为0和key为1的数据还在，key为2的dom被删除。所以最终的结果是，我们删除了数组的第0项，真正被删除的dom是最后一个dom,前面两个dom只是改变了item.text的值，其他并没有发生改变。

从上面的例子，我们可以看出，虽然input的内容不是我么想要的渲染结果，但是text的渲染是正确的。
既然我们可以用text字段来保存文字，我们同样也可以增加一个字段来保存input的内容:

<iframe width="100%" height="300" src="//jsrun.net/uuZKp/embedded/all/light/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

完。