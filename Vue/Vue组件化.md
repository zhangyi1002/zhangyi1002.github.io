## Vue组件化

### 组件通信

#### 父组件 -> 子组件

1. 属性 props

```vue
// parent
<HelloWorld msg="Welcome to My Blog" />

// child HelloWorld
props: {msg: String}
```

2. 特性 $attrs (2.4 新增的) 

```vue
// parent 
<HelloWorld extra="hello"></HelloWorld>

// child 未在props中声明extra
<p>{{$attrs.extra}}</p>
```

**说明**：
- 默认情况下父作用域的不被认作 props 的特性绑定 (attribute bindings) 将会“回退”且作为普通的 HTML 特性应用在子组件的根元素上
- 当撰写包裹一个目标元素或另一个组件的组件时，这可能不会总是符合预期行为
- 通过设置 inheritAttrs 到 false，这些默认行为将会被去掉
- 而通过实例属性 $attrs 可以让这些特性生效，且可以通过 v-bind 显性的绑定到非根元素上
- 这个选项不影响 class 和 style 绑定

3. 引用 refs 

```vue
// parent 使用refs引用拿到子组件实例
<HelloWorld ref="child"></HelloWorld>

mounted() {
    this.$refs.child.msg = 'change msg of child';
}

// child 
data() {
    return {msg: 'value0'};
}
```

4. 子元素 $children

```vue
// parent
this.$children[0].msg = 'change msg of child';
```

#### 子组件 -> 父组件

1. 自定义事件

```vue
//child
this.$emit('change' selfMsg);

// parent
<HelloWorld @change="changeHandler($event)"></HelloWorld>
```

#### 兄弟组件

1. 通过共同祖辈组件搭桥，$parent 或 $root

```vue
// brother1
this.$parent.$on('foo', handler);

// brother2
this.$parent.$emit('foo');
```

#### 祖先 -> 后代

1. 避免层层传递 props，可使用 provide/inject API实现

```vue
// ancestor
provide() {
    return {foo: 'foo'};
}

// descendant
inject: ['foo']
```
**说明**：
- provide 和 inject 绑定并不是可响应的
- 如果传入了一个可监听的对象，那么其对象的属性还是可响应的

#### 任意两个组件

1. 事件总线：创建Bus类负责事件派发、监听和回调管理

```vue
// bus.js
$emit(name, args)
$on(name, fn)

// main.js
Vue.prototype.$bus = new Bus();

// component1 
this.$bus.$emit('event-bus', 'event-bus msg');

// component2
this.$bus.$on('event-bus', msg => console.log(msg));
```
2. vuex：创建唯一的全局数据管理者 store，通过其管理数据并通知组件状态变更

### 插槽

内容分发，用于复合组件开发

#### 匿名插槽

```vue
// component
<div>
    <slot></slot>
</div>

// parent
<comp>hello</comp>
```

#### 具名插槽
```vue
// component
<div>
    <slot name="title"></slot>
    <slot name="content"></slot>
    <slot></slot>
</div>

// parent
<comp>
    <template v-slot:title>comp2 slot title</template>
    <template v-slot:content>comp2 slot content</template>
    <template v-slot:default>comp2 slot default</template>
</comp>
```

**说明**：v-slot 只能添加在 <template>上（与废弃的slot特性不同），[我是唯一的例外](https://cn.vuejs.org/v2/guide/components-slots.html#%E7%8B%AC%E5%8D%A0%E9%BB%98%E8%AE%A4%E6%8F%92%E6%A7%BD%E7%9A%84%E7%BC%A9%E5%86%99%E8%AF%AD%E6%B3%95)

#### 作用域插槽

插槽内容访问子组件中才有的数据来渲染模板

```vue
// component
<slot name="title" :title="titleInfo" :subtitle="subtitle"></slot>
<slot name="content" :content="contentInfo"></slot>

data() {
        return {
            titleInfo: {
                text: '我是title',
                color: 'red'
            },
            contentInfo: {
                text: '我是content',
                color: 'green'
            },
            subtitle: '我是subtitle'
        };
    }
    
// parent  v-slot: 简写为 #
<comp>
    <template #title="slotProps">
        <p :style="`color:${slotProps.title.color}`">{{slotProps.title.text}} {{slotProps.subtitle}}</p>
    </template>
    <template #content="{content}">
        <p :style="`color:${content.color}`">{{content.text}}</p>
    </template>
</comp>
```
### 弹窗类组件
弹窗类组件特点：
- 在当前vue实例之外独立存在，通常挂载于body；
- 通过JS动态创建，不需要在任何组件中声明；
- 常见使用方法：
```vue
this.$create(Notice, {
    title: '提示',
    message:'验证不通过',
    duration: 3000
}).show();
```
需要物料：create函数、Notice组件

#### create 函数
```js
import Vue from "vue";
// 接收组件定义
export default function create(Component, props) {
    // 1.创建实例
    const vm = new Vue({
        render(h) {
            // render函数将传入组件配置对象转换为虚拟dom
            // h === CreateElement
            return h(Component, {props});
        }
    }).$mount(); // 执行挂载函数，但未指定挂载目标，表示只执行初始化工作

    // 2.挂载，将生成dom元素追加至body
    document.body.appendChild(vm.$el);
    
    // 3.给组件实例添加销毁方法
    const comp = vm.$children[0];
    comp.remove = () => {
        document.body.removeChild(vm.$el);
        vm.$destroy();
    };
    return comp;
}
```

#### Notice 组件
```vue
<template>
    <div v-if="isShow" class="dialog-wrapper">
        <h1>{{title}}</h1>
        <p>{{message}}</p>
    </div>
</template>
<script>
export default {
    props: {
        title: {
            type: String,
            default: '提示'
        },
        message: {
            type: String,
            default: ''
        },
        duration: {
            type: Number,
            default: 3000
        }
    },
    data() {
        return {
            isShow: false
        }
    },
    methods: {
        show() {
            this.isShow = true;
            setTimeout(this.hide, this.duration);
        },
        hide() {
            this.isShow = false;
            this.remove();
        }
    }
}
</script>
<style scoped>
    .dialog-wrapper {
        position: fixed;
        top: 30px;
        left: calc((100% - 300px) / 2);
        width: 300px;
        border-radius: 8px;
        border: 1px solid #eee;
        text-align: center;
    }

    h1 {
        border-bottom: 1px solid #eee;
        font-size: 16px;
        color: #000;
    }

    p {
        font-size: 12px;
        color: #666;
    }
</style>
```
#### 使用

```vue
const notice = create(Notice, {
    title: '注意',
    message: '今天要上班',
    duration: 1000
});
notice.show();
```

### 递归组件
递归组件是可以在自己模板中调用自身的组件
应用：实现嵌套菜单导航栏等

```vue
// Node.vue
<template>
    <div>
        <h3>{{data.title}}</h3>
        <Node v-for="item in data.children" :key="item.id" :data="item"></Node>
    </div>
</template>
<script>
export default {
    // name对递归组件是必要的
    name: 'Node',
    props: ['data']
}
</script>

// 使用
<template>
    <Node :data="data"></Node>
</template>
<script>
import Node from './Node'
export default {
    components: {Node},
    data() {
        return {
            data: {
                id: '1',
                title: '我是标题 level1',
                children: [
                    {
                        id: '2-1',
                        title: '我是标题 level2-1',
                        children: [
                            {
                                id: '3-1',
                                title: '我是标题 level3-1'
                            }
                        ]
                    },
                    {
                        id: '2-2',
                        title: '我是标题 level2-2'
                    }
                ]
            }
        };
    }
}
</script>
```
