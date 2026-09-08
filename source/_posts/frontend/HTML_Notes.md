---
title: HTML 学习笔记
date: 2023/9/9 11:09:00
categories:
- Frontend
tags:
- HTML
- web
---

### 1. 问题与笔记

- 关于网页元素嵌入:
	- **iframe** 中的 `allow-script` 和 `sandbox` ：前者允许iframe中的脚本执行，后者能进行 **iframe** 中部分操作的限制
	- **embed** 和 **object** 标签：二者作用相似，用于嵌入多媒体元素。但是 **embed** 受制于浏览器插件，所以后者的兼容性更好
- 创建一个表格，需要使用到 `<th>` 设定表头，`<tr>` 换行，`<td>` 设置表格内部数据；同时可以使用 `col-span` 等属性设置表格占据的大小；也可以用 `col-group` 和 `col` 组合成的标签设置表格中元素的颜色，`<col-group>` 标签放在最开始的位置；同时可以使用 `<thead>`, `<tbody>` , `<tfoot>` 等标签将表格格式化以便于代码维护。

