# vue-data-binding  数据双向绑定初探

今天看了一篇很不错的文章：[`vue 早期源码学习系列之四：如何实现动态数据绑定`](https://juejin.im/entry/57e8d6ac8ac247005bd986a2)，跟着敲了一遍，发现其中有意思的地方还是很多的，一些知识我之前都没有接触过，这里要好好整理一下思路。
### 大致思路
首先Vue会使用documentfragment劫持根元素里包含的所有节点，这些节点不仅包括标签元素，还包括文本，甚至换行的回车。 
然后Vue会把data中所有的数据，用defindProperty()变成Vue的访问器属性，这样每次修改这些数据的时候，就会触发相应属性的get，set方法。  

接下来编译处理劫持到的dom节点，遍历所有节点，根据nodeType来判断节点类型，根据节点本身的属性（是否有v-model等属性）或者文本节点的内容（是否符合{{文本插值}}的格式）来判断节点是否需要编译。对v-model，绑定事件当输入的时候，改变Vue中的数据。对文本节点，将他作为一个观察者watcher放入观察者列表，当Vue数据改变的时候，会有一个主题对象，对列表中的观察者们发布改变的消息，观察者们再更新自己，改变节点中的显示，从而达到双向绑定的目的。

demo：  
```javascript  
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>双向绑定demo</title>
<style>
    body{margin: 200px auto;width: 800px;}
</style>
</head>
<body>
    <div id="app">
        <input type="text" id="input" v-model="text">
        {{ text }}
    </div>



<script>
    // Vue构造函数
    function Vue(options){
        this.data = options.data;
        //发布订阅模式，数据绑定
        var data = this.data;
        observe(data,this);

        var id = options.el;
        var dom = nodeToFragment(document.getElementById(id),this);
        //编译完成后返回到app中
        document.getElementById("app").appendChild(dom);
    }

    //遍历传入实例的data对象的属性，将其设置为Vue对象的访问器属性
    function observe(obj,vm){
        Object.keys(obj).forEach(function(key){
            defineReactive(vm,key,obj[key]);
        });
    }
    //访问器属性是对象中的一种特殊属性，它不能直接在对象中设置，而必须通过defineProperty()方法单独定义。
    //访问器属性的"值"比较特殊，读取或设置访问器属性的值，实际上是调用其内部特性：get和set函数。
    function defineReactive(obj,key,val){
        //这里用到了观察者(订阅/发布)模式,它定义了一种一对多的关系，让多个观察者监听一个主题对象，这个主题对象的状态发生改变时会通知所有观察者对象，观察者对象就可以更新自己的状态。

        //实例化一个主题对象，对象中有空的观察者列表
        var dep = new Dep();
        //将data的每一个属性都设置为Vue对象的访问器属性，属性名和data中相同
        //所以每次修改Vue.data的时候，都会调用下边的get和set方法。然后会监听v-model的input事件，当改变了input的值，就相应的改变Vue.data的数据，然后触发这里的set方法
        Object.defineProperty(obj,key,{
            get: function(){
                //Dep.target指针指向watcher，增加订阅者watcher到主体对象Dep
                if(Dep.target){
                    dep.addSub(Dep.target);
                }
                return val;
            },
            set: function(newVal){
                if(newVal === val){
                    return
                }
                val = newVal;
                //console.log(val);
                //给订阅者列表中的watchers发出通知
                dep.notify();
            }
        });
    }

    //主题对象Dep构造函数
    function Dep(){
        this.subs = [];
    }
    //Dep有两个方法，增加观察者  和  发布消息
    Dep.prototype = {
        addSub: function(sub){
            this.subs.push(sub);
        },
        notify: function(){
            this.subs.forEach(function(sub){
                sub.update();
            });
        }
    }

    //DocumentFragment（文档片段）可以看作节点容器，它可以包含多个子节点，当我们将它插入到DOM中时，只有它的子节点会插入目标节点，所以把它看作一组节点的容器。使用DocumentFragment处理节点，速度和性能远远优于直接操作DOM。Vue进行编译时，就是将挂载目标的所有子节点劫持（真的是劫持）到DocumentFragment中，经过一番处理后，再将DocumentFragment整体返回插入挂载目标。
    //var dom = nodeToFragment(document.getElementById("app"));
    //console.log(dom);
    //返回到app中
    //document.getElementById("app").appendChild(dom);


    function nodeToFragment(node,vm){
        var flag = document.createDocumentFragment();
        var child;
        //劫持node的所有子节点（真的在dom树中消失了，所以要在下边重新返回搭到app中）
        while (child = node.firstChild){
            //先编译所有的子节点，再劫持到文档片段中
            compile(child,vm);
            flag.appendChild(child);
        }
        return flag;
    }

    //编译节点，初始化数据绑定
    function compile(node,vm){
        //该正则匹配的是 ：{{任意内容}}
        var reg = /\{\{(.*)\}\}/;
        //节点类型为元素
        if(node.nodeType === 1){
            var attr = node.attributes;
            //解析属性，不同的属性不用的处理方式，这里只写了v-model属性
            for(var i=0;i<attr.length;i++){
                if (attr[i].nodeName == "v-model") {
                    //获取节点中v-model属性的值，也就是绑定的属性名
                    var name = attr[i].nodeValue;

                    node.addEventListener("input",function(e){
                        //当触发input事件时改变vue.data中相应的属性的值，进而触发该属性的set方法
                        vm[name] = e.target.value;
                    });
                    //改变之后，通过属性名取得数据
                    node.value = vm.data[name];
                    //用完删，所以浏览器中编译之后的节点上没有v-model属性
                    node.removeAttribute("v-model");
                }
            }
        }
        //节点类型为text
        if(node.nodeType === 3){
            //text是否满足文本插值的写法：{{任意内容}}
            if(reg.test(node.nodeValue)){
                //获取匹配到的字符串：这里的RegExp.$1是RegExp的一个属性
                //该属性表示正则表达式reg中，第一个()里边的内容，也就是
                //{{任意内容}} 中的  文本【任意内容】
                var name = RegExp.$1;
                //去掉前后空格，并将处理后的数据写入节点
                name = name.trim();
                //node.nodeValue = vm.data[name];
                //实例化一个新的订阅者watcher
                new Watcher(vm,node,name);
                return;
            }
        }

    }

    //观察者构造函数。
    //上边实例化新的观察者的时候执行这个函数：通过get()取得Vue.data中对应的数据，然后通过update()方法把数据更新到节点中。
    function Watcher(vm,node,name){
        //让全局变量Dep的target属性的指针指向该watcher实例
        Dep.target = this;
        this.vm = vm;
        this.node = node;
        this.name = name;
        //放入Dep.target才能update()?????????????????????????????????????????
        this.update();
        Dep.target = null;
    }
    // 观察者使用update方法，实际上是
    Watcher.prototype = {
        update: function(){
            this.get();
            this.node.nodeValue = this.value;
        },
        //获取data中的属性值 
        get: function(){
            this.value = this.vm[this.name]; //触发相应属性的get
        }
    }



    var vm = new Vue({
        el: "app",
        data: {
            text: "Hello Vue"
        }
    })


function GetRequest() {  
     var url = this.location.search; //获取url中"?"符后的字串  
     var theRequest = {}; 
     if (url.indexOf("?") != -1) {  
        var str = url.substr(1);  
        strs = str.split("&");  
        for(var i = 0; i < strs.length; i ++) {  
           theRequest[strs[i].split("=")[0]]=unescape(strs[i].split("=")[1]);  
        }  
     }  
     return theRequest;  
  }
  console.log(this.GetRequest())
</script>
</body>
</html>



```