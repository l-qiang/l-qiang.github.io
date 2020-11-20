---
title: 关于Select2在Bootstrap模态框无法获得焦点问题的解决办法
date: 2019-07-22 19:21:13
categories: ["前端"]
tags: ["Select2", "Bootstrap"]
toc: false
---

虽然，我比较少写前端的东西，但是遇到这种问题我的直觉告诉我将input框的z-index设大应该就行了。然后，我的直觉错了，改了z-index并没有解决。



于是，我就在网上搜了搜，发现有一下两种解决办法：

1. 把页面中的 tabindex="-1" 删掉
2. $.fn.modal.Constructor.prototype.enforceFocus = function() {};

但是不太熟悉的人表示完全不知道说了啥，然后我在Select2官方文档中找到了[解决办法](https://stackoverflow.com/questions/18487056/select2-doesnt-work-when-embedded-in-a-bootstrap-modal/19574076#19574076)（由于官网之前的链接失效了，所以这附上Stack Overflow的）。

<!--more-->

```html
<div id="myModal" class="modal fade" tabindex="-1" role="dialog" aria-hidden="true">
    ...
    <select id="mySelect2">
        ...
    </select>
    ...
</div>

...

<script>
    $('#mySelect2').select2({
        dropdownParent: $('#myModal')
    });
</script>
```

造成的原因也说明了，`bootstrap的模态框`会`窃取`模态框之外的`焦点`，而select2的下拉菜单是附在body元素（模态框之外）上的。 所以配置`dropdownParent`就行了，亲自试了一下确实有效。而且不一定指定模态框，指定form元素我也亲自试了没问题。



至于这个方法

```javascript
// Do this before you initialize any of your modals
$.fn.modal.Constructor.prototype.enforceFocus = function() {};
```

一定要看清楚是在模态框初始化之前执行



