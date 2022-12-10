---
title: grid网格布局
date: 2012-11-02 17:38:02
categories:
  - 前端
tags:
  - vue
  - ts
author: leellun
---

## 父容器属性

### grid 所有网格容器的简写属性 

```
.grid-container {
  display: grid;
  grid: 160px / auto auto auto;
}
```

 ### grid-template-areas  与子元素 grid-area 结合使用

```
.item1 {
  grid-area: myArea;
}
.grid-container {
  grid-template-areas: 'myArea myArea myArea myArea myArea';
}
```

### grid-template-rows 指定行大小（高度） 

```
.grid-container {
  display: grid;
  grid-template-rows: 100px 300px;
}
```

### grid-template-columns 设置网格中列的默认大小 

```
.grid-container {
  display: grid;
  grid-template-columns: auto auto auto auto;
}
```

### grid-auto-rows 设置网格中行的默认大小 

```
.grid-container {
  display: grid;
  grid-auto-rows: 150px;
}
```

### grid-auto-flow 逐列插入网格元素 

```
.grid-container {
  display: grid;
  grid-auto-flow: column;
}
```

### grid-row-gap 设置行的间隙 

```
.grid-container {
  grid-row-gap: 50px;
}
```

### grid-column-gap 设置列的间隙 

```
.grid-container {
  display: grid;
  grid-column-gap: 50px;
}
```

###  grid-gap 设置行与列的间隙 

```
.grid-container {
  grid-gap: 50px;
}
```



## 子元素属性

### grid-area 与父元素 grid-template-areas结合使用 

```
.item1 { grid-area: header; }
.item1 { grid-area: 1 / 1 / 2 / 2; }
```

### grid-column 定义网格元素列的开始和结束位置 

```
.item1 {
  grid-column: 1 / 5;
}
```



| [column-gap](https://www.runoob.com/cssref/css3-pr-column-gap.html) | 指定列之间的间隙                                 |
| ---------------------------------------- | ---------------------------------------- |
| [gap](https://www.runoob.com/cssref/css3-pr-gap.html) | *row-gap* 和 *column-gap* 的简写属性           |
| [grid](https://www.runoob.com/cssref/css-pr-grid.html) | *grid-template-rows, grid-template-columns, grid-template-areas, grid-auto-rows, grid-auto-columns*, 以及 *grid-auto-flow* 的简写属性 |
| [grid-area](https://www.runoob.com/cssref/pr-grid-area.html) | 指定网格元素的名称，或者也可以是 *grid-row-start*, *grid-column-start*, *grid-row-end*, 和 *grid-column-end* 的简写属性 |
| [grid-auto-columns](https://www.runoob.com/cssref/pr-grid-auto-columns.html) | 指的默认的列尺寸                                 |
| [grid-auto-flow](https://www.runoob.com/cssref/pr-grid-auto-flow.html) | 指定自动布局算法怎样运作，精确指定在网格中被自动布局的元素怎样排列。       |
| [grid-auto-rows](https://www.runoob.com/cssref/pr-grid-auto-rows.html) | 指的默认的行尺寸                                 |
| [grid-column](https://www.runoob.com/cssref/pr-grid-column.html) | *grid-column-start* 和 *grid-column-end* 的简写属性 |
| [grid-column-end](https://www.runoob.com/cssref/pr-grid-column-end.html) | 指定网格元素列的结束位置                             |
| [grid-column-gap](https://www.runoob.com/cssref/pr-grid-column-gap.html) | 指定网格元素的间距大小                              |
| [grid-column-start](https://www.runoob.com/cssref/pr-grid-column-start.html) | 指定网格元素列的开始位置                             |
| [grid-gap](https://www.runoob.com/cssref/pr-grid-gap.html) | *grid-row-gap* 和 *grid-column-gap* 的简写属性 |
| [grid-row](https://www.runoob.com/cssref/pr-grid-row.html) | *grid-row-start* 和 *grid-row-end* 的简写属性  |
| [grid-row-end](https://www.runoob.com/cssref/pr-grid-row-end.html) | 指定网格元素行的结束位置                             |
| [grid-row-gap](https://www.runoob.com/cssref/pr-grid-row-gap.html) | 指定网格元素的行间距                               |
| [grid-row-start](https://www.runoob.com/cssref/pr-grid-row-start.html) | 指定网格元素行的开始位置                             |
| [grid-template](https://www.runoob.com/cssref/pr-grid-template.html) | *grid-template-rows*, *grid-template-columns* 和 *grid-areas* 的简写属性 |
| [grid-template-areas](https://www.runoob.com/cssref/pr-grid-template-areas.html) | 指定如何显示行和列，使用命名的网格元素                      |
| [grid-template-columns](https://www.runoob.com/cssref/pr-grid-template-columns.html) | 指定列的大小，以及网格布局中设置列的数量                     |
| [grid-template-rows](https://www.runoob.com/cssref/pr-grid-template-rows.html) | 指定网格布局中行的大小                              |
| [row-gap](https://www.runoob.com/cssref/css3-pr-row-gap.html) | 指定两个行之间的间距                               |

## grid布局属性

### 父容器属性

| 属性                    | 说明                                       |
| --------------------- | ---------------------------------------- |
| grid-template-rows    | 用于设置网格布局中的行数及高度。                         |
| grid-template-columns | 用于设置网格布局中的列数及宽度。                         |
| grid-template-areas   | 属性用于设置网格布局                               |
| grid-template         | 用于网格布局，设置网格中行、列与分区 。 以下属性的简写形式： grid-template-rows、grid-template-columns grid-template-areas |
| grid-row-gap          | 属性是用来设置网格行之间的间隙。                         |
| grid-gap              | grid-gap 属性是用来设置网格行与列之间的间隙。              |
| grid-column-gap       | 列之间的间隙。                                  |
| grid-auto-flow        | 指定自动布局算法怎样运作，精确指定在网格中被自动布局的元素怎样排列。       |
| grid-auto-rows        | 用于设置网格容器中行的默认大小。                         |
| grid-auto-columns     | 用于设置网格容器中行的默认大小。                         |
| row-gap               | 指定网格行之间的间隔                               |
| gap                   | 设置网格行与列之间的间隙                             |
| grid                  | CSS 所有网格容器的简写属性，                         |
| column-gap            | 指定网格行之间的间隔                               |



### 子节点属性

| 属性                | 说明                                       |
| ----------------- | ---------------------------------------- |
| grid-row          | 属性定义了网格元素行的开始和结束位置。  grid-row-start 和 grid-row-end 属性的简写属性。 |
| grid-area         | 可以对网格元素进行命名 。与grid-template-areas配合使用    |
| grid-row-start    | 指定哪一行开始显示网格元素。                           |
| grid-row-end      | 属性指定哪一行停止显示网格元素，或设置跨越多少行。                |
| grid-column-start | 定义了网格元素从哪一列开始。                           |
| grid-column-end   | 定义了网格元素跨越多少列，或者在哪一列结束。                   |
| grid-column       | grid-column 是 grid-column-start 和 grid-column-end 属性的简写属性。 |



## 表格单文本属性


| 名称                                       | 意思                                   |
| ---------------------------------------- | ------------------------------------ |
| columns                                  | 是 column-width 与 column-count 的简写属性。 |
| column-width                             | 列宽                                   |
| column-count                             | 列数                                   |
| column-gap                               | 在div元素的文本分成三列，并指定一个30像素的列之间的差距       |
| [Column-rule](https://www.runoob.com/try/try.php?filename=trycss3_column-rule) | 指定列之间的规则：宽度，样式和颜色。                   |
| column-span                              | 指定某个元素应该跨越多少列。                       |
| column-rule-width                        | 指定列之间的宽度规则。                          |
| column-rule-style                        | 指定列之间的样式规则                           |
| column-rule-color                        | 指定列之间的颜色规则                           |
| column-fill                              | 指定如何填充列                              |

# flexbox教程

https://www.runoob.com/w3cnote/flex-grammar.html

