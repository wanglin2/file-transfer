富文本编辑器相信大家都用过，相关的开源项目也很多，虽然具体的实现不一样，但是大部分都是使用`DOM`实现的，但其实还有一种实现方式，那就是使用`HTML5`的`canvas`，本文会带大家使用`canvas`简单实现一个类似`Word`的富文本编辑器，话不多说，开始吧。

最终效果抢先看：[https://wanglin2.github.io/canvas-editor-demo/](https://wanglin2.github.io/canvas-editor-demo/)。



# 基本数据结构

首先要说明我们渲染的数据不是`html`字符串，而是结构化的`json`数据，为了简单起见，暂时只支持文本，完整的结构如下：

```js
[
    {
        value: '理',// 文字内容
        color: '#000',// 文字颜色
        size: 16// 文字大小
    },
    {
        value: '想',
        background: 'red'，// 文字背景颜色
        lineheight: 1// 行高，倍数
    },
    {
        value: '青',
        bold: true// 加粗
    },
    {
        value: '\n'// 换行
    },
    {
        value: '年',
        italic: true// 斜体
    },
    {
        value: '实',
        underline: true// 下划线
    },
    {
        value: '验',
        linethrough: true// 中划线
    },
    {
        value: '室',
        fontfamily: ''// 字体
    }
]
```

可以看到我们把文本的每个字符都作为一项，`value`属性保存字符，其他的文字样式通过各自的属性保存。

我们的`canvas`编辑器原理很简单，实现一个渲染方法`render`，能够将上述的数据渲染出来，然后监听鼠标的点击事件，在点击的位置渲染一个闪烁的光标，再监听键盘的输入事件，根据输入、删除、回车等不同类型的按键事件更新我们的数据，再将画布清除，重新渲染即可达到编辑的效果，实际上就是数据驱动视图，相信大家都很熟悉了。



# 绘制页面样式

我们模拟的是`Word`，先来看一下`Word`的页面样式：

![image-20230206153404548](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230206153404548.png)

基本特征如下：

- 支持分页
- 四周存在内边距
- 四个角有个直角折线

当然，还支持页眉、页脚、页码，这个我们就不考虑了。

所以我们的编辑器类如下：

```js
class CanvasEditor {
  constructor(container, data, options = {}) {
    this.container = container // 容器元素
    this.data = data // 数据
    this.options = Object.assign(
      {
        pageWidth: 794, // 纸张宽度
        pageHeight: 1123, // 纸张高度
        pagePadding: [100, 120, 100, 120], // 纸张内边距，分别为：上、右、下、左
		pageMargin: 20,// 页面之间的间隔
        pagePaddingIndicatorSize: 35, // 纸张内边距指示器的大小，也就是四个直角的边长
        pagePaddingIndicatorColor: '#BABABA', // 纸张内边距指示器的颜色，也就是四个直角的边颜色
      },
      options
    )
    this.pageCanvasList = [] // 页面canvas列表
    this.pageCanvasCtxList = [] // 页面canvas绘图上下文列表
  }
}
```

接下来添加一个创建页面的方法：

```js
class CanvasEditor {
    // 创建页面
    createPage() {
        let { pageWidth, pageHeight, pageMargin } = this.options
        let canvas = document.createElement('canvas')
        canvas.width = pageWidth
    	canvas.height = pageHeight
        canvas.style.cursor = 'text'
        canvas.style.backgroundColor = '#fff'
        canvas.style.boxShadow = '#9ea1a566 0 2px 12px'
        canvas.style.marginBottom = pageMargin + 'px'
        this.container.appendChild(canvas)
        let ctx = canvas.getContext('2d')
        this.pageCanvasList.push(canvas)
        this.pageCanvasCtxList.push(ctx)
    }
}
```

很简单，创建一个`canvas`元素，设置宽高及样式，然后添加到容器元素，最后收集到列表中。

接下来是绘制四个直角的方法：

```js
class CanvasEditor {
    // 绘制页面四个直角指示器
    renderPagePaddingIndicators(pageNo) {
        let ctx = this.pageCanvasCtxList[pageNo]
        if (!ctx) {
            return
        }
        let {
            pageWidth,
            pageHeight,
            pagePaddingIndicatorColor,
            pagePadding,
            pagePaddingIndicatorSize
        } = this.options
        ctx.save()
        ctx.strokeStyle = pagePaddingIndicatorColor
        let list = [
            // 左上
            [
                [pagePadding[3], pagePadding[0] - pagePaddingIndicatorSize],
                [pagePadding[3], pagePadding[0]],
                [pagePadding[3] - pagePaddingIndicatorSize, pagePadding[0]]
            ],
            // 右上
            [
                [pageWidth - pagePadding[1], pagePadding[0] - pagePaddingIndicatorSize],
                [pageWidth - pagePadding[1], pagePadding[0]],
                [pageWidth - pagePadding[1] + pagePaddingIndicatorSize, pagePadding[0]]
            ],
            // 左下
            [
                [pagePadding[3], pageHeight - pagePadding[2] + pagePaddingIndicatorSize],
                [pagePadding[3], pageHeight - pagePadding[2]],
                [pagePadding[3] - pagePaddingIndicatorSize, pageHeight - pagePadding[2]]
            ],
            // 右下
            [
                [pageWidth - pagePadding[1], pageHeight - pagePadding[2] + pagePaddingIndicatorSize],
                [pageWidth - pagePadding[1], pageHeight - pagePadding[2]],
                [pageWidth - pagePadding[1] + pagePaddingIndicatorSize, pageHeight - pagePadding[2]]
            ]
        ]
        list.forEach(item => {
            item.forEach((point, index) => {
                if (index === 0) {
                    ctx.beginPath()
                    ctx.moveTo(...point)
                } else {
                    ctx.lineTo(...point)
                }
                if (index >= item.length - 1) {
                    ctx.stroke()
                }
            })
        })
        ctx.restore()
    }
}
```

代码很多，但是逻辑很简单，就是先计算出每个直角的三个端点坐标，然后调用`canvas`绘制线段的方法绘制出来即可。效果如下：

![image-20230206170232092](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230206170232092.png)



# 渲染内容

接下来就到我们的核心了，把前面的数据通过`canvas`绘制出来，当然绘制出来是结果，中间需要经过一系列步骤。我们的大致做法大致如下：

1.遍历数据列表，计算出每项数据的字符宽高

2.根据页面宽度，计算出每一行包括的数据项，同时计算出每一行的宽度和高度，高度即为这一行中最高的数据项的高度

3.逐行进行绘制，同时根据页面高度判断，如果超出当前页，则绘制到下一页

## 计算行数据

`canvas`提供了一个`measureText`方法用来测量文本，但是返回只有`width`，没有`height`，那么怎么得到文本的高度呢，其实可以通过返回的另外两个字段`actualBoundingBoxAscent`、`actualBoundingBoxDescent`，这两个返回值的含义是从`textBaseline`属性标明的水平线到渲染文本的矩形边界顶部、底部的距离，示意图如下：

![image-20230207093046160](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230207093046160.png)

很明显，文本的高度可以通过`actualBoundingBoxAscent + actualBoundingBoxDescent`得到。

当然要准确获取一个文本的宽高，跟它的字号、字体等都相关，所以通过这个方法测量前需要先设置这些文本样式，这个可以通过`font`属性进行设置，`font`属性是一个复合属性，取值和`css`的`font`属性是一样的，示例如下：

```js
ctx.font = `italic 400 12px sans-serif`
```

从左到右依次是`font-style`、`font-weight`、`font-size`、`font-family`。

所以我们需要写一个方法来拼接这个字符串：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.options = Object.assign(
            {
                // ...
                color: '#333',// 文字颜色
                fontSize: 16,// 字号
                fontFamily: 'Yahei',// 字体
            },
            options
        )
        // ...
    }

    // 拼接font字符串
    getFontStr(element) {
        let { fontSize, fontFamily } = this.options
        return `${element.italic ? 'italic ' : ''} ${element.bold ? 'bold ' : ''} ${
            element.size || fontSize
        }px  ${element.fontfamily || fontFamily} `
    }
}
```

需要注意的是即使在`font`中设置了行高在`canvas`中也不会生效，因为`canvas`规范强制把它设成了`normal`，无法修改，那么怎么实现行高呢，很简单，自己处理就好了，比如行高`1.5`，那么就是文本实际的高度就是`文本高度 * 1.5`。

```js
this.options = Object.assign(
    {
        // ...
        lineHeight: 1.5,// 行高，倍数
    },
    options
)
```

现在可以来遍历数据进行计算了：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.rows = [] // 渲染的行数据
    }

    // 计算行渲染数据
    computeRows() {
        let { pageWidth, pagePadding, lineHeight } = this.options
        // 实际内容可用宽度
        let contentWidth = pageWidth - pagePadding[1] - pagePadding[3]
        // 创建一个临时canvas用来测量文本宽高
        const canvas = document.createElement('canvas')
        const ctx = canvas.getContext('2d')
        // 行数据
        let rows = []
        rows.push({
            width: 0,
            height: 0,
            elementList: []
        })
        this.data.forEach(item => {
            let { value, lineheight } = item
            // 实际行高倍数
            let actLineHeight = lineheight || lineHeight
            // 获取文本宽高
            let font = this.getFontStr(item)
            ctx.font = font
            let { width, actualBoundingBoxAscent, actualBoundingBoxDescent } =
                ctx.measureText(value)
            // 尺寸信息
            let info = {
                width,
                height: actualBoundingBoxAscent + actualBoundingBoxDescent,
                ascent: actualBoundingBoxAscent,
                descent: actualBoundingBoxDescent
            }
            // 完整数据
            let element = {
                ...item,
                info,
                font
            }
            // 判断当前行是否能容纳
            let curRow = rows[rows.length - 1]
            if (curRow.width + info.width <= contentWidth && value !== '\n') {
                curRow.elementList.push(element)
                curRow.width += info.width
                curRow.height = Math.max(curRow.height, info.height * actLineHeight)
            } else {
                rows.push({
                    width: info.width,
                    height: info.height * actLineHeight,
                    elementList: [element]
                })
            }
        })
        this.rows = rows
    }
}
```

创建一个临时的`canvas`来测量文本字符的宽高，遍历所有数据，如果当前行已满，或者遇到换行符，那么新创建一行。行高由这一行中最高的文字的高度和行高倍数相乘得到。

![image-20230207103407962](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230207103407962.png)

## 渲染行数据

得到了行数据后，接下来就可以绘制到页面上了。

```js
class CanvasEditor {
    // 渲染
    render() {
        this.computeRows()
        this.renderPage()
    }

    // 渲染页面
    renderPage() {
        let { pageHeight, pagePadding } = this.options
        // 页面内容实际可用高度
        let contentHeight = pageHeight - pagePadding[0] - pagePadding[2]
        // 从第一页开始绘制
        let pageIndex = 0
        let ctx = this.pageCanvasCtxList[pageIndex]
        // 当前页绘制到的高度
        let renderHeight = 0
        // 绘制四个角
        this.renderPagePaddingIndicators(pageIndex)
        this.rows.forEach(row => {
            if (renderHeight + row.height > contentHeight) {
                // 当前页绘制不下，需要创建下一页
                pageIndex++
                // 下一页没有创建则先创建
                let page = this.pageCanvasList[pageIndex]
                if (!page) {
                    this.createPage()
                }
                this.renderPagePaddingIndicators(pageIndex)
                ctx = this.pageCanvasCtxList[pageIndex]
                renderHeight = 0
            }
            // 绘制当前行
            this.renderRow(ctx, renderHeight, row)
            // 更新当前页绘制到的高度
            renderHeight += row.height
        })
    }
}
```

很简单，遍历行数据进行渲染，用一个变量`renderHeight`记录当前页已经绘制的高度，如果超出一页的高度，那么创建下一页，复位`renderHeight`，重复绘制步骤直到所有行都绘制完毕。

绘制行数据调用的是`renderRow`方法：

```js
class CanvasEditor {
    // 渲染页面中的一行
    renderRow(ctx, renderHeight, row) {
        let { color, pagePadding } = this.options
        // 内边距
        let offsetX = pagePadding[3]
        let offsetY = pagePadding[0]
        // 当前行绘制到的宽度
        let renderWidth = offsetX
        renderHeight += offsetY
        row.elementList.forEach(item => {
            // 跳过换行符
            if (item.value === '\n') {
                return
            }
            ctx.save()
            // 渲染文字
            ctx.font = item.font
            ctx.fillStyle = item.color || color
            ctx.fillText(item.value, renderWidth, renderHeight)
            // 更新当前行绘制到的宽度
            renderWidth += item.info.width
            ctx.restore()
        })
    }
}
```

跟绘制页的逻辑是一样的，也是通过一个变量来记录当前行绘制到的距离，然后调用`fillText`绘制文本，背景、下划线、删除线我们待会再补充，先看一下当前效果：

![image-20230207135851502](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230207135851502.png)

从第一行可以发现一个很明显的问题，文本绘制位置不对，超出了内容区域，绘制到了内边距里，难道是我们计算位置出了问题了，我们不妨渲染一根辅助线来看看：

```js
renderRow(ctx, renderHeight, row) {
    // ...
    // 辅助线
    ctx.moveTo(pagePadding[3], renderHeight)
    ctx.lineTo(673, renderHeight)
    ctx.stroke()
}
```

![image-20230207140307754](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230207140307754.png)

可以看到辅助线的位置是正确的，那么代表我们的位置计算是没有问题的，这其实跟`canvas`绘制文本时的文本基线有关，也就是`textBaseline`属性，默认值为`alphabetic`，各个取值的效果如下：

![image-20230207140506271](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230207140506271.png)

知道了原因，修改也就不难了，我们可以修改`fillText`的`y`参数，在前面的基础上加上行的高度：

```js
ctx.fillText(item.value, renderWidth, renderHeight + row.height)
```

![image-20230208100115710](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208100115710.png)

比前面好了，但是依然有问题，问题出在行高，`始终相信`那一行设置了`3`倍的行高，我们显然是希望文本在行内垂直居中的，现在还是贴着行的底部，这个可以通过行的实际高度减去文本的最大高度，再除以二，累加到`fillText`的`y`参数上就可以了，但是行数据`row`上只保存了最终的实际高度，文本高度并没有保存，所以需要先修改一下`computeRows`方法：

```js
computeRows() {
    rows.push({
        width: 0,
        height: 0
        originHeight: 0, // 没有应用行高的原始高度
        elementList: []
    })

    if (curRow.width + info.width <= contentWidth && value !== '\n') {
        curRow.elementList.push(element)
        curRow.width += info.width
        curRow.height = Math.max(curRow.height, info.height * actLineHeight) // 保存当前行实际最高的文本高度
        curRow.originHeight = Math.max(curRow.originHeight, info.height) // 保存当前行原始最高的文本高度
    } else {
        rows.push({
            width: info.width,
            height: info.height * actLineHeight,
            originHeight: info.height,
            elementList: [element]
        })
    }
}
```

我们增加一个`originHeight`来保存没有应用行高的原始高度，这样我们就可以在`renderRow`方法里使用了：

```js
ctx.fillText(
    item.value,
    renderWidth,
    renderHeight + row.height - (row.height - row.originHeight) / 2
)
```

![image-20230208100602011](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208100602011.png)

可以看到虽然正常一些了，但仔细看还是有问题，文字还是没有在行内完全居中，这其实也跟文字基线`textBaseline`有关，因为有的文字部分会绘制到基线下面，导致整体偏下，你可能觉得把`textBaseline`设为`bottom`就可以了，但实际上这样它又会偏上了：

![image-20230208100917476](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208100917476.png)

怎么办呢，可以这么做，在计算行数据时，再增加一个字段`descent`，用来保存该行元素中最大的`descent`值，然后在绘制该行文字`y`坐标在前面的基础上再减去`row.descent`，修改`computeRows`方法：

```js
computeRows() {
    // ...
    rows.push({
        width: 0,
        height: 0,
        originHeight: 0,
        descent: 0,// 行内元素最大的descent
        elementList: []
    })
    this.data.forEach(item => {
        // ...
        if (curRow.width + info.width <= contentWidth && value !== '\n') {
            // ...
            curRow.descent = Math.max(curRow.descent, info.descent)// 保存当前行最大的descent
        } else {
            rows.push({
                // ...
                descent: info.descent
            })
        }
    })
    // ...
}
```

然后绘制文本时减去该行的`descent：`

```js
ctx.fillText(
    item.value,
    renderWidth,
    renderHeight + row.height - (row.height - row.originHeight) / 2 - row.descent
)
```

效果如下：

![image-20230208101522742](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208101522742.png)

接下来补充绘制背景、下划线、删除线的逻辑：

```js
renderRow(ctx, renderHeight, row) {
    // 渲染背景
    if (item.background) {
        ctx.save()
        ctx.beginPath()
        ctx.fillStyle = item.background
        ctx.fillRect(
            renderWidth,
            renderHeight,
            item.info.width,
            row.height
        )
        ctx.restore()
    }
    // 渲染下划线
    if (item.underline) {
        ctx.save()
        ctx.beginPath()
        ctx.moveTo(renderWidth, renderHeight + row.height)
        ctx.lineTo(renderWidth + item.info.width, renderHeight + row.height)
        ctx.stroke()
        ctx.restore()
    }
    // 渲染删除线
    if (item.linethrough) {
        ctx.save()
        ctx.beginPath()
        ctx.moveTo(renderWidth, renderHeight + row.height / 2)
        ctx.lineTo(renderWidth + item.info.width, renderHeight + row.height / 2)
        ctx.stroke()
        ctx.restore()
    }
    // 渲染文字
    // ...
}
```

很简单，无非就是绘制矩形和线段：

![image-20230208101707125](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208101707125.png)

到这里，渲染的功能就完成了，当然，如果要重新渲染，还要一个擦除的功能：

```js
class CanvasEditor {
    // 清除渲染
    clear() {
        let { pageWidth, pageHeight } = this.options
        this.pageCanvasCtxList.forEach(item => {
            item.clearRect(0, 0, pageWidth, pageHeight)
        })
    }
}
```

遍历所有页面，调用`clearRect`方法进行清除，`render`方法也需要修改一下，每次渲染前都先进行清除及一些复位工作：

```js
render() {
    this.clear()
    this.rows = []
    this.positionList = []
    this.computeRows()
    this.renderPage()
}
```



# 渲染光标

要渲染光标，首先要计算出光标的位置，以及光标的高度，具体来说，步骤如下：

1.监听`canvas`的`mousedown`事件，计算出鼠标按下的位置相对于`canvas`的坐标

2.遍历`rows`，遍历`rows.elementList`，判断鼠标点击在哪个`element`内，然后计算出光标坐标及高度

3.定位并渲染光标

为了方便我们后续的遍历和计算，我们添加一个属性`positionList`，用来保存所有元素，并且预先计算一些信息，省去后面遍历时重复计算。

```js
class CanvasEditor {
  constructor(container, data, options = {}) {
      // ...
      this.positionList = []// 定位元素列表
      // ...
  }
}
```

在`renderRow`方法里进行收集：

```js
class CanvasEditor {
    // 新增两个参数，代表当前所在页和行数
    renderRow(ctx, renderHeight, row, pageIndex, rowIndex) {
        // ...
        row.elementList.forEach(item => {
            // 收集positionList
            this.positionList.push({
                ...item,
                pageIndex, // 所在页
                rowIndex, // 所在行
                rect: {// 包围框
                    leftTop: [renderWidth, renderHeight],
                    leftBottom: [renderWidth, renderHeight + row.height],
                    rightTop: [renderWidth + item.info.width, renderHeight],
                    rightBottom: [
                        renderWidth + item.info.width,
                        renderHeight + row.height
                    ]
                }
            })
            // ...
        })
        // ...
    }
}
```

保存每个元素所在的页、行、包围框位置信息。

## 计算光标坐标

先给`canvas`绑定`mousedown`事件，可以在创建页面的时候绑定：

```js
class CanvasEditor {
    // 新增了要创建的页面索引参数
    createPage(pageIndex) {
        // ...
        canvas.addEventListener('mousedown', (e) => {
            this.onMousedown(e, pageIndex)
        })
        // ...
    }
}
```

创建页面时需要传递要创建的页面索引，这样方便在事件里能直接获取到点击的页面，鼠标的事件位置默认是相对于浏览器窗口的，需要转换成相对于`canvas`的：

```js
class CanvasEditor {
    // 将相对于浏览器窗口的坐标转换成相对于页面canvas
    windowToCanvas(e, canvas) {
        let { left, top } = canvas.getBoundingClientRect()
        return {
            x: e.clientX - left,
            y: e.clientY - top
        }
    }
}
```

接下来计算鼠标点击位置所在的元素索引，遍历`positionList`，判断点击位置是否在某个元素包围框内，如果在的话再判断是否是在这个元素的前半部分，是的话点击元素就是前一个元素，否则就是该元素；如果不在，那么就判断点击所在的那一行是否存在元素，存在的话，点击元素就是这一行的最后一个元素；否则点击就是这一页的最后一个元素：

```js
class CanvasEditor {
    // 页面鼠标按下事件
    onMousedown(e, pageIndex) {
        let { x, y } = this.windowToCanvas(e, this.pageCanvasList[pageIndex])
        let positionIndex = this.getPositionByPos(x, y, pageIndex)
    }

    // 获取某个坐标所在的元素
    getPositionByPos(x, y, pageIndex) {
        // 是否点击在某个元素内
        for (let i = 0; i < this.positionList.length; i++) {
            let cur = this.positionList[i]
            if (cur.pageIndex !== pageIndex) {
                continue
            }
            if (
                x >= cur.rect.leftTop[0] &&
                x <= cur.rect.rightTop[0] &&
                y >= cur.rect.leftTop[1] &&
                y <= cur.rect.leftBottom[1]
            ) {
                // 如果是当前元素的前半部分则点击元素为前一个元素
                if (x < cur.rect.leftTop[0] + cur.info.width / 2) {
                    return i - 1
                }
                return i
            }
        }
        // 是否点击在某一行
        let index = -1
        for (let i = 0; i < this.positionList.length; i++) {
            let cur = this.positionList[i]
            if (cur.pageIndex !== pageIndex) {
                continue
            }
            if (y >= cur.rect.leftTop[1] && y <= cur.rect.leftBottom[1]) {
                index = i
            }
        }
        if (index !== -1) {
            return index
        }
        // 返回当前页的最后一个元素
        for (let i = 0; i < this.positionList.length; i++) {
            let cur = this.positionList[i]
            if (cur.pageIndex !== pageIndex) {
                continue
            }
            index = i
        }
        return index
    }
}
```

![image-20230207165012007](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230207165012007.png)

接下来就是根据这个计算出具体的光标坐标和高度，高度有点麻烦，比如下图：

![image-20230208102711467](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208102711467.png)

我们首先会觉得应该和文字高度一致，但是如果是标点符号呢，完全一致又太短了，那就需要一个最小高度，但是这个最小高度又是多少呢？

```js
// 获取光标位置信息
getCursorInfo(positionIndex) {
    let position = this.positionList[positionIndex]
    let { fontSize } = this.options
    // 光标高度在字号的基础上再高一点
    let height = position.size || fontSize
    let plusHeight = height / 2
    let actHeight = height + plusHeight
    // 元素所在行
    let row = this.rows[position.rowIndex]
    return {
        x: position.rect.rightTop[0],
        y:
            position.rect.rightTop[1] +
            row.height -
            (row.height - row.originHeight) / 2 -
            actHeight +
            (actHeight - Math.max(height, position.info.height)) / 2,
        height: actHeight
    }
}
```

我们没有把文字的实际高度作为光标高度，而是直接使用文字的字号，另外你仔细观察各种编辑器都可以发现光标高度是会略高于文字高度的，所以我们还额外增加了高度的`1/2`，光标位置的`y`坐标计算有点复杂，可以对着下面的图进行理解：

![image-20230208162553425](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208162553425.png)

我们先用`canvas`绘制线段的方式来测试一下：

![2023-02-08-10-55-39](E:\图片\2023-02-08-10-55-39.gif)

当然目前考虑到的是常规情况，还有两种特殊情况：

1.页面为空、或者页面不为空，但是点击的是第一个元素的前半部分

这类情况的共同点是计算出来的`positionIndex = -1`，目前我们还没有处理这个情况：

```js
getCursorInfo(positionIndex) {
    let position = this.positionList[positionIndex]
    let { fontSize, pagePadding, lineHeight } = this.options
    let height = (position ? position.size : null) || fontSize
    let plusHeight = height / 2
    let actHeight = height + plusHeight
    if (!position) {
      // 当前光标位置处没有元素
      let next = this.positionList[positionIndex + 1]
      if (next) {
        // 存在下一个元素
        let nextCursorInfo = this.getCursorInfo(positionIndex + 1)
        return {
          x: pagePadding[3],
          y: nextCursorInfo.y,
          height: nextCursorInfo.height
        }
      } else {
        // 不存在下一个元素，即文档为空
        return {
          x: pagePadding[3],
          y: pagePadding[0] + (height * lineHeight - actHeight) / 2,
          height: actHeight
        }
      }
    }
    // ...
}
```

当当前光标处没有元素时，先判断是否存在下一个元素，存在的话就使用下一个元素的光标的`y`和`height`信息，避免出现下面这种情况：

![image-20230208165631981](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208165631981.png)

如果没有下一个元素，那么代表文档为空，默认返回页面文档内容的起始坐标。

以上虽然考虑了行高，但是实际上和编辑后还是会存在偏差。

2.点击的是一行的第一个字符的前半部分

当我们点击的是一行第一个字符的前半部分，目前显示会有点问题：

![image-20230208170020599](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208170020599.png)

和后一个字符重叠了，这是因为我们计算的问题，前面的计算行数据的逻辑没有区分换行符，所以计算出来换行符也存在宽度，所以可以修改前面的计算逻辑，也可以直接在`getCursorInfo`方法判断：

```js
getCursorInfo(positionIndex) {
    // ...
    // 是否是换行符
    let isNewlineCharacter = position.value === '\n'
    return {
        x: isNewlineCharacter ? position.rect.leftTop[0] : position.rect.rightTop[0],
        // ...
    }
}
```



## 渲染光标

光标可以使用`canvas`渲染，也可以使用`DOM`元素渲染，简单起见，我们使用`DOM`元素来渲染，光标元素也是添加到容器元素内，容器元素设置为相对定位，光标元素设置为绝对定位：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.cursorEl = null // 光标元素
    }

    // 设置光标
    setCursor(left, top, height) {
        if (!this.cursorEl) {
            this.cursorEl = document.createElement('div')
            this.cursorEl.style.position = 'absolute'
            this.cursorEl.style.width = '1px'
            this.cursorEl.style.backgroundColor = '#000'
            this.container.appendChild(this.cursorEl)
        }
        this.cursorEl.style.left = left + 'px'
        this.cursorEl.style.top = top + 'px'
        this.cursorEl.style.height = height + 'px'
    }
}
```

前面我们计算出的光标位置是相对于当前页面`canvas`的，还需要转换成相对于容器元素的：

```js
class CanvasEditor {
    // 将相对于页面canvas的坐标转换成相对于容器元素的
    canvasToContainer(x, y, canvas) {
        return {
            x: x + canvas.offsetLeft,
            y: y + canvas.offsetTop
        }
    }
}
```

有了前面这么多铺垫，我们的光标就可以出来了：

```js
class CanvasEditor {
    onMousedown(e, pageIndex) {
        // 鼠标按下位置相对于页面canvas的坐标
        let { x, y } = this.windowToCanvas(e, this.pageCanvasList[pageIndex])
        // 计算该坐标对应的元素索引
        let positionIndex = this.getPositionByPos(x, y, pageIndex)
        // 根据元素索引计算出光标位置和高度信息
        let cursorInfo = this.getCursorInfo(positionIndex)
        // 渲染光标
        let cursorPos = this.canvasToContainer(
            cursorInfo.x,
            cursorInfo.y,
            this.pageCanvasList[pageIndex]
        )
        this.setCursor(cursorPos.x, cursorPos.y, cursorInfo.height)
    }
}
```

![2023-02-08-11-18-40](E:\图片\2023-02-08-11-18-40.gif)

当然，现在光标还是不会闪烁的，这个简单：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.cursorTimer = null// 光标元素闪烁的定时器
    }

    setCursor(left, top, height) {
        clearTimeout(this.cursorTimer)
        // ...
        this.cursorEl.style.opacity = 1
        this.blinkCursor(0)
    }

    // 光标闪烁
    blinkCursor(opacity) {
        this.cursorTimer = setTimeout(() => {
            this.cursorEl.style.opacity = opacity
            this.blinkCursor(opacity === 0 ? 1 : 0)
        }, 600)
    }
}
```

通过一个定时器来切换光标元素的透明度，达到闪烁的效果。



# 编辑效果

终于到了万众瞩目的编辑效果了，编辑大致就是删除、换行、输入，所以监听一下`keydown`事件，区分按下的是什么键，然后对数据做对应的处理，最后重新渲染就可以了，当然，光标的位置也需要更新，不过继续之前我们需要做另一件事：聚焦。

## 聚焦

如果我们用的是`input`、`textarea`标签，或者是`DOM`元素的`contentedit`属性实现编辑，不用考虑这个问题，但是我们用的是`canvas`标签，无法聚焦，不聚焦就无法输入，不然你试试切换输入语言都不生效，所以当我们点击页面，渲染光标的同时，也需要手动聚焦，创建一个隐藏的`textarea`标签用于聚焦和失焦：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.textareaEl = null // 文本输入框元素
    }

    // 聚焦
    focus() {
        if (!this.textareaEl) {
            this.textareaEl = document.createElement('textarea')
            this.textareaEl.style.position = 'fixed'
            this.textareaEl.style.left = '-99999px'
            document.body.appendChild(this.textareaEl)
        }
        this.textareaEl.focus()
    }

    // 失焦
    blur() {
        if (!this.textareaEl) {
            return
        }
        this.textareaEl.blur()
    }
}
```

然后在渲染光标的同时调用聚焦方法：

```js
setCursor(left, top, height) {
    // ...
    setTimeout(() => {
      this.focus()
    }, 0)
}
```

为什么要在`setTimeout0`后聚焦呢，因为`setCursor`方法是在`mousedown`方法里调用的，这时候聚焦，`mouseup`事件触发后又失焦了，所以延迟一点。

## 输入

输入我们选择监听`textarea`的`input`事件，这么做的好处是不用自己区分是否是按下了可输入按键，可以直接从事件对象的`data`属性获取到输入的字符，如果按下的不是输入按键，那么`data`的值为`null`，但是有个小问题，如果我们输入中文时，即使是在打拼音的阶段也会触发，这是没有必要的，解决方法是可以监听`compositionupdate`和`compositionend`事件，当我们输入拼音阶段会触发`compositionstart`，然后每打一个拼音字母，触发`compositionupdate`，最后将输入好的中文填入时触发`compositionend`，我们通过一个标志位来记录当前状态即可。

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.isCompositing = false // 是否正在输入拼音
    }

    focus() {
        if (!this.textareaEl) {
            this.textareaEl.addEventListener('input', this.onInput.bind(this))
            this.textareaEl.addEventListener('compositionstart', () => {
                this.isCompositing = true
            })
            this.textareaEl.addEventListener('compositionend', () => {
                this.isCompositing = false
            })
        }
    }

    // 输入事件
    onInput(e) {
        setTimeout(() => {
            let data = e.data
            if (!data || this.isCompositing) {
                return
            }
        }, 0)
    }
}
```

可以看到在输入方法里我们又使用了`setTimeout0`，这又是为啥呢，其实是因为`compositionend`事件触发的比`input`事件晚，不延迟就无法获取到输入的中文。

获取到了输入的字符就可以更新数据了，更新显然是在光标位置处更新，所以我们还需要添加一个字段，用来保存光标所在元素位置：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.cursorPositionIndex = -1// 当前光标所在元素索引
    }

    onMousedown(e, pageIndex) {
        // ...
        let positionIndex = this.getPositionByPos(x, y, pageIndex)
        this.cursorPositionIndex = positionIndex
        // ...
    }
}
```

那么插入输入的字符就很简单了：

```js
class CanvasEditor {
    onInput(e) {
        // ...
        // 插入字符
        let arr = data.split('')
        let length = arr.length
        this.data.splice(
            this.cursorPositionIndex + 1,
            0,
            ...arr.map(item => {
                return {
                    value: item
                }
            })
        )
        // 重新渲染
        this.render()
        // 更新光标
        this.cursorPositionIndex += length
        this.computeAndRenderCursor(
            this.cursorPositionIndex,
            this.positionList[this.cursorPositionIndex].pageIndex
        )
    }
}
```

`computeAndRenderCursor`方法就是把前面的计算光标位置、坐标转换、设置光标的逻辑提取了一下，方便复用。

![2023-02-08-15-00-37](E:\图片\2023-02-08-15-00-37.gif)

可以输入了，但是有个小问题，比如我们是在有样式的文字中间输入，那么预期新输入的文字也是带同样样式的，但是现在显然是没有的：

![image-20230208150305425](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230208150305425.png)

解决方法很简单，插入新元素时复用当前元素的样式数据：

```js
onInput(e) {
    // ...
    let cur = this.positionList[this.cursorPositionIndex]
    this.data.splice(
        this.cursorPositionIndex + 1,
        0,
        ...arr.map(item => {
            return {
                ...(cur || {}),// ++
                value: item
            }
        })
    )
}
```

## 删除

删除很简单，判断按下的是否是删除键，是的话从数据中删除光标当前元素即可：

```js
class CanvasEditor {
    focus() {
        // ...
        this.textareaEl.addEventListener('keydown', this.onKeydown.bind(this))
        // ...
    }

    // 按键事件
    onKeydown(e) {
        if (e.keyCode === 8) {
            this.delete()
        }
    }

    // 删除
    delete() {
        if (this.cursorPositionIndex < 0) {
            return
        }
        // 删除数据
        this.data.splice(this.cursorPositionIndex, 1)
        // 重新渲染
        this.render()
        // 更新光标
        this.cursorPositionIndex--
        let position = this.positionList[this.cursorPositionIndex]
        this.computeAndRenderCursor(
            this.cursorPositionIndex,
            position ? position.pageIndex : 0
        )
    }
}
```

![2023-02-09-09-26-41](E:\图片\2023-02-09-09-26-41.gif)

## 换行

换行也很简单，监听到按下回车键就向光标处插入一个换行符，然后重新渲染更新光标：

```js
class CanvasEditor {
    // 按键事件
    onKeydown(e) {
        if (e.keyCode === 8) {
            this.delete()
        } else if (e.keyCode === 13) {
            this.newLine()
        }
    }

    // 换行
    newLine() {
        this.data.splice(this.cursorPositionIndex + 1, 0, {
            value: '\n'
        })
        this.render()
        this.cursorPositionIndex++
        let position = this.positionList[this.cursorPositionIndex]
        this.computeAndRenderCursor(this.cursorPositionIndex, position.pageIndex)
    }
}
```

![2023-02-09-09-34-05](E:\图片\2023-02-09-09-34-05.gif)

其实我按了很多次回车键，但是似乎只生效了一次，这是为啥呢，其实我们插入是没有问题的，问题出在一行中如果只有换行符那么这行高度为`0`，所以渲染出来没有效果，修改一下计算行数据的`computeRows`方法：

```js
computeRows() {
    let { fontSize } = this.options
    // ...
    this.data.forEach(item => {
        // ...
        // 尺寸信息
        let info = {
            width: 0,
            height: 0,
            ascent: 0,
            descent: 0
        }
        if (value === '\n') {
            // 如果是换行符，那么宽度为0，高度为字号
            info.height = fontSize
        } else {
            // 其他字符
            ctx.font = font
            let { width, actualBoundingBoxAscent, actualBoundingBoxDescent } =
                ctx.measureText(value)
            info.width = width
            info.height = actualBoundingBoxAscent + actualBoundingBoxDescent
            info.ascent = actualBoundingBoxAscent
            info.descent = actualBoundingBoxDescent
        }
        // ...
    })
    // ...
}
```

如果是换行符，那么高度默认为字号，否则还是走之前的逻辑，同时我们把换行符存在宽度的问题也一并修复了。

![2023-02-09-09-43-14](E:\图片\2023-02-09-09-43-14.gif)

到这里，基本的编辑就完成了，接下来实现另一个重要的功能：选区。



# 选区

选区其实就是我们鼠标通过拖拽选中文档的一部分，就是一段区间，可以通过一个数组来保存：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.range = [] // 当前选区，第一个元素代表选区开始元素位置，第二个元素代表选区结束元素位置
        // ...
    }
}
```

如果要支持多段选区的话可以使用二维数组。

然后渲染的时候判断是否存在选区，存在的话再判断当前绘制到的元素是否在选区内，是的话就额外绘制一个矩形作为选区。

## 计算选区

选择选区肯定是在鼠标按下的时候进行的，所以需要添加一个标志代表鼠标当前是否处于按下状态，然后监听鼠标移动事件和松开事件，这两个事件我们绑定在`body`上，因为鼠标是可以移出页面的。鼠标按下时需要记录当前所在的元素索引：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.isMousedown = false // 鼠标是否按下
        document.body.addEventListener('mousemove', this.onMousemove.bind(this))
        document.body.addEventListener('mouseup', this.onMouseup.bind(this))
        // ...
    }

    // 鼠标按下事件
    onMousedown(e, pageIndex) {
        // ...
        this.isMousedown = true
        let positionIndex = this.getPositionByPos(x, y, pageIndex)
        this.range[0] = positionIndex
        // ...
    }
    
    // 鼠标移动事件
    onMousemove(e) {
        if (!this.isMousedown) {
            return
        }
    }

    // 鼠标松开事件
    onMouseup() {
        this.isMousedown = false
    }
}
```

因为前面我们写的`windowToCanvas`方法需要知道是在哪个页面，所以还得新增一个方法用于判断一个坐标在哪个页面：

```js
class CanvasEditor {
    // 获取一个坐标在哪个页面
    getPosInPageIndex(x, y) {
        let { left, top, right, bottom } = this.container.getBoundingClientRect()
        // 不在容器范围内
        if (x < left || x > right || y < top || y > bottom) {
            return -1
        }
        let { pageHeight, pageMargin } = this.options
        let scrollTop = this.container.scrollTop
        // 鼠标的y坐标相对于容器顶部的距离
        let totalTop = y - top + scrollTop
        for (let i = 0; i < this.pageCanvasList.length; i++) {
            let pageStartTop = i * (pageHeight + pageMargin)
            let pageEndTop = pageStartTop + pageHeight
            if (totalTop >= pageStartTop && totalTop <= pageEndTop) {
                return i
            }
        }
        return -1
    }
}
```

然后当鼠标移动时实时判断当前移动到哪个元素，并且重新渲染达到选区的选择效果：

```js
onMousemove(e) {
    if (!this.isMousedown) {
        return
    }
    // 鼠标当前所在页面
    let pageIndex = this.getPosInPageIndex(e.clientX, e.clientY)
    if (pageIndex === -1) {
        return
    }
    // 鼠标位置相对于页面canvas的坐标
    let { x, y } = this.windowToCanvas(e, this.pageCanvasList[pageIndex])
    // 鼠标位置对应的元素索引
    let positionIndex = this.getPositionByPos(x, y, pageIndex)
    if (positionIndex !== -1) {
        this.range[1] = positionIndex
        this.render()
    }
}
```

`mousemove`事件触发频率很高，所以为了性能考虑可以通过节流的方式减少触发次数。

计算鼠标移动到哪个元素和光标的计算是一样的。

## 渲染选区

选区其实就是一个矩形区域，和元素背景没什么区别，所以可以在渲染的时候判断是否存在选区，是的话给在选区中的元素绘制选区的样式即可：

```js
class CanvasEditor {
    constructor(container, data, options = {}) {
        // ...
        this.options = Object.assign(
            {
                // ...
                rangeColor: '#bbdfff', // 选区颜色
                rangeOpacity: 0.6 // 选区透明度
            }
        )
        // ...
    }

    renderRow(ctx, renderHeight, row, pageIndex, rowIndex) {
        let { rangeColor, rangeOpacity } = this.options
        // ...
        row.elementList.forEach(item => {
            // ...
            // 渲染选区
            if (this.range.length === 2 && this.range[0] !== this.range[1]) {
                let range = this.getRange()
                let positionIndex = this.positionList.length - 1
                if (positionIndex >= range[0] && positionIndex <= range[1]) {
                    ctx.save()
                    ctx.beginPath()
                    ctx.globalAlpha = rangeOpacity
                    ctx.fillStyle = rangeColor
                    ctx.fillRect(renderWidth, renderHeight, item.info.width, row.height)
                    ctx.restore()
                }
            }
            // ...
        })
        // ...
    }
}
```

调用了`getRange`方法获取选区，为什么还要通过方法来获取呢，不就是`this.range`吗，非也，鼠标按下的位置和鼠标实时的位置是存在前后关系的，位置不一样，实际的选区范围也不一样。

如下图，如果鼠标实时位置在鼠标按下位置的后面，那么按下位置的元素实际上是不包含在选区内的：

![image-20230209152836289](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230209152836289.png)

如下图，如果鼠标实时位置在鼠标按下位置的前面，那么鼠标实时位置的元素实际上是不需要包含在选区内的：

![image-20230209152856618](C:\Users\wanglin25\AppData\Roaming\Typora\typora-user-images\image-20230209152856618.png)

所以我们需要进行一下判断：

```js
class CanvasEditor {
    // 获取选区
    getRange() {
        if (this.range.length < 2) {
            return []
        }
        if (this.range[1] > this.range[0]) {
            // 鼠标结束元素在开始元素后面，那么排除开始元素
            return [this.range[0] + 1, this.range[1]]
        } else if (this.range[1] < this.range[0]) {
            // 鼠标结束元素在开始元素前面，那么排除结束元素
            return [this.range[1] + 1, this.range[0]]
        } else {
            return []
        }
    }
}
```

效果如下：

![2023-02-09-14-35-22](E:\图片\2023-02-09-14-35-22.gif)

## 处理和光标的冲突

到这里结束了吗，没有，目前选择选区时光标还是在的，并且单击后选区也没有消失。接下来解决一下这两个问题。

解决第一个问题很简单，在选择选区的时候可以判断一下当前选区范围是否大于`0`，是的话就隐藏光标：

```js
class CanvasEditor {
    onMousemove(e) {
        // ...
        let positionIndex = this.getPositionByPos(x, y, pageIndex)
        if (positionIndex !== -1) {
            this.rangeEndPositionIndex = positionIndex
            if (Math.abs(this.range[1] - this.range[0]) > 0) {
                // 选区大于1，光标就不显示
                this.cursorPositionIndex = -1
                this.hideCursor()
            }
            // ...
        }
    }

    // 隐藏光标
    hideCursor() {
        clearTimeout(this.cursorTimer)
        this.cursorEl.style.display = 'none'
    }
}
```

第二个问题可以在设置光标时直接清除选区：

```js
class CanvasEditor {
    // 清除选区
    clearRange() {
        if (this.range.length > 0) {
            this.range = []
            this.render()
        }
    }

    setCursor(left, top, height) {
        this.clearRange()
        // ...
    }
}
```

![2023-02-09-14-44-23](E:\图片\2023-02-09-14-44-23.gif)

## 编辑选区内容

目前选区只是绘制出来了，但是没有实际用处，不能删除，也不能替换，先增加一下删除选区的逻辑：

```js
delete() {
    if (this.cursorPositionIndex < 0) {
        let range = this.getRange()
        if (range.length > 0) {
            // 存在选区，删除选区内容
            let length = range[1] - range[0] + 1
            this.data.splice(range[0], length)
            this.cursorPositionIndex = range[0] - 1
        } else {
            return
        }
    } else {
        // 删除数据
        this.data.splice(this.cursorPositionIndex, 1)
        // 重新渲染
        this.render()
        // 更新光标
        this.cursorPositionIndex--
    }
    let position = this.positionList[this.cursorPositionIndex]
    this.computeAndRenderCursor(
        this.cursorPositionIndex,
        position ? position.pageIndex : 0
    )
}
```

很简单，如果存在选区就删除选区的数据，然后重新渲染光标。

接下来是替换，如果存在选区时我们输入文字，输入的文字会替换掉选区的文字，实现上我们可以直接删除：

```js
onInput(e) {
    // ...
    let range = this.getRange()
    if (range.length > 0) {
        // 存在选区，则替换选区的内容
        this.delete()
    }
    let cur = this.positionList[this.cursorPositionIndex]
    // 原来的输入逻辑
}
```

输入时判断是否存在选区，存在的话直接调用删除方法删除选区，删除后的光标位置也是正确的，所以再进行原本的输入不会有任何问题。



# 总结

到这里我们实现了一个类似`Word`的富文本编辑器，支持文字的编辑，支持有限的文字样式，支持光标，支持选区，当然，这是最基本最基本的功能，随便想想就知道还有很多功能没实现，比如复制、粘贴、方向键切换光标位置、拖拽选区到其他位置、前进后退等，以及支持图片、表格、链接、代码块等文本之外的元素，所以想要实现一个完整可用的富文本是非常复杂的，要考虑的问题非常多。

本文完整源码：[https://github.com/wanglin2/canvas-editor-demo](https://github.com/wanglin2/canvas-editor-demo)。

有兴趣了解更多的可以参考这个项目：[https://github.com/Hufe921/canvas-editor](https://github.com/Hufe921/canvas-editor)。