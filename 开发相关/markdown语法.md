# Typora中的行内跳转
[参考](https://support.typora.io/Links/#faq)
可以使用 `#` 创建链接，指向任何markdown文件中的头，例如：
```md
# This is a title
...
...
...
A[link](#This-is-a-title) 跳转到目标头
```
也可以使用原生HTML写**named**锚点:
```html 
<a name="anchor">Anchor</a>
<a href="#anchor">Link to Anchor</a>
```