平时我们写`JS`代码时会有一些架构的设计，`css`也不例外，它也有自己的自己架构设计，尤其是在组件库中。今天带来的是`BEM`架构在`Element Plus`组件库中的设计详解，包括代码的实现，以及最重要的——这样设计的目的是什么。
## 前置知识
### BEM
`BEM` 是由 `Yandex` 团队提出的一种 `CSS` 命名方法论，是`OOCSS`模式的一种实现，`BEM`实际上是`block`、`element`、`modifier`的缩写，分别为块层、元素层、修饰符层，命名格式为：
```scss
block-name__<element-name>--<modifier-name>-<modifier_value>
```

- `block`与element使用`__`相接
- modifier与前面修饰的结构用`--`相接（可以与block或element相接）
- 修饰符的值用`-`与修饰符层相连

例如`button`组件
```html
<button class="button">
  <div class="button__inner button__inner--primary">
    button
  </div>
</button>
```
在组件库中，每一个组件被分为一个个的`block`，组件中的每一块被分成了一个个的`element`，对`block`或`element`的修饰被分成一个个的`modifier`，这样不但有效解决了`css`类名的命名冲突，也方便了我们维护。
### 调试
我们写的是`scss`文件，调试`scss`文件可使用`sass`npm包来完成，先全局安装
```scss
npm install -g sass
```
先在项目中使用`@debug`打印在控制台或使用`@error`抛出错误。
然后对`scss`文件所在目录运行如下命令，可以使用`-w`开启监听模式，若`scss`文件或所依赖的文件发生变化，将自动重新编译
```scss
sass .\icon.scss icon.css -w
```
在你的`scss`文件没有错误的情况下，以上命令将`icon.scss`编译为`icon.css`文件。
## 组件库中的BEM模式有哪些写法
`BEM`的`scss`一共有以下几种形态

- 常规写法
- `b--m`嵌套`b__e`
- 伪类或表示状态的`.is-`选择器中嵌套`b__e`
- 特殊选择符`+、~` 等的衔接
- 共享和继承
- 多个`element`或多个`modifier`共享

组件库中常见的写法有这些，一个组件这样写还好，如果是所有的组件的样式都要加上`__`、`--`等符号，写起来将会特别麻烦，接下来我们就看看怎么来解决这个问题。

## 常规写法
BEM架构最平常的写法就是这样，我们拿`b、e、m`分别为`icon、item、color`举例：
```scss
.icon {
  &__item {
    &--color {}
  }
}
```
`block`后面接`element`，`element`后面接`modifier`，现在我们要简化的就是拼接`__e`和`--m`的过程，所以我们使用`定义混合指令``@mixin`来表示`b、e、m`，在这之前我们先定义一些常量
```scss
$common-separator: '-' !default; // 公共的连接符
$element-separator: '__' !default; // 元素以__分割
$modifier-separator: '--' !default; // 修饰符以--分割
$state-prefix: 'is-' !default; // 状态以is-开头
```
然后定义三个`@mixin`
```scss
@mixin b($block) {
  .#{$block} {
    @content;
  }
}

@mixin e($element) {
  $selector: &; // & 里面保存着上下文，在这个 mixin 中其实指的就是 block

  @at-root {
    // @at-root 指规则跳出嵌套，写在最外层
    #{$selector + $element-separator + $element} {
      @content;
    }
 }
}

@mixin m($modifier) {
  $selector: &;

  @at-root {
    #{$selector + $modifier-separator + $modifier} {
      @content;
    }
   }
}
```
写一个例子试一下
```scss
@include b(icon) {
  --color: inherit;
  height: 1em;

  svg {
    height: 1em;
    width: 1em;
  }

  @include e(item) {
    width: 0.5rem;
    @include m(color) {
      color: aqua;
    }
  }
}
```
编译后的css如下
```scss
.icon {
  --color: inherit;
  height: 1em;
}
.icon svg {
  height: 1em;
  width: 1em;
}
.icon__item {
  width: 0.5rem;
}
.icon__item--color {
  color: aqua;
}
```
可以看到编译后的`css`并没有生成`后代选择器`，也就是类似`.icon .icon__item`，可以知道生成的`scss`文件中类名选择器并没有嵌套，而是拼接后被`@at-root`提到最外面，可以推知拼接类名后的`scss`的形式为：
```scss
.icon {
  --color: inherit;

  svg {
    height: 1em;
    width: 1em;
  }

  @at-root {
    .icon--item {
      color: aqua;

      @at-root {
        .icon__item--color {
          width: 0.5rem;
        }
      }
      
    }
  }
}
```
有同学疑惑到底是为啥需要`@at-root`，我们让只有`block`和`element`的例子试一下如果没有`@at-root`会是怎样：
```scss
// 转换前
@include b(icon) {
  --color: inherit;

  svg {
    height: 1em;
    width: 1em;
  }

  @include e(item) {
    color: aqua;
  }
}

// 转换后的scss
.icon {
  --color: inherit;

  svg {
    height: 1em;
    width: 1em;
  }

  .icon--item {
    color: aqua;
  }
}

// 编译后的css
.icon {
  --color: inherit;
}
.icon svg {
  height: 1em;
  width: 1em;
}
.icon .icon--item {
  color: aqua;
}
```
加上`modifier`试一下
```scss
// 转换后的css
.icon {
  --color: inherit;
}
.icon svg {
  height: 1em;
  width: 1em;
}
.icon .icon__item {
  color: aqua;
}
.icon .icon__item .icon .icon__item--color {
  width: 0.5rem;
}

// 可以推知转换前的scss的形态
.icon {
  --color: inherit;

  svg {
    height: 1em;
    width: 1em;
  }

  .icon--item {
    color: aqua;

  	.icon__item--color {
      width: 0.5rem;
    }
  }
}
```
可以看到如果没有`@at-root`，将会无限嵌套。
所以得出`@at-root`的作用：在编译时不进行嵌套形成（类似`.icon .icon__item`这种选择器） 
## b--m嵌套b__e
按`b、e、m`的顺序生成已经可以，但是我们可能要生成`.icon--color`内嵌`.icon__item`，也就是`b--m`内嵌`b__e`，我们先试一下如果直接写：
```scss
@include b(icon) {
  --color: inherit;

  @include m(color) {
    color: aqua;
    @include e(item) {
      width: 0.5rem;
    }
  }
}

// 编译后的css
.icon {
  --color: inherit;
}

.icon--color {
  color: aqua;
}

.icon--color__item {
  width: 0.5rem;
}

// 实际上期望的
.icon--color {
  color: aqua;
}
.icon--color .icon__item {
  width: 0.5rem;
}
```
所以当`@mixin e()`的父选择器的类名是`modifier`的时候不进行类名拼接，而是嵌套。
创建一个`function.scss`文件来放置函数，因为选择器是一个数组（`scss`的数组形式是`(,,,)`），而判断类名用到的字符串方法的操作对象是字符串，所以需要先将数组转为字符串，我们抽成一个函数：

```scss
@function selectorToString($selector) {
  $selector: inspect($selector); // Debug: (.icon--color,)
  $selector: str-slice($selector, 2, -2); // 把前后的括号删掉
  @return $selector;
}
```
关于Sass字符串的方法可以参考[Sass String Functions](https://www.w3schools.com/sass/sass_functions_string.php)和[Sass inspect() Function](https://wikimass.com/sass/inspect)
再判断`modifier`
```scss
// 判断是否有Modifier
@function containsModifier($selector) {
  $selector: selectorToString($selector);
	// str-index()返回 $modifier-separator在$selector的位置
  @if str-index($selector, $modifier-separator) {  
    @return true;
  } @else {
    @return false;
  }
}
```
我们还需要或许被嵌套的`b__e`中`block`的名称
`block`的名称可以通过传入的字符串`$selector`切割获得，也可以利用全局变量，这里贴上字符串切割代码
```scss
@import "function";
$modifier-string: selectorToString($selector);
$ms-index: str-index($modifier-string, $modifier-separator);
$end-index: $ms-index + 1;
$block: str-slice($modifier-string, 2, $ms-index - 1);
```
不过我们还是通过全局变量来实现更为简洁，在`@mixin b()`和`@mixin e()`都加上全局变量，`element`的名称后面也可能用上
根据`function.scss`的函数改写`@mixin e()`
在`@mixin e()`里面判断上层是否是`b--m`，如果是，手动进行嵌套（因为`@at-root的存在`）且改变类名
```scss
@mixin b($block) {
  $B: $block !global;
  .#{$B} {
    @content;
  }
}

@mixin e($element) {
  $selector: &;
  $E: $element !global;

  @if containsModifier($selector) {
    // 如果这是一个b--m，也就是先走了@mixin m()，里面就变成b--m嵌套b__e
    @at-root {
      #{$selector} {  // b--m
        .#{$B + $element-separator + $E} {  // b__e
          @content;
        }
      }
    }
  } @else {
    @at-root {
      // @at-root 指规则跳出嵌套，写在最外层
      #{$B + $element-separator + $E} {
        @content;
      }
    }
  }
}

```
## 伪类或状态state中嵌套b__e
我们约定使用`.is-xxx`类名选择器来表示`dom`节点的状态，而且将伪元素和`state`也用`@mixin`表示
```scss
@mixin when($state) {
  $selector: &;
  @at-root {
    // 这里不是拼接，而是使用兄弟
    #{$selector + "." + $state-prefix + $state} {
      @content;
    }
  }
}
// 伪类
@mixin pseudo($pseudo) {
  $selector: &;
  @at-root {
    #{$selector + ":" + $pseudo} {
      @content;
    }
  }
}
```
当在`@mixin e`中没有处理外层的伪类和`state`时，依旧是直接将`__e`直接拼上去
```scss
@include b(icon) {
	@include pseudo(hover) {
    @include e(side) {
      color: blueviolet;
    }
  }
  @include when(click) {
    @include e(side) {
      color: darkcyan;
    }
  }
}

// 编译后的css
.icon:hover__side {
  color: blueviolet;
}

.icon.is-click__side {
  color: darkcyan;
}
```
所以这里的处理方法和前面`b--e`嵌套`b__e`一样，即判断外层是否有表示状态的`.is-`或伪类，我们将这几个判断合并在一起写在`function.scss`中
```scss
// 判断是否有表示状态的 .is-
@function containWhenFlag($selector) {
  $selector: selectorToString($selector);

  @if str-index($selector, "." + $state-prefix) {
    @return true;
  } @else {
    @return false;
  }
}

// 判断是否有伪类
@function containPseudoClass($selector) {
  $selector: selectorToString($selector);

  @if str-index($selector, ":") {
    @return true;
  } @else {
    @return false;
  }
}

// 判断是否含有 Modifier、表示状态的 .is- 和 伪类
@function hitAllSpecialNestRule($selector) {
  @return containsModifier($selector) or containWhenFlag($selector) or containPseudoClass($selector);
}

```
然后将`@mixin e`的`containsModifier`换成`hitAllSpecialNestRule`即可
再编译一下上面的`scss`
```scss
// 编译后的css
.icon:hover .icon__side {
  color: blueviolet;
}

.icon.is-click .icon__side {
  color: darkcyan;
}
```
## 特殊选择符拼接
在使用`>`、`+`等符号时，右边的选择器就得自己写了，比如：
```scss
@include b(icon) {
  &:focus + .icon__item {
    position: inherit;
  }
}
```
为了方便，写一个`@mixin spec-selector`来进行特殊选择符的拼接
```scss
@mixin spec-selector($specSelector: "", $element: $E, $modifier: false, $block: $B) {
  $modifierCombo: "";
  // 判断输出的是 b__e 还是 b__e--m
  @if $modifier {
    // 如果是b__e--m，就将m接上
    $modifierCombo: $modifier-separator + $modifier;
  }

  @at-root {
    // 默认是父级用特殊符号接上目前的b__e(--m)，bem都是可选的，
    #{&}#{$specSelector}.#{$block + $element-separator + $element + $modifierCombo} {
      @content;
    }
  }
}
```
`@mixin spec-selector`支持自定义右边选择器，默认是当前的`b__e`，这是在大部分应用场景下的情况，`modifier`默认为`false`，也就是不拼接上`modifier`
## 共享
首先，当有多个元素`@extends`一个占位符或者选择器时，会被编译为下面这样
> Sass 额外提供了一种特殊类型的选择器：占位符选择器 (placeholder selector)。与常用的 id 与 class 选择器写法相似，只是 # 或 . 替换成了 %。必须通过 [@extend](https://www.sass.hk/docs/#t7-3) 指令调用，更多介绍请查阅 [@extend-Only Selectors](https://www.sass.hk/docs/#t7-3-6)。

```scss
%share {
  position: absolute;
  left: 50%;
  top: 50%;
}

.test1 {
  @extend %share;
  color: antiquewhite;
}
.test2 {
  @extend %share;
}
.test3 {
  @extend %share;
}

// 编译后的css
.test3, .test2, .test1 {
  position: absolute;
  left: 50%;
  top: 50%;
}

.test1 {
  color: antiquewhite;
}
```
当`%share`在其他作用域时，会被编译成下面这样
```scss
.box {
  %share-test {
    position: absolute;
    left: 50%;
    top: 50%;
  }
}

.test1 {
  @extend %share-test;
  color: antiquewhite;
}
.test2 {
  @extend %share-test;
}
.test3 {
  @extend %share-test;
}

// 编译后的css
.box .test3, .box .test2, .box .test1 {
  position: absolute;
  left: 50%;
  top: 50%;
}

.test1 {
  color: antiquewhite;
}
```
所以，如果当`%share`在`@include b()`里面时，他的作用域时在`.icon`里面，编译出来的`css`选择器前面就会多出一个`.icon`，这个是不符合预期的，因为我们的`element`写在`b`里面只是为了拼接，而不是嵌套
```scss
@include b(item) {
  %share {
    position: absolute;
    left: 50%;
    top: 50%;
  }

  @include e(item) {
    @extend %share;
  }
}

// 会被编译成下面的css
.icon .icon__item {
  position: absolute;
  left: 50%;
  top: 50%;
}

// 也就是说scss是这样的
.icon {
  .icon__item {
    position: absolute;
    left: 50%;
    top: 50%;
  }
}
```
解决方法，就是利用`@at-root`把`%share`提到`@include b(item)`外面去，这样`%share`就不在`.icon`里面了
```scss
@mixin share-rule($name) {
  $rule-name: "%shared-" + $name;
  @at-root #{$rule-name} {
    @content;
  }
}

@mixin extend-rule($name) {
  @extend #{"%shared-" + $name};
}
```
重新编译
```scss
@include b(item) {
    @include share-rule(position) {
    position: absolute;
    left: 50%;
    top: 50%;
  }
  @include e(item) {
    @include extend-rule(position);
  }
}

// 编译后
.icon__item {
  position: absolute;
  left: 50%;
  top: 50%;
}
```
## 多个element或多个modifier共享
如果有多个`element`（`modifier`同理）使用同一套样式，在没有创建这些`element`之前（或者说这些`element`需要其他样式），我们需要这样写：
```scss
@include b(item) {
  @include share-rule(color-shared) {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
  }
  @include e(white) {
    @include extend-rule(color-shared);
  }
  @include e(black) {
    @include extend-rule(color-shared);
  }
  @include e(origin) {
    @include extend-rule(color-shared);
  }
  @include e(blue) {
    @include extend-rule(color-shared);
  }
  // ...
}
```
要创建很多次`element`，每次都要去extend，特别麻烦；我们试试在创建`element`的时候就生成一套公用的样式：
```scss
@include b(icon) {
  // 传入不确定个参数
  @include e(white, black, origin, blue) {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
  }
}
```
让我们改写`@mixin e()`，里用参数数组传入多个参数
> 这里的`$element...`类似JS的拓展运算符，`$element`为数组

```scss
@mixin e($element...) {
  $selector: &;
  $E: $element !global;
  $currentSelector: "";
  // 用@each in遍历数组，在每一个选择器后面加逗号
  @each $unit in $element {
    $currentSelector: #{$currentSelector + "." + $B + $element-separator + $unit + ","};
  }
  @if hitAllSpecialNestRule($selector) {
    @at-root {
        #{$currentSelector} {
          @content;
        }
      }
    }
  } @else {
    @at-root {
      #{$currentSelector} {
        @content;
      }
    }
  }
}
```
编译后的`css`如下
```scss
.icon__white, .icon__black, .icon__origin, .icon__blue {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}
```
`@mixin modifier()`也是同样道理
```scss
@mixin m($modifier...) {
  $selector: &;
  $currentSelector: "";
  @each $unit in $modifier {
    // 这里$selector带了. ，不用加 .
    $currentSelector: #{$currentSelector + $selector + $modifier-separator + $unit + ","};
  }
  @debug $currentSelector;
  @at-root {
    #{$currentSelector} {
      @content;
    }
  }
}
```
测试一下
```scss
@include b(icon) {
  // 传入不确定个参数
  @include e(item) {
    @include m(primary, err) {
      color: red;
    }
  }
}

// 编译后
.icon__item--primary, .icon__item--err {
  color: red;
}
```

## 总结
可以大概总结一下这个过程：遇到`bem、伪类、状态is-、特殊选择符`就拼接，在`element`中遇到父选择器是`modifier、状态is-、伪类`就嵌套，通过混合指令，在组件库中我们能更好地使用`BEM`架构去构建样式。
相关代码链接：[https://github.com/plutoLam/pluto-ui/tree/master/src/theme/src](https://github.com/plutoLam/pluto-ui/tree/master/src/theme/src)
希望这篇文章对屏幕前的你有帮助，原创不易，欢迎点赞、收藏、转发、关注~~！
