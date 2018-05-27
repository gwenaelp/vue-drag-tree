<template>
    <div>
        <div
          :id='model.id' @click="toggle" @dblclick="changeType" draggable='true' @dragstart='dragStart' @dragover='dragOver' @dragenter='dragEnter' @dragleave='dragLeave' @drop='drop' @dragend='dragEnd' :class="selectedItem === model ? 'treeNodeText active': 'treeNodeText'" @mouseover='mouseOver' @mouseout='mouseOut' :style='styleObj' v-if="model.type">
            <span
              :class="[open ? 'nodeClicked' : '','vue-drag-node-icon']"
              v-if="isFolder"
              @click="open = !open"
            />
            {{model[titleKey] || '[Untitled]' }}
            ({{model.type}})
            <span @click="removeChild(model)" v-if='model.type'>&nbsp;x</span>
        </div>
        <div v-else>ROOT</div>
        <div class='treeMargin' v-show="open" v-if="isFolder">
            <vue-drag-tree v-for="model in model[childrenKey]" :model="model" :key='model.id' :current-highlight='currentHighlight' :default-text='defaultText' :hover-color='hoverColor' :highlight-color='highlightColor' :childrenKey="childrenKey" @nodeClicked="$emit('nodeClicked', $event)" :selectedItem="selectedItem">
            </vue-drag-tree>
        </div>
    </div>
</template>

<script>
let id = 1000
let fromData = ''
let toData = ''
let fromParentModelChildren = ''  // from 节点的父组件的model
let nodeClicked = undefined   // Attention: 递归的所有组件共享同一个＂顶级作用域＂（这个词或许不太正确，但就这个意思）．即：共享上面这几个let变量．这为实现当前节点的高亮提供了基础．

export default {
    name: 'vue-drag-tree',
    data: function () {
        return {
            open: true,
            isClicked: false,
            styleObj: {
                background: 'white',
            }
        }
    },
    props: {
        model: Object,
        'default-text': String, // 填加节点时显示的默认文本．
        'current-highlight': Boolean, // 当前节点高亮
        'hover-color': String,
        'highlight-color': String,
        'childrenKey': {
          type: String,
          default: 'children',
        },
        'titleKey': {
          type: String,
          default: 'title',
        },
        selectedItem: {
          type: Object
        }
    },
    computed: {
        isFolder() {
            return this.model[this['childrenKey']] &&
                this.model[this['childrenKey']] .length
        },
    },
    methods: {
        toggle() {
            // 调用vue-drag-tree的父组件中的方法,以传递出当前被点击的节点的id值
            let rootTree = this.findRoot()

            this.$emit('nodeClicked', this.model);

            // 纪录节点被点击的状态
            this.isClicked = !this.isClicked

            // 用户需要节点高亮？　-->　this.currentHighlight ? 高亮 : 不高亮
            if (this.currentHighlight) {
                // 第一次点击当前节点．当前节点高亮，遍历重置其他节点的样式
                if (nodeClicked != this.model) {
                    let treeParent = rootTree.$parent

                    // 遍历重置所有树组件的高亮样式
                    let nodeStack = [treeParent.$children[0]]
                    while (nodeStack.length != 0) {
                        let item = nodeStack.shift()
                        item.styleObj.background = 'white'
                        if (item.$children && item.$children.length > 0) {
                            nodeStack = nodeStack.concat(item.$children)
                        }
                    }
                    // 然后把当前节点的样式设置为高亮
                    this.styleObj.background = this.highlightColor ? this.highlightColor : '#99A9BF'

                    // 设置为当前节点
                    nodeClicked = this.model
                }
            }
        },
        exchangeData(rootCom, from, to) {
            //如果drag的目的地是 + - 符号的话，退出。
            if (!to || !from || typeof to == 'string' || from == to) {
                return
            }

            from = JSON.parse(JSON.stringify(from))
            to = JSON.parse(JSON.stringify(to));
            // copy一个,最后再赋值给state.treeData.这样能保证值的变化都会触发视图刷新(因为JS判断引用类型是否相同是根据内存地址.)
            let treeData = JSON.parse(JSON.stringify(this.model));
            let nodeStack = [treeData];
            let status = 0;
            const childrenKey = this.childrenKey;

            console.log('exchangeData', from, to, rootCom);

            // 如果from或者to节点存在父子/祖辈关系，会报id of undefined的错。这种情况不考虑拖拽功能，所以catch住，返回/return就行
            try {
                // 广度优先遍历,找到涉及数据交换的两个对象.然后交换数据.
                while (!(status === 2)) {
                    let item = nodeStack.shift()
                    if (item == from) {
                        item = to
                        item.title = to.title
                        if (to[childrenKey] && to[childrenKey].length > 0) {
                            item[childrenKey] = to[childrenKey]
                        } else {
                            item[childrenKey] = []
                        }
                        status++
                        //找到后,跳出当前循环.
                        continue;
                    }
                    if (item == to) {
                        item = from
                        item.title = from.title
                        if (from[childrenKey] && from[childrenKey].length > 0) {
                            item[childrenKey] = from[childrenKey]
                        } else {
                            item[childrenKey] = []
                        }
                        status++
                        //找到后,跳出当前循环.
                        continue;
                    }
                    if (item[childrenKey] && item[childrenKey].length > 0) {
                        nodeStack = nodeStack.concat(item[childrenKey])
                    }
                }
            } catch (e) {
                return
            }
            //API: 对外开放交换后的数据的赋值操作
            rootCom.assignData(treeData)
        },
        changeType() {
            // 用户需要高亮-->才纪录当前被点击节点
            if (this.currentHighlight) {
                nodeClicked = this.model
            }
            if (!this.isFolder) {
                this.$set(this.model, this.childrenKey, [])
                this.addChild()
                this.open = true
                this.isClicked = true
            }
        },
        mouseOver(e) {
            if ((this.styleObj.background != '#99A9BF' || this.styleObj.background != this.hightlightColor) && e.target.className === 'treeNodeText') {
                e.target.style.background = this.hoverColor ? this.hoverColor : '#E5E9F2'
            }
        },
        mouseOut(e) {
          e.target.style.background = 'white'
        },
        findRoot() {
            // 返回Tree的根,即递归Tree时的最顶层那个vue-drag-tree组件
            let ok = false
            let that = this
            while (!ok) {
                // 如果父组件有data属性，说明当前组件是Tree组件递归调用发生时的第一个组件。
                // Warning: 因为是判断data属性是否存在，所有在别人使用该组件时，属性名必须得是data
                // v1.0.9-update: add another judgement method.
                if (!/VueDragTreeCom42/.test(that.$parent.$vnode.tag) || that.$parent.data) {
                    ok = true
                    // 交换两者的数据
                    break
                }
                that = that.$parent
            }
            return that
        },
        addChild() {
//            this.model[this.childrenKey].push({
//                title: this.defaultText ? this.defaultText : 'New Node',
//            })
        },
        removeChild(item) {
            // 获取父组件的model.children
            let parent_model_children = this.$parent.model[this.childrenKey]

            var i = parent_model_children.indexOf(item);
            if(i !== -1) {
              parent_model_children.splice(i, 1);
            }

            this.$parent.model[this.childrenKey] = [...parent_model_children];
        },
        dragStart(e) {
            // fromData = this.model
            e.dataTransfer.effectAllowed = "move";
            e.dataTransfer.setData("nottext", e.target.innerHTML);
            return true
        },
        dragEnd(e) {
            fromData = undefined
            toData = undefined
            fromParentModelChildren = undefined
        },
        dragOver(e) {
            e.preventDefault()
            return true
        },
        dragOverPlus(e) {
            e.preventDefault()
        },
        dragEnterPlus(e) {
        },
        dragEnter(e) {
            toData = this.model
            let rootTree = this.findRoot()
            rootTree.exchangeData(rootTree.$parent, fromData, toData)
        },
        dragLeave(e) {
          if(this.$parent.model) {
            fromData = this.model
            fromParentModelChildren = this.$parent.model[this.childrenKey]
            //e.target.style.background = '#7B1FA2';
          }
        },
        drop(e) {
            toData = this.model
            // e.target.style.background = '#7B1FA2'
        },
        dropPlus(e) {
            // 把from节点插入到当前层级节点的最后一个
            if (this.model.hasOwnProperty(this.childrenKey)) {
                this.model[this.childrenKey].push(fromData)
            } else {
                this.model[this.childrenKey] = [fromData]
            }

            // 把from节点从之前的节点删除
            for (let i in fromParentModelChildren) {
                if (fromParentModelChildren[i] == fromData) {
                    fromParentModelChildren.splice(i, 1)
                }
            }
        }
    },
    beforeCreate() {
      this.$options.components['vue-drag-tree2'] = require('./vue-drag-tree');
    },
}
</script>

<style>
.item {
    cursor: pointer;
}

.bold {
    font-weight: bold;
}

.treeNodeText {
    margin: 2px;
    padding: 0.2rem 0.5rem;
    width: fit-content;
    background: #F9FAFC;
    color: #324057;
}

.active {
    background: #CCCCCC!important;
}

.treeMargin {
    margin-left: 1rem;
}

.changeTree {
    width: 1rem;
    color: #324057;
}

.vue-drag-node-icon {
    display: inline-block;
    width: 0;
    height: 0;
    padding-right: 3px;
    border-left: 6px solid black;
    border-top: 6px solid transparent;
    border-bottom: 6px solid transparent;
    border-right: 0 solid yellow;
    transition: transform .3s ease-in-out;
}

.nodeClicked {
    transform: rotate(90deg);
}
</style>

