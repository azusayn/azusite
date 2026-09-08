---
title: CSS 学习笔记
date: 2023/9/9 11:17:00
categories:
- Frontend
tags:
- web
- css
---

### 0. 参考网站

- [MDN Docs](https://developer.mozilla.org/zh-CN/docs/Learn/CSS) 
- [W3C](https://www.w3school.com.cn/css/index.asp)

### 1. 问题与笔记

- 当使用百分比设置元素的长宽时，一般元素默认的参考是父元素对应的长宽。而像 `margin` 和 `padding` 则 **只会根据父元素的宽度** 进行计算

- 当设置 `box-sizing: border-box` 的时候，会使浏览器将元素的边距边框考虑在内，以防止超出父元素。而默认情况下使用的 `box-sizing` 是 `content-box` ，即只考虑元素的内容部分进行布局。

- 在 **CSS** 中进行对齐

  - 一般的布局模式下对齐

  	- 用 `margin: auto` 会自动将块元素左右的 `margin` 设置为相同的值，达到居中的效果。

  	- 如果单纯左右对齐，可以将元素设置为 `position` 或者 `float` 定位，并设置位置数值。

  	- 加入是元素内的文本对齐可以对包含文本的元素使用 `text-align` 属性

  		```css
  			text-align: 
  					center 		# 靠中间对齐
  					left, right # 靠左, 右对齐
  					start, end  # 根据文本的方向确定 start 为 left 或者 right
  					justify		# 文本向两边靠拢
  		```

  - 在 `flex` 布局下 `aligin-items` 和 `justify-content` **分别控制 <u>交叉轴</u> 和 <u>主轴</u>** 方向的对齐。（这意味着如果改变`flex-direction` 的值会影响这两个字段的行为）。同时要注意 **CSS** 中有着 **items** 和 **content** 结尾的另外两个方法，结论上而言，**items** 结尾方法控制 **flex** 盒子中各个行单独的对齐行为，而 **content** 方法会把 **flex** 盒子内各个行看成一个整体进行对齐。 

  - 除了上述，还有 `place-items` 可以同时控制主轴和交叉轴的对齐模式

- 选择器的用法：**CSS** 中的选择器用于定位 **HTML** 中的不同标签或属性以便添加样式；选择器主要包括 **类型**、**类**、**ID**、**属性 ** 和 **关系** 五种选择器 （但其实还有一个全局选择器）。另外逗号代表对多个元素应用同一个样式。

  - **类型**、**类**、**ID** 选择器

  	```css
  	    h1, span, div{} # 类型选择器就是使用标签作为目标进行匹配
  	    .box{} 			# 类选择器根据class中的值进行匹配，这里会匹配class带有box的标签
  	    #box{}			# 类似于类选择器, id选择器根据标签的id中的值进行匹配
  	```

  - **属性** 选择器

  	```css
  		li[class]		# 匹配带有class属性的li标签
  		li[class="a"]	# 带有class属性且正好是"a"
  		li[class~="a"]  # 带有class属性且包含"a"
  		li[class^="a"]  # 属性值开头为"a"
  		li[class$="a"]  # 属性值结尾为"a"
  		li[class*="a"]  # 属性值出现了"a"
  		li[class="a" i] # 添加i属性可以设置大小写不敏感
  	```

  - **关系** 选择器

  	```css
  		div p 	# 用空格符代表p是div的后代 (不一定是直接后代)
  		div > p # 代表p是div的直接后代
  		div ~ p # div和p属于同一元素的子代
  		div + p # div和p属于同一元素的子代且相邻
  	```

  - **全局** 选择器，可以使用 `*` 符号作为通配符进行匹配

- 伪类和伪元素，更多的伪类和伪元素可以参考 [**MDN** 文档](https://developer.mozilla.org/zh-CN/docs/Learn/CSS/Building_blocks/Selectors/Pseudo-classes_and_pseudo-elements#%E5%8F%82%E8%80%83%E8%8A%82)

  - **伪类**：根据 **MDN** 文档，*伪类是选择器的一种，它用于选择处于特定状态的元素，比如当它们是这一类型的第一个元素时，或者是当鼠标指针悬浮在元素上面的时候。它们表现得会像是你向你的文档的某个部分应用了一个类一样，帮你在你的标记文本中减少多余的类* 。常见的伪类有 `:hover` , `:focus` ，`:last-child` 等

  	```css
  		# 匹配article后的p, 且为第一个子代
  	    article p:first-child{ 
  	    	color: brown;      
  		}
  	```

  - **伪元素**：*伪元素以类似方式表现，不过表现得是像你往标记文本中加入全新的 HTML 元素一样，而不是向现有的元素上应用类。伪元素开头为双冒号* 。可以通过 `::before` 和 `::after` 这样的伪元素在标签中添加内容

  	```css
  		# 在box类的元素之前添加内容
  	    .box::before {
  	    	content: "This should show before the other content. ";
  		} 
  	```


- 在选择器中，元素之间的空格有特殊含义

	```css
		article p :last-child # 代表p的后代且是最后一个子代
		article p:last-child  # 代表p, 且是最后一个子代
	```

- **OOCSS** 和 **BEM** ：都是 **CSS** 的组织方式，具有工程意义

  - **OOCSS**，面向对象 **CSS** 。对具有重复定义的 CSS 代码进行提炼，制造出继承关系。([参考来源](https://juejin.cn/post/7021067874139635726))

  	```css
  		# 应用OOCSS前
  		.box-1 {
  	      border: 1px solid #ccc;
  	      width: 200px;
  	      height: 200px;
  	      border-radius: 10px;
  	    }
  	
  	    .box-2 {
  	      border: 1px solid #ccc;
  	      width: 120px;
  	      height: 120px
  	      border-radius: 10px;
  	    }
  	
  		# 应用OOCSS之后
  		.box-border{
  	      border: 1px solid #CCC;
  	      border-radius: 10px;
  	    }
  	
  	    .box-1 {
  	      width: 200px;
  	      height: 200px;
  	    }
  	
  	    .box-2 {
  	      width: 120px;
  	      height: 120px;
  	    }
  	```

  - **BEM** （Block，Element，Modefier）：**BEM** 是一个分层系统，它把我们的网站分为三层：**块层、元素层、修饰符层**。要体现到代码上，我们需要遵循三个原则（[参考来源](https://juejin.cn/post/7021461539236347940))：

  	- 使用 `__` 两个下划线将块名称与元素名称分开
  	- 使用 `--` 两个破折号分隔元素名称及其修饰符
  	- **一切样式都是一个类，不能嵌套**。

- 描述颜色的三种方式

	- RGB 
	- HSL：Hue (色相)，Saturation，Lightness
	- HEX：即用 **#** 开头的十六进制字符表示颜色