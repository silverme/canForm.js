<img src="logo.png">

>canForm.js 是一款简洁、灵活、高扩展性的表单验证组件。
> 
>它基于原生JS，融合自定义标签与自定义属性，通过强大的链式规则与丰富的回调机制，专注于以一种更优雅的声明式规范，使其能适用于更多的业务场景，尽可能地做到最大程度的“通用化”。
>
>欢迎各位同仁、前辈批评指正。 

# 引言

## 一. canForm.js是怎么处理表单验证的？

<br />

### 1. 表单再分类
canForm将一个表单的组成部分分为如下5大类：

<br />

#### 1） scope（作用域）
 
作用域是表单运作的基础，每个作用域包含若干表单子元素，也需要包含提交的按钮。

canForm在建立关系结构时会从各作用域入手，您也可以将作用域理解为 `<form>` ，但考虑到更多可能的业务场景（如不通过 `<form>`，以异步JSON格式提交等），canForm定义了 `<c-scope>` 作为作用域标识，详情请见后面 `<c-scope>` 的相关介绍。

<br />

#### 2） 可接受键盘输入或文本粘贴的第一类表单元素

这类表单元素包括：
	
- `<input>` 中的 `text` `password` `email` `search` `tel` `url` `datetime`等
- `<textarea>`

表单元素启动检测的条件可以是：

- 获取到键盘输入(keydown).
- 当值发生了改变(change).
- 当表单元素失去焦点(blur).

<br />

#### 3） 浏览器自带输入控件的第二类表单元素

这类表单元素包括：
	
- `<input>` 中的 `range` `date` `color` `file`等（除 `第一类表单元素` 中的 `<input>` 之外的）
- `<select>`

表单元素启动检测的条件可以是：

- 当值发生了改变(change).
- 当表单元素失去焦点(blur).

<br />

#### 4） `<input type="radio|checkbox">` 与模拟表单元素等

这类判定情景可根据业务需要而变得十分复杂，需要单独封装处理。

首先看radio和checkbox，我们可能会碰到如下的问题：

- 选择不同radio，实时显示不同的文字。
- checkbox只能选择3个以内，且选了a选项，b、c选项不能选。
- 选择了某个radio或checkbox，后面带了一个必填的文本框。
- ……

遇到这种情况，我们可能需要单独写控件操作DOM，取值、检测，以及对正误的分别处理，十分麻烦。
 
另外，在面对如 自定义表单元素样式 等需求中，考虑到界面的整体性与兼容性，很多前端工程师会选择采用`<div>` `<ul>`等标签进行模拟 `<select>` 、模拟勾选、模拟radio等一些操作，那么对这些模拟元素的合法性验证也会重复第一种情况的情形。

<br />

canForm定义了 `<c-emu>` 标签和 `c-child` `c-value` 两个属性帮助您更好地解决此类问题，详情请参阅后面的关于 `<c-emu>` 标签的使用介绍。

>在这里，表单元素启动检测的条件根据元素类型不同进行分类：
>
> - 为checkbox或radio时，当值发生了改变(change).
> - 为普通块级元素，当子元素发生了点击(click).

<br />

#### 5） `<input type="submit">` 与 `<c-commit>` 表单提交按钮

不管采用 `AJAX+JSON` 提交还是普通 `form.submit()` 提交，一个表单最终都是要走到提交这一步的。
考虑到使用JSON传值的可能性，canForm定义了 `<c-commit>` 标签。

当提交元素的父级存在 `<form>` 标签时，点击提交按钮，如果验证通过，则会向 `action` 指向的URL提交表单，否则会在验证成功的回调函数中返回一个 `result` ，它包含了本 `<c-scope>` 所有带 `c-name` 属性的表单元素与 `<c-emu>` 的 `key-value` 对，格式如下：
 
     {
         name1: "value",
         name2: "value",
         ......
     }
     
您可以定义一个函数接收这个参数，通过 `JSON.stringfy()` 方法转成字符串，并通过canForm自带封装的`ajax` 函数提交数据。

关于这两者的具体用法，请参阅后面关于 `<c-commit>和<input type="submit">` 的相关介绍。

<br />

**在实际使用时，canForm将 ① 作为 `canScope` 类的实例，将 ② ③ ④ 作为 `canEl` 类的实例，将 ⑤ 作为 `canCommit` 类的实例，与标签的自定义属性紧密结合。**

<br />

### 2. 属性多样化

在canForm的眼中，表单的属性应该至少包括如下：

- 未检测、正确、错误三种状态的提示信息。
- 检测开始、结束、成功、失败的回调函数。
- 四种状态对应的样式改变。
- 元素的检测启动方式：如keydown、change、blur等。

自然地，canForm为如上列举的情况提供了很好的支持。

<br />

### 3. 规则多样化

在canForm的眼中，验证的规则应该包括多组正则表达式或自定义 `function` 的组合，以灵活地满足多变的需求，因此，canForm引入了一套"链式规则"的机制。

在规则属性 `c-rules` 里，您可以这样写：

> - qq-required
> - qq-min5-max10
> - !qq-max9
> - !qq-myFunc()

如上所示，规则或函数以"-"连接，以"!"代表取相反结果，以"()"结尾表示函数。

在初始化时，canForm会找到每个 `<c-scope>` 里每个带 `c-rules` 属性的元素，并将规则转化为数组存储。检测时，非函数的会在规则集中查找，对函数会执行，并传入两个参数。

您可以通过内置的 `setRules` 方法动态添加规则。

    var cf = new canForm();
	cf.setRules({
		one: value,
		two: function(value, el){}
	})

具体的细节请参阅后面的 `setRules` 方法的相关介绍。

<br />

这套机制同样适用于 检测开始、结束、成功、失败这四种回调情况，以检测成功的 `c-yes` 属性举例：

> - green-yesFunc()
> - !red-!yellow-yesFunc()

以上几种写法都是被支持的：函数的写法与 `c-rules` 中的相同，但对非函数样式的字符，canForm会将其理解为 `className` ，`!className` 即为 `removeClass` 操作，不带 `!` 即为 `addClass`。

<br />

**canForm.js为您准备了以上积木，就是为了让您的表单验证尽可能的更简单，让您尽可能少的操作DOM，专注业务逻辑，它已为您磨好了一把锋利的刀，剩下的事情，就是去披荆斩棘啦！**

<br />

## 二. canForm.js的其他“天生骄傲”的特性

- 不依赖jQuery、Zepto等库，使组件少了一份臃肿。
- 支持AMD模块化加载规范。
- 支持IE7+以及其他现代浏览器。
- 多样的内置正则验证，与时俱进，更新至2015版。
- 支持提示文字信息的模板渲染。



<br />

**需要注意的是：**

1. 初始化时会建立属于canForm自己的元素结构，并且所有属性设置都是在初始化时一次性从HTML标签读入， 因此无论您是修改属性，还是动态新增表单元素，都请执行接下来会介绍的内置方法 `updateNodes()` 更新节点结构。

2. 另外为了命名规范、避免冲突，canForm.js只处理以 “c-” 开头的自定义标签或属性。

<br />

# DEMO AREA

- 示例一：基础用户名密码点击验证 c-tip 原生submit提交
- 示例一.5 ：c-rules规则与c-yes c-no链式玩法classname + function
- Blur和chgchk
- 示例二：开启autohide和blur的用户名密码验证
- 示例三：带验证码ajax的手机验证码（自定义验证函数）
- 示例四：textarea字数统计（keyup）、select、checkbox验证
- 示例五：自定义规则
- 示例六：二选一、多选一验证
- 示例6.5：c-emu之模拟下拉 c-emu之radio与checkbox
- 示例七：模拟下拉框验证
- 示例八：至少选择一个（问卷）checkbox

<br />

#一个简单的工程

接下来，我们通过一个简单的工程快速上手canForm。

<br />

## 一、引入

请在HTML中引入 `canForm.min.js` 与 `canForm.base.css`。

`<script src="js/canForm.min.js"></script>`

`<link href="css/canForm.base.css" rel="stylesheet>`

您也可以通过AMD方式加载。

<br />

## 二、初始化

我们可以通过如下语句进行初始化。

	var cf = new canForm();
	cf.init();

HTML代码如下：


	<!DOCTYPE html>
	<html>
	<head>
		<meta charset="utf-8" />
		<title>canForm</title>
		<link href="css/canForm.base.css" rel="stylesheet">
		<script type="text/javascript" src="js/canForm.src.js" ></script>
		<script>
			var cf = new canForm();
			cf.init();
		</script>
	</head>
	
	<body>
		<c-scope>
			<div class="input-wrap">
				<input type="text"
					   c-name="qq_input"
					   c-rules="qq-required" 
					   c-conf="chgchk-autohide"
					   c-yes="!red-myFunc()"
                       c-no="red-noFunc()">
				<c-yes>yes</c-yes>
				<c-no>{{QQ_OK}}</c-no>
				<c-dp>请填写值</c-dp>
			</div>
			
			<div class="submit-wrap">
				<c-commit c-yes="submitValue()"></c-commit>
				<c-no>检测失败</c-no>
			</div>
		</c-scope>

		<script>
			function submitValue(flag,element,check_details,result) {
				alert(result);
			}
		</script>
	</body>

这样就能实现了一个初步的demo。

<br />

> 此demo释义如下：

>当前HTML中存在一个 `<c-scope>` ，在 `<c-scope>` 中存在一个 `<input>` 标签元素，其规则为：

> - 1. 检测qq号结果为真0
> - 2. 必填项结果为真（即不为空）

> 根据 `c-conf` 属性：
> 
> - `chgchk` 开启，说明只有在 `onchange` 事件发生时才会进行检测
> - `autohide` 开启，说明在 `focus` 事件触发后，`c-yes` `c-no` `c-start` `c-end` 属性中涉及到的 `className` 都会被 `remove`。

> 根据 `c-yes` 和 `c-no` 属性：
> 
> - 当检测成功时，名为 `red` 的 `class` 将被移除，之后执行 `myFunc` 函数。
> - 当检测失败时，名为 `red` 的 `class` 将被加入，之后执行 `noFunc` 函数。

> 另外，本demo中存在一个 `<c-commit>` 元素，点击之后作自定义提交的功能，若所属 `scope` 所有子元素验证通过，调用 `submitValue` 方法。由于父级没有 `<form>` 元素，因此不调用默认的 `submit`。 

<br />

**特别注意：初始化函数为 `init()`，需要注意的是由于IE8及以下版本不支持自定义标签，因此canForm通过 `createElement()` 方法进行变通，如果您想要支持IE8及以下浏览器，那么 `init()` 的位置必须先于自定义标签，因为IE浏览器必须在元素解析前知道这个元素，所以这个函数不能在页面底部调用。因此，若要兼容也避免在window.onload中包含它。**

<br />

# 内置类与属性

## `canScope类`

每个 `<c-scope>` 都对应于一个 `canScope` 实例：

	this.canScope = function() {
		this.enabled = true;
		this.children = [];  //canEl对象集合
	}

- `enabled` 默认为  `true`
- `children` 为当前 `scope` 所有的 `children` 的实例化 `canEl` 对象数组。

<br />

## `canEl类`

每个表单元素对应一个 `canEl` 类，不管是 `<input>` `<textarea>` `<select>` 还是 `<c-emu>`：

	this.canEl = function() {
		this.rules = ""; 	//判定规则，数组
		this.conf = "";  	//额外配置, 可为 blur,autohide,realtime,chgchk
		this.yes = "";   	//数组，判定为真添加的className或执行的function
		this.no = "";    	//数组，判定为假添加的className或执行的function
		this.end = "";   	//数组，判定完成添加的className或执行的function，执行顺序在yes和no之后
		this.start = ""; 	//数组，判定开始前添加的className或执行的function
		this.value = []; 	//c-emu类专用，对应的c-value值集合
		this.status = 0; 	//本元素目前的状态，验证成功(1)、失败(-1) or 未验证(0)
		this.scope = "";	//canScope对象，本元素所在的scope
		this.el = "";		//DOM节点，本元素对应的DOM
		this.name = "";     //供c-commit使用，识别表单元素
	}

<br />

## `canCommit类`

每个 `<c-commit>` 或 `<input type="submit">` 元素对应 `canCommit` 的一个实例：

	//各成员释义同上，canCommit无conf，value和rules
	this.canCommit = function() {
		this.end = "";
		this.start = "";
		this.yes = "";
		this.no = "";
		this.scope = "";
		this.el = "";
		this.status = 0;
	}

<br />

## `rules属性`

内置的规则集合：

	this.rules = {
		
		//正则类
		script: "<script[^>]*?>.*?</script>",
		mail: /^([a-zA-Z0-9_-])+@([a-zA-Z0-9_-])+(.[a-zA-Z0-9_-])+$/,
		qq: /^[1-9]\d{4,9}$/,
		tel: /^(\d{3}-\d{8})|(\d{4}-\d{7})$/,
		phone: /^((13[0-9])|(15[^4,\d])|(14[57])|(17[0])|(18[0,0-9]))\d{8}$/,
		
		//函数类，需要返回true or false
		required: function(value,el) { if (value=="") return false; else return true; },
		//max与min的处理方法不同，验证函数单独处理，此处避免修改
		max: function(digit) { return "^[a-zA-Z0-9_]{0,"+digit+"}$" }, 
		min: function(digit) { return "^[a-zA-Z0-9_]{"+digit+",}$" }
		
	};

规则为 `key-value` 形式，`key` 须为小写， `value` 可为正则对象或 `function`。

若为 `function` ，须返回 `true` or `false `，同时 canForm 在调用时会传入两个参数： `value` 与 `el`，前者为待测值，后者为待测 `canEl` 对象。

另外，对 `max` 和 `min`，canForm也采用正则匹配的方式，自动检测后面带的数字，这两条规则在判定阶段会区别对待，因此请避免修改或新增重名的规则。

调用举例：

	<input type="text" c-rules="qq-max10-required">

**您可以通过 `setRules` 方法新增规则，详见后面的说明。**

<br />

## `tips属性`

它是功能性提示文字，在初始化时，canForm会自动检测 `<c-yes>` `<c-no>` `<c-dp>`元素中的模板标签 `{{tpl}}` ，发现匹配值则替换，此渲染仅初始化执行，非动态。

	this.tips = {
		QQ_ERROR: "不是合法的QQ号",
		QQ_OK: "合法的QQ号"
	};

调用举例：

	<c-yes>
		{{QQ_OK}}
	</c-yes>

**您可以通过 `setTips` 方法新增规则，并以与示例相同的方法进行调用，详见后面的说明。**

<br />

# 内置可调用方法

## `changeStatus(element, status)`

手动改变某一个 `canEl` 或 `canCommit` 对象的 `status` (状态)。

@param **element** `[object canEl] | [object canCommit]`

@param **status** `[number]` 状态码：`1（验证成功）` `-1（验证失败）` `0（未验证）`

注：

> - 当设置 `status` 为 `1` ，canForm会将同级的 `<c-yes></c-yes>` 标签进行显示，并隐藏 `<c-no></c-no>`、`<c-dp></c-dp>` ，并将 `this.start`、`this.end`、`this.no` 中出现的不带`!`的`className`进行`remove`，但不执行三种状态包含的函数。
> - `status` 为 `0` 同理。
> - `status` 为 `-1` 时，canForm会将`this.start`、`this.end`、`c-yes`、`this.no` 中出现的不带`!`的`className`进行`remove`，但不执行三种状态包含的函数。

<br />

## `setRules(rules)`

动态添加一条或多条规则，尽量在 `init()` 之前调用。

@param **rules** `[object]`

	{
		one: value,
		two: value
	}

@param **key** `[string]` 规则名（读入时自动转小写）

@param **value** `[string] | [function] ` 正则表达式或函数

注：

> - 函数需要返回 `true` or `false`
> - 提供给函数两个参数：
> - @param **value** `[string]` 待测值；
> - @param **element** `[object canEl]` 待测元素；

<br />

## `setTips(tips)`

添加一条或多条提示信息，需要在 `init()` 之前调用。

@param **tips** `[object]`

	{
		one: value,
		two: value
	}

@param **key** `[string]` 提示名（读入时不转小写）

@param **value** `[string]` 提示信息

注：

> - `init()` 执行后就会渲染 `this.tips` 中的模板，因此需要在 `init()` 之前调用。

<br />

## `updateNodes()`

重新建立节点结构，注册事件，适用于动态新增表单元素等场景，作刷新功能。

调用示例：

	var cf = new canForm();
	cf.init();
	...
	cf.updateNodes();

<br />

## `ajax(config)`

canForm内置了一个简易的 `ajax` 封装。

示例：

	var cf = new canForm();
	cf.ajax({
		type: "post",
		url: "www.canform.com/getData",
		data: "hello world",
		success: function(data) {
		},
		error: function(data) {
		}
	})

@param **config** `[object]` ajax对应的参数

@param **config.type** `[string]` "post" / "get"

@param **config.url** `[string]` 请求的url

@param **config.data** `[string] | [object]` 提交的数据

@param **config.success** `[function]` 成功的回调函数，返回 `data` `[string]`

@param **config.error** `[function]` 失败的回调函数，返回 `data` `[string]`

<br />

# HTML标签与属性

##  `<c-scope>`

canForm表单作用域。在应用canForm时，请保证需要验证的表单元素都被包含在 `<c-scope></c-scope>` 中。

一个 `HTML` 中可存在若干个 `<c-scope>`，建议每份表单对应一个 `<c-scope>`。

<br />

## `<input>、<textarea>与<select>`

> 此处不包括的类型：
> 
> 1. `<input>` 中的 `submit`、`checkbox` 、`radio`
> 2. `<c-emu>` 中的带 `c-child` 属性的元素（在 `<c-emu>` 部分会提到）

这类表单元素是canForm的基础表单元素，每个元素对应于一个 `canEl` 实例。

初始化时canForm会遍历每个 `<c-scope>` ，仅当 `scope` 类的这些元素含有 `c-rules` 属性的时候才会被识别。

如下的情况不会被识别：

	<c-scope>
		<input type="text">
	</c-scope>

<br />

### 属性

#### `c-rules` （必填项）

`c-rules` 支持内置规则和通过 `setRules` 方法创建的正则或函数规则，其中自定义函数的规范请参照 `setRules` 方法的介绍。

以链式写法为基础，即以 `-` 作为多个条件的分隔，按序查找规则并执行匹配：

	<input type="text c-rules="qq-abc">

若之中有一条规则匹配失败，即验证失败。

它也支持逻辑非 `!`，表示将单条规则的验证结果取反，以canForm内置的qq号验证为例：

	!qq

代表当内部的值不是qq号时，判定成功。这对函数规则同样适用：

	var cf = new canForm();
	cf.setRules({
		demo: function(value, el) {
			if (value=="")
				return false;
			else return true;
		}	
	});
	cf.init();

我们通过 `setRules` 方法创建了一个名为 `demo` 的规则，当调用 `!demo` 时，会将原本的结果取反，因此规则变为当 `value` 为空的时候判定成功。

> 注：不支持直接的函数式写法，如 `func()`


另外，canForm内置了与其他规则较为不同的 `max` 和 `min` 的方法，调用示例如下：

	<input type="text c-rules="min5-max10">

上述例子表明该 `<input>` 需要满足不小于5位（含5位），不大于10位（含10位），请填写整数。

**若要查阅更多的内置规则，请参阅canForm的 `rules属性`。**

<br />

#### `c-conf` （可选）

`c-conf` 是一个配置参数的属性，支持链式写法，调用示例如下：

	<input type="text c-rules="min2" c-conf="realtime-autohide">

可能的取值如下：

- `realtime`，仅针对可键盘输入的表单元素，当触发 `keydown` 事件，自动执行一次检测。
- `chgchk`，意为 `change check`，仅针对原生的表单元素，当触发 `change` 事件，自动执行一次检测，与 `blur` 只能二选一。
- `blur`，仅针对原生的表单元素，当触发 `blur` 事件，自动执行一次检测，与 `chgchk` 只能二选一（都写只认 `chgchk`）。
- `autohide`，当触发 `focus` 事件，会执行如下操作：

1） `c-yes` `c-no` `c-start` `c-end` 中出现的 `className` 全都会被 `remove`，但函数不执行，且对 `!className` 不会进行操作，如：

	<input type="text c-rules="min2" 
	
	c-conf="realtime-autohide"
	
	c-yes="green-!red-func()"
	
	c-no="!green-red-func2()">

在上例中，当 `focus` 事件触发，

`c-yes`中的`class` `green`会被除掉，但 `!red` 和 `func()` 不处理；

`c-no`中的`class` `red`会被除掉，但 `!green` 和 `func2()` 不处理。

<br />

2）同时，元素同级的`<c-yes></c-yes>` `<c-no></c-no>` 均会被隐藏，`<c-dp></c-dp>` 出现。

	<c-scope>
		
		<div class="input-wrap">
			<input type="text c-rules="min2" c-conf="autohide">
			<c-yes>1</c-yes>
			<c-no>2</c-no>
			<c-dp>3></c-dp>
		</div>
		
	</c-scope>

上例仅显示3。

<br />

#### `c-yes` （可选）

`c-yes` 是包含当前元素判定成功后的回调的属性，支持链式写法。

回调的类型可以有两种，`className` 或 `function`，例：

	<input type="text c-rules="min2" c-conf="realtime" c-yes="green-!red-func()">

以上的例子表示了如下几部操作

- 1. 为当前元素添加名为 `green` 的 class
- 2. 为当前元素移除名为 `red` 的 class
- 3. 执行 `func` 函数

在每个 `c-yes` 回调的函数中，canForm将传入如下参数：

- @param **type** `[string]` 状态字，包括 "YES" "NO" "START"
- @param **element** `[object canEl] | [object canCommit]` 当前元素的 `canEl` 或 `canCommit` 实例化对象
- @param **detail** `[object]`

结构如下：

    detail = {
	   status: true,
	   yes: {
	     key:value,
		 key:value
       },
	   no: {
		 key:value,
		 key:value
	   }
	};

其中

- `status`为状态，`true` or `false`；
- `yes` 为验证成功的规则 `key-value`；
- `no` 为验证失败的规则 `key-value`；

<br />


#### `c-no` （可选）

用法与 `c-yes` 相同。

#### `c-start` （可选）

用法与 `c-yes`、`c-no`基本相同，状态字为 `START`，但在回调函数中不会返回 **`detail`** 参数。

#### `c-end` （可选）

用法与 `c-yes`、`c-no`相同，状态字为 `YES` 或 `NO`，回调函数中会返回 **`detail`** 参数。

因此，在这个属性中调用的函数，可以根据 @param **type** 来根据不同结果进行不同响应。

<br />

#### `c-name` （可选）

canForm中对表单元素的唯一标识，类似于 `<input>` 标签中的 `name` 属性，与 `<c-commit>` 配合使用，详见 `<c-commit>` 标签说明。

<br />


## `<c-emu>`

`<c-emu>` 是canForm中另一个重要的标签，在之前的介绍中，我们提到过用它来解决 `<input type="radio|checkbox">` 与模拟表单元素的问题。

> canForm使用 `c-child` 属性去标记 `<c-emu>` 中每个子元素，用 `c-value` 属性初始化 `<c-emu>` 元素 或 去代表子元素各自的值，当子元素发生了 `change`（ `input` 标签） 或 `click`（普通块级元素）事件，canForm会更新 `<c-emu>` 对应的 `canEl` 实例的值，再执行规则的判定操作。

例子（注意 `c-yes` `c-no` 的位置）：

	<div class="el-wrap">
		<c-emu c-rules="aaa" c-value="-1">
			<li c-child c-value="Asia">亚洲</li>
			<li c-child c-value="Africa">非洲</li>
		</c-emu>
		<c-yes>正确</c-yes>
		<c-no>不对</c-no>
	</div>

由于内置的 `this.value` 是一个数组（canEl对象实例），在 `<c-emu>` 中的判定不能用简单的正则处理，因此需要单独写函数来处理，关于自定义函数规范参见 `setRules` 方法。

<br />

### 私有属性

#### `c-child` （子元素必填）

标记 `<c-emu>` 的子元素，未标记的元素将不被视为子元素。

#### `c-toggle`（子元素可选）

该标记表示连续两次事件触发后是否从当前值的集合中剔除本元素的 `c-value`。

在 `<input type="checkbox" c-value="a" c-child>` 中，您必须得引入 `c-toggle` 属性。

#### `c-unique`（父元素可选）

该标记表示 `<c-emu>` 元素的值是唯一的，适用于模拟下拉框等场景，当应用此元素时，`c-toggle`会失效。

#### `c-value`（子元素必填，且不重复）

该值可出现在两处：

- `<c-emu>` 标签处，在初始化时给对应的canEl的 `this.value` 一个默认值。
- 带有 `c-child` 的表单元素，这是**必填项**，同时请保证值的**唯一性**。

<br />

### 公用属性

#### `c-rules`（父元素必填）、`c-yes`、`c-no`、`c-start`、`c-end` （父元素可选）

用法请参见上篇介绍，注意 `<c-emu>` 没有 `c-conf` 属性。

<br />

## `<c-yes> <c-no> <c-dp>`

这些标签为提示信息标签，分别代表验证成功、验证失败、未验证三种情况下的提示信息，非必有项。

- 默认 `<c-dp>` 显示，而 `<c-yes>` `<c-no>` 隐藏。
- 您可以自由地在这三个标签中插入HTML代码。
- 它们都支持`tips`模板，详细请参阅 `tips属性`。
- 支持携带他们的元素： `<input>` `<textarea>` `<select>` `<c-emu>` 等支持 `c-rules` 属性的元素，以及 `<c-commit>` 与 `<input type="submit">` 表单提交元素。

- 在使用他们的时候，请保证携带他们的元素与他们同级：

		<div class="input-wrap">
			<input type="text" c-rules="aaa">
			<c-yes>
				哈哈哈
			</c-yes>
		</div>

	canForm的建议是用一个标签，如 `div.input-wrap` 把每个需要采用提示标签的元素和提示标签一起包起来，清晰可读。

<br />

## `<c-commit>与<input type="submit">`

这两种标签起表单提交的作用。当遇到 `<input type="submit">` ，canForm会自动将其识别为 `<c-commit>`。

点击后的情况分为两种：

**1. 有原生 `<form>` 存在：**

	首先，若您的需求中有 `<form>` 原生的表单，结构应当如下：
	<c-scope>
		...
		<form action="...">
			//提交按钮方式1：利用canForm定义的 `<c-commit>` 标签提交
			<c-commit></c-commit>
			//提交按钮方式2：原生 `<input type="submit">` 方式提交
			<input type="submit">
		</form>
	</c-scope>

点击提交按钮后，这两个标签会依据已有结构，根据 `this.scope` 找到父级，并通过父级的 `this.children` 遍历所有元素并重新检查一遍规则，如果有匹配失败的情况，会终止表单提交，匹配成功则会执行 `form.submit()` 方法提交表单。

<br />

**2. 没有原生 `<form>` 存在：**
	
适用于非原生 `form.submit()` 表单提交的情况，如 `ajax + json` ，此时一般情况下是手动提取各元素的 `key-value`，canForm可帮助您更自动地完成。

	结构可能如下：
	<c-scope>
		<input type="text" c-name="aaa" c-rule="qq" c-conf="realtime-autohide">
		...
		//提交按钮方式1：利用canForm定义的 `<c-commit>` 标签提交
		<c-commit></c-commit>
		//提交按钮方式2：原生 `<input type="submit">` 方式提交
		<input type="submit">
	</c-scope>

在对需要采集数据的表单元素定义 `c-name` 属性，并赋值后，当提交表单时:
	
**如果他们自带了 `c-yes` 或 `c-end` 属性，且链中有 `function`，如：**
	
	<c-commit c-yes="green-upload()"></c-commit>
	<c-commit c-end="green-upload()"></c-commit>

那么在 `upload` 这个函数中，canForm原本会传入如下参数：
	
- @param **type** `[string]` 状态字，包括 "YES" "NO" "START"
- @param **element** `[object canEl] | [object canCommit]` 当前元素的 `canEl` 或 `canCommit` 实例化对象
- @param **detail** `[object]`
	
在以上基础之上会返回第四个参数：
	
- @param **result** `[object]`
	
这个参数返回的是父级 `<c-scope>` 中所有带 `c-name` 的表单元素的 `key-value` 集，如：
	
	<c-scope>
		<input type="text" c-name="input_a" c-rule="a" c-conf="realtime"> //假定此时 value 为 1
		<input type="text" c-name="input_b" c-rule="a" c-conf="realtime"> //假定此时 value 为 2
	</c-scope>
	
返回的 `result` 格式：
	
	result = {
		input_a: 1,
		input_b: 2	
	}

之后您可以用过canForm自带的 `ajax` 方法与后台进行数据交互。

<br />
以上两种方式全都支持公用属性，公用属性列表如下：

### 公用属性

#### `c-yes`、`c-no`、`c-start`、`c-end` （可选）

除了 `c-yes` 和 `c-end` 新增的第四个参数，其他用法请参见前面的介绍，注意 `<c-commit>` 没有 `c-conf` `c-rules` 属性。
