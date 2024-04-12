# tldraw 文档
## Editor
使用[Editor](https://tldraw.dev/reference/editor/Editor) 类，通过这个类可以控制编辑器内部的`状态`、提交修改以及对编辑器状态的变化做出响应。

`Editor`的功能非常强大，可以通过[Edtor.createShapes](https://tldraw.dev/reference/editor/Editor#createShapes) 创建图形（shape)，可以通过`Editor.deleteShapes`删除图形，通过`Editor.getCurrentPageShapesSorted`对当前页面的图形进行排序。

## Store

编辑器状态记录在`Editor.store`属性内，数据以`JSON`对象的形式保存。举例，`store`为每一个页面保存一个`TLPage`的记录，一个`TLInstancePageState`的记录存储编辑器页面的状态和唯一的`TLInstance`保存编辑器的实例。

编辑器暴露出许多的`computed`值，这些值是通过其他值计算出来的。例如`Edtor.getSelectedShapeIds`方法可以获取当前页面选中图形的`id`。

```jsx
import { track, useEditor } from 'tldraw'

export const SelectedShapeIdsCount = track(() => {
	const editor = useEditor()

	return <div>{editor.getSelectedShapeIds().length}</div>
})
// 这里的`track`函数与`vue`中的`watchEffect`相似，如果编辑器的状态发生变化那么使用`track`包裹的组件会重新渲染
```

## 修改状态

`Editor`有大量的方法可以更新内部状态，例如，你可以使用`Editor.setSelectedShapes`修改当前页的选中图形，也可以使用`Editor.select`、`Editor.selectAll`、`Editor.selectNone`等操作图形的选中。
```js

editor.selectNone() // 取消所有选中
editor.select(myShapeId, myOtherShapeId) // 设置图形的选中
editor.getSelectedShapes() // [myShapeId, myOtherShapeId]
```
每次对状态的修改都会在一个事务（transation）内做出响应。所以最好通过`Editor.batch`方法将状态变化打包到一个事务里面。最好是所有可能的修改都打包处理，这样可以减少撤销重作的步骤以及持久化的数据。

## 监听变化

可以使用`Store.listen`监听`Editor.store`的变化。每个事务完成时编辑器都会带一个历史记录调用监听函数。这个历史记录包含`added`、`changed`、`deleted·还会表示这些变化是谁产生的例如`user`或者`remote`

```js
editor.store.listen(entry=> {
  entry // {changes, source}  changes 是事务结束的变化，source 是谁产生的变化
  // 这里的变化会包括鼠标悬停移动的数据，所以这个监听函数产生的数据量很大
})
```

## 远程变化

默认情况下，编辑器的变化都来自于`user`，当然也可以使用`Store.mergeRemotechanges`合并来自远程的状态变化。合并完成后`Store.listen`接收到的`source`属性会被标识成`remote`。

## 撤销重作

历史记录包含两种数据类型：`marks`和`commands`。`commands`有自己的`undo`和`redo`方法。

可以使用`Editor.mark`在历史记录中添加一个标记
```js

editor.mark('my-id')
// 做一些操作
editor.bailToMark('my-id')
```

当调用`Editor.undo`时，编辑器会撤销所有的操作直到上一个标记或者历史记录的栈顶。当调用`Editor.redo`时，编辑器会重作所有的操作直到下一个标记或者历史记录的栈底。

```js
// 标记 duplicate everything
editor.mark('duplicate everything')

editor.selectAll()
editor.duplicateShapes(editor.getSelectedShapeIds()) // 复制选中图形
// 栈底

editor.undo() // 返回标记 duplicate everything
editor.redo() // 返回栈底
```

可以调用`Editor.bail`撤销操作并删除第一个标记到当前标记的所有操作记录
```js

// 标记 duplicate everything
editor.mark('duplicate everything')

editor.selectAll()
editor.duplicateShapes(editor.getSelectedShapeIds()) // 复制选中图形
// 栈底

editor.bail() // 返回标记 duplicate everything
editor.redo() // 不做任何操作
```
可以使用`Editor.bailToMark` 撤销并删除栈底到指定标记的所有操作记录。
```js

// 标记 first
editor.mark('first')
editor.selectAll()
// 标记second
editor.mark('second')
editor.duplicateShapes(editor.getSelectedShapeIds())
// 栈底

editor.bailToMark('first') // 回到标记first
```
## 事件

`Editor`接收`Editor.dispatch`释放的事件。当`Editor`接收到事件时会首先更新内部的`Editor.inputs` 然后将事件传送进编辑器的`状态图`（state chart)。

不要自己写`Editor.dispatch`触发自定义事件，这样需要在状态图里面写代码处理这些事件。

### 状态图

`状态图`是一个`StateNode`的树状结构。当使用编辑器工具如选中工具、画图工具时，会进入到状态图。用户的交互操作像移动鼠标也会进入到这个状态图，状态图里的状态发生变化进而激活对应的节点。

树图里的每个状态节点都可以是激活或者不激活状态，每个节点也可以有0个或者多个子节点。一个状态节点激活时，子节点也会激活。一个节点接收到父级节点的事件时，它可以先处理事件然后将事件派发给激活的子节点。节点可以任意处理传递过来的事件如忽略该事件、更新store中的记录、激活某些其他节点。

当交互事件传入后，事件通过编辑器的`根状态节点`向下传递，直到事件进入状态图的叶子节点或者某个节点提交了一个事务（transaction)。

![event](./events.png) 

## 路径

可以通过`editor.root.path`获取编辑器当前激活节点的`路径`。上图状态图的路径就是`root.select.idle`。

通过`Editor.isIn`查看一条路径是否是激活的，通过`Editor.isInAny`查看多条路径是否有一条是激活的。
```js
editor.store.path // root.select.idle
editor.isIn('root.select') // true
editor.isIn('root.select.pointing_shape') // false
editor.isInAny('editor.select.idle', 'editor.select.pointing_shape') // true

```

可以看到传入的路径可以是`全路径`也可能是`非全路径`，对于完整路径`root.select.idle`，传入的无论是`root`，还是`root.select`，亦或是`root.select.idle` `Editor.isIn`返回的都是`true`

> 可以通过`Editor.getCurrentToolId`获取编辑器当前选中的工具

```js

import { track, useEditor } from 'tldraw'

export const BubbleToolUi = track(() => {
	const editor = useEditor()

    // 当bubble 工具激活时才展示UI
	if (!editor.getCurrentToolId() === 'bubble') return null
	return <div>Creating bubble</div>
})

```

## 输入

`Editor.inputs`负责保存用户当前输入的状态信息，这些状态信息包括：当前坐标位置（页面位置或者屏幕位置）、按下的按键、鼠标连击的状态、是否拖动缩放等。

注意到像`shift`、`alt` 等这些辅助键按下时会有短暂的延迟，当按下`shift`键时，`editor.inputs.shiftKey` 会一直是`true`的状态，直到松开100毫秒之后。

## 编辑器实例的状态

`Editor.getInstanceState`可以获取每一个编辑器实例的状态。用户在多个标签页使用了同一个编辑器，或者在一个页面有多个编辑器，这些情况编辑器都有自己的实例状态。

## 用户偏好

用户偏好在所有的编辑器实例中都是共享的，见[TLUserPreferences](https://tldraw.dev/reference/editor/TLUserPreferences)。

## 关于编辑器一些常见操作

### 创建图形id
使用`createShapeId` 为一个图形生成`id`

```js
import { createShapeId } from 'tldraw'

createShapeId() // `shape:some-random-uuid`
createShapeId('kyle') // `shape:kyle`
```
tldraw 中的`id`记录都会带有该记录的类型，对于`shape`图形，它的`id`都是类似于`shape:{id}`这种形式。一个记录的`id` `typescript` 类型也会包含该记录所属类型的信息，直接使用 `shape:some-id` 这样的字符串 ts 会报错，而`createShapeId`会提供类型信息。

### 创建图形

使用`Editor.createShape`或`Editor.createShapes`创建图形

```js

editor.createShapes([
	{
		id,
		type: 'geo',
		x: 0,
		y: 0,
		props: {
			geo: 'rectangle',
			w: 100,
			h: 100,
			dash: 'draw',
			color: 'blue',
			size: 'm',
		},
	},
])
```
一个图形包含的属性必须是图形全量属性([TLShapePartial](https://tldraw.dev/reference/tlschema/TLShapePartial))的一个子集。除了`type`属性，其他所有的属性都是可选的。图形对应的`ShapeUtil`会提供默认的属性，如果有属性在创建时没有提供。如果创建图形的时候没有提供`id`，他也会自动生成。

### 更新图形

使用`Editor.updateShape`或`Editor.updateShapes`更新图形

```js
editor.updateShapes([
	{
		id: shape.id, // 必需
		type: shape.type, // 必需
		x: 100,
		y: 100,
		props: {
			w: 200,
		},
	},
])

```
### 删除图形
使用 `Editor.deleteShape`或`Editor.deleteShapes`删除图形

```js
// 可以使用id 或者图形的实例
editor.deleteShapes([shape.id]) // return editor
editor.deleteShapes([shape])

```
### 获取图形

使用`Editor.getShape`获取图形
```
editor.getShape(myShapeId) // return TLShape|undefined
editor.getShape(myShape)
```
### 打开只读模式
使用`Editor.updateInstanceState`打开只读模式
```js

editor.updateInstanceState({isReadonly:true})

```

### 移动镜头

通过设置镜头(也就是画布的视口)的`x`,`y`和`zoom`值将镜头移动到指定位置

```js
editor.setCamera(0,0,1)
```

### 固定镜头
阻止用户变动相机同样也可以使用`Editor.updateInstanceState`
```js

editor.updateInstanceState({canMoveCamera: false})
```

### 开启夜间模式

使用`setUserPreferences` 开启夜间模式，因为改的是用户偏好，所以夜间模式开启后所有的编辑器实例全部共享。

```js
setUserPreferences({isDarkMode: true})
```

## 图形 Shapes

图形是页面上的看到的几乎所有东西如箭头、文字、图片等

### 图形的类型
首先需要对图形的类型做一个区分，图形类型有3种：`core`、`default`、`custom`
#### Core shapes

编辑器的`core` 图形是内置的且总是展示的。当前只有一种`core`图形 (group shape)[https://tldraw.dev/reference/tlschema/TLGroupShape]。

#### Default shapes

编辑器的`default` 图形全都默认包含在tldraw的组件里面，如`TLArrowShape`、`TLDrawShape`。它们被导出为`defaultShapeUtils`。

#### Custom shapes

自定义图形顾名思义就是开发者自己创建的，它的结构如下

### 图形对象

```js
{
    "parentId": "page:somePage",
    "id": "shape:someId",
    "typeName": "shape"
    "type": "geo",
    "x": 106,
    "y": 294,
    "rotation": 0,
    "index": "a28",
    "opacity": 1,
    "isLocked": false,
    "props": {
        "w": 200,
        "h": 200,
        "geo": "rectangle",
        "color": "black",
        "labelColor": "black",
        "fill": "none",
        "dash": "draw",
        "size": "m",
        "font": "draw",
        "text": "diagram",
        "align": "middle",
        "verticalAlign": "middle",
        "growY": 0,
        "url": ""
    },
    "meta": {},
}
```
#### 基础属性

每一个图形包含几种基本信息：`type`、位置、`rotation`、`opacity` 等等。

#### Props

每种图形都包含一些独有的信息，这些信息叫做`props`。每种图形都会有不同的`props`，例如文字图形的`props`和箭头图形的`props`会有很大不同。

#### Meta

`meta`信息不会被tldraw使用，但是可能会在被开发者使用。例如，可以在`meta`对象里面存储用户的名称，或者图形的创建时间。

### `ShapeUtil` 类

已知tldraw里面的图形本质就是一些JSON对象，而`ShapeUtil`类就是一个“操作员”，譬如当需要渲染一个文本图形的时候，先找到`TextShapeUtil`然后调用它的`ShapeUtil.component`并把文本图形的JSON对象传进去，然后就得到了渲染出来的文本图形。

### 自定义图形

接下来通过创建一个`card`图形展示如何添加自定义图形

#### 图形类型

首先创建一个`ts`类型描述这个对象

```ts
import {TLBaseShape} from 'tldraw'

type CardShap = TLBaseShape<'card', {w: number; h: number}

```

通过`TLBaseShape`定义`Card`图形的类型，它的`props`属性数据类型为`{w: number, h: number}`。这里的`card`字符串可以替换为其他任何字符串，但是props必须是JSON对象的形式。

`TLBaseShape`工具还可以添加其他的属性像`x`、`y`、`rotation`等。

#### ShapeUtil

使用`ShapeUtil`类可以根据图形的JSON对象创建和操作图形

```ts
import { HTMLContainer, ShapeUtil } from 'tldraw'

class CardShapeUtil extends ShapeUtil<CardShape> {
	static override type = 'card' as const

	getDefaultProps(): CardShape['props'] {
		return {
			w: 100,
			h: 100,
		}
	}

	getGeometry(shape: CardShape) {
		return new Rectangle2d({
			width: shape.props.w,
			height: shape.props.h,
			isFilled: true,
		})
	}

	component(shape: CardShape) {
		return <HTMLContainer>Hello</HTMLContainer>
	}

	indicator(shape: CardShape) {
		return <rect width={shape.props.w} height={shape.props.h} />
	}
}

```

这样实现了一个极简的`ShapeUtil`。这个过程中，绑定了`static` 的 `type`已匹配前面定义的`CardShape` 类型。同时还实现了`ShapeUtil.getDefaultProps`、`ShapeUtil.getBounds`、`ShapeUtil.component` 及 `ShapeUtil.indicator`等抽象方法。

#### ShapeUtils 属性

将创建好的`CardShapeUtil`传入进`Tldraw`组件的`shapeUtils`属性中。

```ts
const MyCustomShapes = [CardShapeUtil]

export default function () {
	return (
		<div style={{ position: 'fixed', inset: 0 }}>
			<Tldraw shapeUtils={MyCustomShapes} />
		</div>
	)
}
```
通过`Editor.createShapes` 可以创建一个刚刚自定义的图形。

```js

export default function () {
	return (
		<div style={{ position: 'fixed', inset: 0 }}>
			<Tldraw
				shapeUtils={MyCustomShapes}
				onMount={(editor) => {
					editor.createShapes([{ type: 'card' }])
				}}
			/>
		</div>
	)
}

```
可以在画布的最左上角看到这个新加的图形。

### Meta 信息

前面提到过可以在`meta`对象中加入开发人员需要用到的数据，但是tldraw里面的常用的图形类型中`typescript`并不能检测到`meta`这个对象，这里可以定义一个union类型，绑定meta对象类型。

```ts
type Meta = TLGeoShape["meta"] // 类型是JsonObject 类似于一个Record没有具体的字段名
type MyShapeWithMeta = TLGeoShape & { meta: { createdBy: string } }

const shape = editor.getShape<MyShapeWithMeta>(myGeoShape.id)
```
当然也可以使用`Editor.updateShapes`将meta数据写入到图形

```js
editor.updateShapes<MyShapeWithMeta>([
	{
		id: myGeoShape.id,
		type: 'geo',
		meta: {
			createdBy: 'Steve',
		},
	},
])
```
此外，通过`Editor.getInitialMetaForShape`方法可以指定初始的meta数据

```ts
editor.getInitialMetaForShape = (shape: TLShape) => {
	if (shape.type === 'text') {
		return { createdBy: currentUser.id, lastModified: Date.now() }
	} else {
		return { createdBy: currentUser.id }
	}
}
```
在使用`Editor.createShapes`创建图形的时候，图形的meta数据都会通过`Editor.getInitialMetaForShape`指定，如果这个方法没有重新定义过默认情况下会返回一个空对象。

### 基类图形

使用基类图形`BaseBoxShapeUtil`可以得到一个规则的矩形图形

### 标记

可以使用像`ShapeUtil.hideRotateHandle`的标记属性隐藏某些UI。这里的`ShapeUtil.hideRotateHandle`就是隐藏旋转操作框。

### 交互

可以打开`pointer-events`允许用户与图形内部发生交互

## Assets 资源

Assets 存储共享资源的动态记录。图片、视频这些资产不可能直接使用原生的文件，而是保存一个对资源的引用。tldraw里面把图片等资源都会保存为`base64`的形式，所以使用`assets`保持对资源的引用是有必要的。

### 例子

- [使用托管的图片](https://github.com/tldraw/tldraw/blob/main/apps/examples/src/examples/hosted-images/HostedImagesExample.tsx) 这个例子可以学习如何将图片上传到像 OSS 这类的平台并使用

- [修改资源的默认项](https://github.com/tldraw/tldraw/blob/main/apps/examples/src/examples/asset-props/AssetPropsExample.tsx) 譬如允许使用哪些图片类型

- [处理粘贴拖放外部内容](https://github.com/tldraw/tldraw/blob/main/apps/examples/src/examples/external-content-sources/ExternalContentSourcesExample.tsx) 这个例子描述了通过粘贴文本生成一个自定义的dom容器

## Tools 工具

tldraw里面`tool`是状态图里面”顶层“的节点，例如，选中、画图、箭头工具都是用户可以进入的顶层节点。

!(tools)[./tools.png]

### Tool 的类型

tldraw 自带一些内置的工具 `core tools`：选中、缩放、文本工具。这些工具一直在状态图里面。

也会有默认的工具`default tools`例如：画图、手型、箭头工具等等。这些工具会被`<Tldraw>`组件自动加入到状态图里面。

可以创建自定义的工具`custom tools`，然后通过向`Tldraw`组件的`tools`属性传入工具类数组将自定义工具加入到状态图中。

> 在UI中调用工具可以看`UI`部分的内容

### 转移工具
 
通过`editor.setCurrentTool`修改当前使用的工具

```js

editor.setCurrentTool('select')

```

也可以使用路径 id 深度转移当前工具
```js
editor.setCurrentTool('select.eraser.pointing')

```

### 工具的内部实现

每个工具要有一个`id`，这是为了在状态表识别它。

```js
class MyTool extends StateNode {
	static override id = 'my-tool'
}
```
工具可以包含`children`，例如手型工具包含三个子工具: `Idle`、`Pointing`、`Dragging`。如果一个状态节点有子节点，那么它**必须**要有一个初始（initial）状态，以便知道从哪个状态开始。

```js
class MyIdleState extends StateNode {
	static override id = 'my-idle-state'
}

class MyPointingState extends StateNode {
	static override id = 'my-pointing-state'
}

class MyTool extends StateNode {
	static override id = 'my-tool'
	static override initial = 'my-idle-state'
	static override children = [MyIdleState, MyPointingState]
}
```
### 处理事件

当编辑器从`Editor.dispatch`接收到事件时，编辑器会先更新`inputs`，然后才会传入进状态图。

从root节点开始，每个节点会先将事件处理之后再传递给激活的子节点。这个处理和传递过程会一直继续，直到某个节点没有子节点或者状态节点发生了转移。

```js

class MyIdleState extends StateNode {
	static override id = 'my-idle-state'

	onPointerDown: TLEventHandlers['onPointerDown'] = (info) => {
		console.log('world')
	}
}

class MyTool extends StateNode {
	static override id = 'my-tool'
	static override initial = 'my-idle-state'
	static override children = [MyIdleState]

	onPointerDown: TLEventHandlers['onPointerDown'] = (info) => {
		console.log('hello')
	}
}

// hello
// world

```

像上面的例子`pointer_down`事件会先进入`MyTool`，`MyTool`的`onPointerDown`会先被调用，之后`MyIdleState`的`onPointerDown`再被调用。

### 转移阻止传递

```ts

class MyIdleState extends StateNode {
	static override id = 'my-idle-state'

	onPointerDown: TLEventHandlers['onPointerDown'] = (info) => {
        // 这个方法不会允许
		console.log("this won't run")
	}
}

class MyTool extends StateNode {
	static override id = 'my-tool'
	static override initial = 'my-idle-state'
	static override children = [MyIdleState]

	onPointerDown: TLEventHandlers['onPointerDown'] = (info) => {
        // 发生了转移
		editor.setCurrentTool('select')
	}
}

```

上面的代码`MyTool`的`onPointerDown`中当前节点发生了转移，事件就不会向下传递，因此`MyIdleState`里面的`onPointerDown`不会触发。

## 持久化

持久化意味着编辑器的状态可以存储到数据库，tldraw 提供了一些选项供数据的导入导出。

### `persistenceKey` 属性

通过`persistenceKey`，`<Tldraw>`或`<TldrawEditor>`组件都支持本底持久化和跨标签页的内容同步。传递一个值给这个属性，编辑器将可以将内容存储到`IndexedDb`里面实现持久化。

```js
import { Tldraw } from 'tldraw'
import 'tldraw/tldraw.css'

export default function () {
	return (
		<div style={{ position: 'fixed', inset: 0 }}>
			<Tldraw persistenceKey="my-persistence-key" />
		</div>
	)
}
```
使用同一`persistenceKey`值的tldraw组件可以同步数据，即使这些组件分布在不同的浏览器标签页。

```js

import { Tldraw } from 'tldraw'
import 'tldraw/tldraw.css'

export default function () {
	return (
		<div style={{ position: 'fixed', inset: 0 }}>
			<div style={{ width: '50%', height: '100%' }}>
				<Tldraw persistenceKey="my-persistence-key" />
			</div>
			<div style={{ width: '50%', height: '100%' }}>
				<Tldraw persistenceKey="my-persistence-key" />
			</div>
		</div>
	)
}

```
上面的例子中，两个编辑器会在本地互相同步数据，但是他们会独自的实例状态（如：selections），两个编辑器实例通过设置相同的key同步彼此的数据。

### Snapshots 快照

通过`Editor.store`的`Store.getSnapshot`获取编辑器内容的JSON快照。

```js
function SaveButton() {
	const editor = useEditor()
	return (
		<button
			onClick={() => {
				const snapshot = editor.store.getSnapshot()
				const stringified = JSON.stringify(snapshot)
				localStorage.setItem('my-editor-snapshot', stringified)
			}}
		>
			Save
		</button>
	)
}
```
也可以通过`Store.loadSnapshot`方法将快照装载到新的编辑器。

```js
function LoadButton() {
	const editor = useEditor()
	return (
		<button
			onClick={() => {
				const stringified = localStorage.getItem('my-editor-snapshot')
				const snapshot = JSON.parse(stringified)
				editor.store.loadSnapshot(snapshot)
			}}
		>
			Load
		</button>
	)
}

```

快照包含序列化的数据和序列化的`schema`，以方便迁移。

> 默认情况下，`getSnapshot`只会返回编辑器的数据，如果想要获取不同维度的数据譬如`session`、`document`、`presence`、`all` 等，可以将对应的维度传入，以获取数据。

注意：加载快照不会重置编辑器在内存里的状态，例如，做`resizing`操作的时候加载快照可能会使编辑器崩溃。这是因为`resizing`状态可能正在操作一个图形，但是加载快照导致那个图形不存在了。

### `store` 属性

虽然可以先加载编辑器然后将数据导入到`store`，但是最好先创建`store`，然后设置数据，最后将`store`传入到编辑器。

`<Tldraw>`组件的`store`属性接受外部传入的`store`。

```ts
export default function () {
	const [store] = useState(() => {
        // 创建store
		const newStore = createTLStore({
			shapeUtils: defaultShapeUtils,
		})

        // 获取快照
		const stringified = localStorage.getItem('my-editor-snapshot')
		const snapshot = JSON.parse(stringified)

        // 加载快照
		newStore.loadSnapshot(snapshot)

		return newStore
	})

	return <Tldraw persistenceKey="my-persistence-key" store={store} />
}
```
如果不想同步地进入`store`，可以使用`TLStoreWithStatus`标识store的加载的状态。

```js
export default function () {
	const [storeWithStatus, setStoreWithStatus] = useState<TLStoreWithStatus>({
		status: 'loading',
	})

	useEffect(() => {
		let cancelled = false
		async function loadRemoteSnapshot() {
            // 获取快照
			const snapshot = await getRemoteSnapshot()
			if (cancelled) return

            // 创建store
			const newStore = createTLStore({
				shapeUtils: defaultShapeUtils,
			})

            // 加载快照
			newStore.loadSnapshot(snapshot)

            // 带status 更新store
			setStoreWithStatus({
				store: newStore,
				status: 'ready',
			})
		}

		loadRemoteSnapshot()

		return () => {
			cancelled = true
		}
	})

	return <Tldraw persistenceKey="my-persistence-key" store={storeWithStatus} />
}
```

一个比较好的[例子](https://github.com/tldraw/tldraw-yjs-example)

## UI

UI包括菜单、工具栏、快捷键、分析事件。

### 隐藏UI

使用`<Tldraw>`组件里的`hideUi`属性隐藏UI，这个属性可以关闭可见的所有组件和快捷键。

```js

function Example() {
	return <Tldraw hideUi />
}

```
可以看这个[例子](https://examples.tldraw.com/hide-ui)，在编辑器里面无法选中任何工具，也不能使用快捷键。但是可以通过`setCurrentTool`设置工具，譬如
```js
editor.setCurrentTool('draw')

```
开始绘画。

一个自定义UI的[例子](https://examples.tldraw.com/custom-ui)仓库[地址](https://github.com/tldraw/tldraw/blob/main/apps/examples/src)

### 事件

`<Tldraw>`组件有一个`onUiEvent`属性，对这个属性传入回调函数时，UI上的事件会触发该回调函数。
```js
function Example() {
	function handleEvent(name, data) {
		// do something with the event
	}

	return <Tldraw onUiEvent={handleEvent} />
}
```
`onUiEvent`函数被触发时会带有事件的名称和与该事件相关的数据，例如事件触发源(如 menu或context-menu)、每种事件的特有数据（如`align-shapes`事件的方向）等。

注意：`onUiEvent`只会在用户和UI交互时才会触发，如果直接调用`Editor.alignShapes`方法并不会触发`onUiEvent`回调。

### Overrides

`<Tldraw>`组件的菜单可以通过`overrides`属性被覆盖重写。该属性传入`TLUiOverrides`对象，该对象包含每种菜单UI的方法，如`toolbar`、`keyboardShortcutsMenu`。

#### Actions

UI有一系列被菜单和快捷键公用的`actions`。这些actions可以通过对`overrides.actions`重新传入方法而重写。

新建、更新或者删除actions时需要提供一个`actions`方法，这个方法有两个参数`editor`和`actions`。actions参数时默认的actions对象。该方法需要返回修改后的actions对象。

```js
const myOverrides: TLUiOverrides = {
	actions(editor, actions) {
        // 可以删除掉某个action，但是需要也删除引用这个action的菜单项
		delete actions['insert-embed']

        // 创建一个新的action 或者替换掉原来的
		actions['my-new-action'] = {
			id: 'my-new-action',
			label: 'My new action',
			readonlyOk: true,
			kbd: '$u',
			onSelect(source: any) {
                // 执行想要的任何行为
				window.alert('My new action just happened!')
			},
		}
		return actions
	},
}
```
`actions`是`TLUiActionItem`的map对象，这个对象的key值就是action的id。

#### Tools

`Tools`的执行方式与上面的actions一样。可以传入`tools`方法重写默认的工具方法。它的也是有两个参数`editor`和`tools`，返回一个修改的`tools`对象。

```js
const myOverrides: TLUiOverrides = {
	tools(editor, tools) {
        // 为tools加入一个card项
		tools.card = {
			id: 'card',
			icon: 'color',
			label: 'tools.card',
			kbd: 'c',
			onSelect: () => {
                // 执行想要的任何行为
				editor.setCurrentTool('card')
			},
		}
		return tools
	},
}
```

`tools`是`TLUiToolItem`的map对象，这个对象的key值就是tool的id。

#### 国际化
`translations`属性接受新加入的翻译字段，如果想要引用`tools.card`的工具展示一个英文文案可以给这个字段加上英文翻译。
```js
const myOverrides: TLUiOverrides = {
	translations: {
		en: {
			'tools.card': 'Card',
		},
	},
}
```
















































































