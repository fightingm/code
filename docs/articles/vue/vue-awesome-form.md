# vue-awesome-form的实现及踩坑记录

最近实现了一个`vue-awesome-form`组件，主要功能是根据`json`来生成一个表单，支持同时渲染多个表单，表单嵌套，表单验证，对于一个简单的项目，生成表单只需要一个`json`就可以完成。而且有时候表单项不是前端写死的，而是由后端控制的，这个时候我们这个组件就派上用场了。

[项目地址](https://github.com/fightingm/vue-awesome-form)。

[项目demo](https://jsrun.net/bKgKp/embedded/all/light/)。

本文主要介绍组件的实现方式及踩过的一些坑。

## 组件实现

### 递归组件

我们的json对象是可能有多层嵌套的，所以这里要用递归的方式来实现。关于vue的递归组件参考了官网的做法[https://cn.vuejs.org/v2/examples/tree-view.html](),在项目中实现方式如下

```html
    <template>
        <div class="jf-tree">
            <the-title :title="title" :level="objKey.length"></the-title>
            <div class="jf-tree-item">
            <component
                v-for="item in orderProperty(properties)"
                :key="item.key"
                :is="item.val.type"
                :objKey="getObjKeys(objKey, item.key)"
                :objVal="getObjVal(item.key)"
                v-bind="item.val">
            </component>
            </div>
        </div>
    </template>
```

对应的json数据格式是这样的:

```json
    "register": {
        "type": "TheTree",
        "title": "注册",
        "properties": {
            "name": {
                "type": "TheInput",
                "title": "姓名",
                "rules": {
                    "required": true,
                    "message": "The name cannot be empty"
                }
            },
            "location": {
                "type": "TheTree",
                "title": "地址信息",
                "propertyOrder": 3,
                "properties": {
                    "province": {
                        "type": "TheInput",
                        "title": "省份",
                        "rules": {
                            "required": true,
                            "message": "The 省份 cannot be empty"
                        }
                    },
                    "city": {
                        "type": "TheInput",
                        "title": "市",
                        "rules": {
                            "required": true,
                            "message": "The 市 cannot be empty"
                        }
                    }
                }
            }
        }
    }
```

最终的渲染效果如下：
<img src="./_media/vue/1.png" />
<img src="./_media/vue/2.png" />

json对象的每一项都要一个`type`字段，表示当前对象的渲染类型，目前支持支持的组件有：

 `TheTree`表示该项是个树形组件，它应该有一个`properties`字段来包含它的子组件。它渲染出来是一个`TheTitle`组件和properties属性下的所有表单项。

- `TheTitle`会渲染成一个h2，随着层级的深度font-size递减

- `TheInput`会渲染成一个input

- `TheTextarea`会渲染成一个textarea

- `ThePassInput`会渲染成一个type='password'的input

- `TheCheckbox`会渲染成一个 type ='checkbox'的input

- `TheRadio`会渲染成一个type=‘radio’的input

- `TheSelect`会渲染成一个下拉列表组件

- `TheAddInput`会渲染成一个可以动态增加，删除一个`TheInput`组件的组件

- `TheTable`会渲染成一个可以动态增加上述除`TheTree`和`TheAddInput` 组件的组件

上面的demo中包含了所有可能的渲染结果

tip: 因为我们的组件是根据`type`字段动态渲染的，所以这里使用`Vue`内置的动态组件`component`，它可以根据传入的`is`属性来自动渲染对应的组件，我们就不需要写一大堆的`v-if`来判断应该渲染哪个组件了。

### 表单项排序

因为我们的表单项是一个`json`对象，所以我们使用`v-for`渲染的时候无法保证数据的渲染顺序，如果我想要某一个表单项先渲染，你把它写在前面可能并没有用。就像你无法在`for-in`遍历对象中保证遍历的顺序一样。这是一个例子：

<iframe width="100%" height="300" src="//jsrun.net/YpgKp/embedded/all/light/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

所以我们需要在每一项数据中加一个`propertyOrder`字段表示它在同一层级中的顺序。然后我们根据`propertyOrder`字段把对象转成数组然后从小到大排序，如果没有这个字段的话默认值为999，代码如下：

``` js
    // 根据propertyOrder 从小到大排序
    orderProperty(oldObj) {
      // 先遍历对象，生成数组
      // 对数组排序
      const keys = Object.keys(oldObj);
      // 如果对象只有一个字段，不需要排序
      if(keys.length <= 1) return oldObj;
      return keys.map(key => {
        return {
          key,
          val: oldObj[key]
        };
      }).sort((pre, cur) => {
        return (pre.val.propertyOrder || 999) - (cur.val.propertyOrder || 999);
      });
    }
```
tip: 这里在排序的时候有一个运算符优先级的问题`-`优先级高于`||`，所以如果不确定运算符优先级的话要用`()`把想要先运算的表达式包起来。

### 组件间通信

我们的组件结构是这样设计的:
<img src="./_media/vue/4.png" />

以`TheTable`组件为例，我们的数据是这样传递的`SchemaForm->TheTree->TheTable->TheInput等表单组件`,我们把表单的值从`SchemaForm`一层层传递到`TheInput`组件,绑定为`TheInput`组件的`v-model`,然后当我们在`TheInput`组件中执行输入的时候,我们希望在`SchemaForm`组件中拿到新的值，从而更新数据，然后新的数据会再次通过`props`传递到`TheInput`组件中。对于这种组件的通信，我想到三种方式:

- 通过父子组件通信的方式，将数据一层层传回到Schema组件中
- 使用Vuex统一管理组件间通信
- 使用一个EventBus实现事件的统一监听和派发

第一种方式实现太过繁琐,不推荐。

对于第二种方式,vuex的文档中有这样一句话:
> 如果您不打算开发大型单页应用，使用 Vuex 可能是繁琐冗余的。确实是如此——如果您的应用够简单，您最好不要使用 Vuex。一个简单的 global event bus 就足够您所需了。但是，如果您需要构建一个中大型单页应用，您很可能会考虑如何更好地在组件外部管理状态，Vuex 将会成为自然而然的选择。

显然我们的组件并不复杂,不必要使用vuex,所以根据上面这句话里面提到的`global event bus`,我们采用第三种方式实现。

首先我们需要一个global对象,代码如下

```js
import Vue from "vue";

export const EventBus = new Vue();
```

是的，它什么也没做，就只是返回了一个Vue的实例对象。

然后我们在`TheInput`组件中是这样使用的:

```html
<template>
    <input class="jf-input" type="text" v-model="msg" />
</template>
<script>
    import { EventBus } from '../utils'

    export default {
        .....
        computed: {
            msg: {
                get: function() {
                    return this.objVal;
                },
                set: function(value) {
                    EventBus.$emit('on-set-form-data', {
                        key: this.keyName,
                        value
                    });
                }
            }
        }
        .....
    }
</script>
```

这里的`objVal`就是通过`SchemaForm`传过来的表单项的值,这里的`keyName`是一个表示当前属性链的一个数组,比如这样一个json对象：

```json
    {
        SchemaForm: {
            TheTree: {
                TheTable: {
                    TheInput: 123
                }
            }
        }
    }
```

`TheInput`的`objVal`就是123,`keyName`就是`['SchemaForm', 'TheTree', 'TheTable', 'TheInput']`

回到组件通信的问题,我们在TheInput组件中触发了一个`on-set-form-data`的事件,然后在`SchemaForm`我们是这样接收的:

```js
import { EventBus } from '../utils'

export default {
    .....
    created: function() {
        EventBus.$on('on-set-form-data', payload => {
            this.setFormData(payload);
        });
    },
    methods: {
        setFormData(payload) {
            const { key, value } = payload;
            key.reduce((pre, cur, curIndex, arr) => {
                // 如果是最后一项，就是我们要改变的字段
                if(curIndex === arr.length - 1) {
                    // Vue 不能检测直接用索引设置数组某一项的值
                    if(typeof(cur) === 'number') {
                        return pre.splice(cur, 1, value);
                    } else {
                        return pre[cur] = value;
                    }
                }
                return pre[cur] = pre[cur] || {}
            }, this.formValue);
        }
    }
    .....
}
```
我们通过$on监听`on-set-form-data`事件,然后触发setFormData方法,进而修改`formValue`的值，然后新的`formValue`就会传递给子组件的`objVal`,从而实现状态更新。

### 表单提交

我们将表单提交控制权交给使用者，在`SchemaForm`组件中暴露`validate`方法用来验证整个表单,使用者可以这样调用:

```js
handleSubmit() {
    this.$refs.schemaForm.validate((err, values) => {
        if(err) {
            console.log('验证失败');
        } else {
            // values是表单的值，你可以用来提交表单或者其他任何事情
            console.log('验证成功', values);
        }
    })
}
```
表单验证我们使用的是[async-validator](https://github.com/yiminghe/async-validator),它的验证是异步的,我们只能在回调函数中获取到验证结果,我们在`SchemaForm`中需要验证所有的表单项,就要拿到每一项的验证结果,我们使用`Promise`来完成这个功能,首先是每个表单项的验证函数:

```js
        validate() {
            return new Promise((resolve, reject) => {
                if(!this.rules) resolve({title: this.title, status: true});
                let descriptor = {
                    name: this.rules
                };
                let validator = new schema(descriptor);
                validator.validate({name: this.msg}, (err, fields) => {
                    if(err) {
                        resolve({
                            title: this.title,
                            status: false
                        });
                    }else {
                        resolve({
                            title: this.title,
                            status: true
                        });
                    }
                })
            })
        }
```

然后是SchemaForm的validate函数:

```js
validate(cb) {
    let err = false;
    // 这里的fields是所有表单组件组成的数组
    let len = this.fields.length;
    this.fields.forEach((field, index) => {
        field.validate().then(res => {
            const { title, status } = res;
            if(!status) {
                err = true;
            }
            if((index + 1) === len) {
                cb(err, this.formValue);
            }
        }).catch(err => {
            console.log(err);
        })
    })
}
```

## 踩到的坑

### v-for中的key

对于需要使用`v-for`来渲染的元素，比如`checkbox`的`options`,`select`的`options`,我都是用`value`作为每一项的`key`，因为可以保证唯一(其实用`index`作为`key`也没有什么影响，因为这些数据不会发生改变)。但是对于`TheAddInput`组件和`TheTable`组件来说，它们所包含的表单项是可以动态增删的，所以不存在可以唯一标识的字段。所以这里我们使用`index`作为`key`,但是这样会产生一些问题，vue的文档中是这样说的：
> 当 Vue.js 用 v-for 正在更新已渲染过的元素列表时，它默认用“就地复用”策略。如果数据项的顺序被改变，Vue 将不会移动 DOM 元素来匹配数据项的顺序， 而是简单复用此处每个元素，并且确保它在特定索引下显示已被渲染过的每个元素。这个类似 Vue 1.x 的 track-by="$index" 。

>这个默认的模式是高效的，但是只适用于不依赖子组件状态或临时 DOM 状态 (例如：表单输入值) 的列表渲染输出。

关于依赖临时 DOM 状态的列表渲染会遇到的问题我写了一个demo:
<iframe width="100%" height="300" src="//jsrun.net/RuZKp/embedded/all/light/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
在姓名,年龄,地址后面的输入框中输入一些信息,然后点击下面的按钮删除第一项,这时候你会发现,虽然第一项变成了年龄,但是年龄后面的输入内容却变成了原来姓名的输入内容,地址后面的输入内容变成了原来年龄的输入内容。这就是因为使用了`index`做为`key`,第一次的时候三个列表项的key分别是0,1,2;当我们删除第一项之后,新的列表的的`key`变成了0,1。就会造成真正删除的其实是`key`为2的元素,这时候每一项的label根据数据渲染出来还是正确的,但是后面`input`的内容是复用之前的`input`所以并没有相应发生变化。

而我们这里使用`index`作为`key`就属于依赖子组件的状态。以`TheAddInput`组件为例,这个组件内部调用了`TheInput`组件,而`TheInput`组件内部有一个自己的`data`: `validateState`用来控制验证信息的渲染。如果我们用`index`作为`key,`会存在这样一种情况：我们先增加一个`input`,然后它的校验规则是不能为空，当我们鼠标离开的时候触发校验，这时候`validateState`变成了`error`,校验信息就会显示在这个`input`下面,然后我们再增加一个input,在里面输入一些内容，这时候我们鼠标离开，第二个`input`的输入内容是符合校验规则的，所以它的`validateState`是`success,`不会显示校验信息，这时候我们删除第一个`input`,我们会发现第一个`input`的输入内容变成了第二个,但是校验信息却还在这个`input`下面。
<img src="./_media/vue/validate.gif" />

对于这种情况，我的处理方式是这样的：将`TheInput`的校验信息交由`TheAddInput`组件管理,在`TheAddInput`组件中新增一个`data`: `validateArray`;用来保存子组件的`validateState`,当我们新增一个表单项的时候我们就向`validateArray`中`push`一个`validateState`,然后使用`v-for`渲染`TheInput`组件的时候根据数据的`index`取到`validateArray`中对应的验证信息,每次`TheInput`组件触发验证的时候将事件传递给`TheAddInput`组件来更新`validateArray`的对应指定项，当我们删除的时候把`validateArray`中对应index的验证信息删除。这样的话当我们删除第0项的时候，虽然实际删除的是key为1的dom,但是对应的`validateArray`第0项也被删除，新的`validateArray`的第0项保存的是原来第1项的验证信息，这样数据就能对应上了。

### vue更新检测

接着上面`TheInput`的验证问题，一开始我是这样做的，在`TheInput`触发验证之后

```js
    this.dispatch('on-input-validate', {
        index: index,
        validateState: state
    })
```

然后在`TheAddInput`组件中监听

```js
    this.$on('on-input-validate', obj => {
      this.validateArray[obj.index] = obj.validateState;
    })
```

写完之后发现并没有效果,鼠标离开之后触发了验证,但是验证信息并没有显示出来。通过`vue-devtools`发现`TheAddInput`的`validateArray`已经更改了,但是`TheInput`组件的`props`并没有更新。突然想起来好像在vue的文档里面看到过这个，去找了找，果然发现了原因:

> 由于 JavaScript 的限制，Vue 不能检测以下变动的数组：

> 当你利用索引直接设置一个项时，例如：vm.items[indexOfItem] = newValue

> 当你修改数组的长度时，例如：vm.items.length = newLength

根据文档的解决方案,改成了下面这种写法:

```js
this.$on('on-input-validate', obj => {
    this.validateArray.splice(obj.index, 1, obj.validateState);
})
```

类似的,对于对象的更新检测也是有问题的,详细可以参考[vue文档](https://cn.vuejs.org/v2/guide/list.html#key),这里不做赘述。

### 不可变数据的重要性

对于`TheTable`组件,当我们点击新增一行的时候我们会根据表单`schema`的`addDefault`字段来生成一行默认的数据,这是[demo](http://jsrun.net/bKgKp)中表格的`addDefault`字段:

```json
    "addDefault": {
        "type": "",
        "name": "",
        "gender": "",
        "interests": []
    }
```
当我们点击添加一行的时候会触发`TheTable`组件的`add`方法：

```js
add() {
    this.msg.push(this.addDefault);
}
```

看上去没什么问题，但是在测试的时候发现了这样一个问题:
<img src="./_media/vue/immutable.gif" />

造成这种情况的原因就是因为后面每一个新增的数据使用的数据都共享了同一个`addDefault`,所以保持数据的不可变是很重要的，稍不注意就可能发生这种错误,对于大型项目的话可以使用immutable.js,我这个组件本身数据并不复杂，所以对这个`addDefault`实现了一层浅拷贝来解决这个问题:

```js
add() {
    this.msg.push({...this.addDefault});
}
```

### nextTick

对于`TheInput`组件,我们在`onInput`的时候将新的输入值传递给`SchemaForm`组件,然后在`blur`的时候来触发验证,这时候组件内的`objVal`是新的值,但是对于`TheRadio`组件和`TheCheckbox`组件，我们是在`onChange`事件中将新的值传给`SchemaForm`组件,并且同时进行验证，这时候我们拿到的`objVal`其实并不是新的值,而是当前的值,所以这里的验证要等待数据更新之后再触发,我写了一个`asyncValidate`来解决这个问题:

```js
asyncValidate() {
    this.$nextTick(() => {
        this.validate();
    });
}
```

## 最后

以上是个人开发`vue-awesome-form`的实现方式与总结，如有错误，欢迎指正，对组件有什么建议或者bug欢迎交流，谢谢。

完。