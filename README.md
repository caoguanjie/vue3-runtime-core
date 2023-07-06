# @vue/runtime-core
Teleport组件存在严重的内存泄露问题，[使用 Teleport 时不会触发 onUnmounted 回调 ](https://github.com/vuejs/core/issues/6347),因为没有及时删除子后代，导致了父组件被缓存下来了。

## 截止到2023年7月6日，官方还没有解决Teleport组件
由于项目利用了大量了element-plus的弹窗式组件，导致所有页面都也反复地缓存，导致占用了大量的内存得不到释放，从而内存溢出导致页面崩溃，因此修改vue3的`runtime-core`源码,从而达到临时修复的目的


只要修改的关键源码是：

**旧的代码：**
```js
// 路径：src/components/Teleport.ts
const isTeleportDisabled = (props) => props && (props.disabled || props.disabled === "");

if (doRemove || !isTeleportDisabled(props)) {
    hostRemove(anchor);
    if (shapeFlag & 16) {
      for (let i = 0; i < children.length; i++) {
        const child = children[i];
        unmount(
          child,
          parentComponent,
          parentSuspense,
          true,
          !!child.dynamicChildren
        );
      }
    }
  }
```

**修改后的新代码**
```js
doRemove && hostRemove(anchor);
if (shapeFlag & 16 /* ShapeFlags.ARRAY_CHILDREN */) {
    const shouldRemove = doRemove || !isTeleportDisabled(props);
    for (let i = 0; i < children.length; i++) {
        const child = children[i];
        unmount(child, parentComponent, parentSuspense, shouldRemove, !!child.dynamicChildren);
    }
}
```

为什么修改这里呢？我们以el-dialog的例子来测试
当我们使用apeendToBody的时候，传给isTeleportDisabled的props如图，props里面的disabled属性为false，可以参考


![图 0](/src/assets/images20230706112601.jpg)  

当没有使用appendToBody的时候，props里面的disabled为true，导致isTeleportDisabled判断为true，导致无法执行umount

![图 1](/src/assets/2111688614211_.pic.jpg)  

从上面代码可以知道：
```js
const isTeleportDisabled = (props) => props && (props.disabled || props.disabled === "");
```

## 注意事项
这个runtime-core是修改的时候，注意版本号，如果在官方还没有修复前，如果vue的版本升级到新版的后，要对应修改这个文件`src/components/Teleport.ts`，然后进行发布。

具体步骤是：
1. 先到vue的github源码[地址](https://github.com/vuejs/core)
2. 下拉分支，找到对应版本的分支源码进行下载。例如：`v3.2.47`
3. 源码下载后，解压出来，就可以执行命令：

```shell
npm i pnpm
pnpm i
pnpm run release
# 如果执行上一个页面报错，记得要要执行git init然后全部提交到本地仓库，才可以执行上面的命令
```