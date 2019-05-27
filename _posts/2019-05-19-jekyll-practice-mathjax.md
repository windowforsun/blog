--- 
layout: single
classes: wide
title: "[Jekyll] Jekyll 에 MathJax 으로 수식 적용하기"
header:
  overlay_image: /img/blog-bg.jpg
excerpt: 'Jekyll 블로그에 MathJax 로 수식을 표현하자'
author: "window_for_sun"
header-style: text
categories :
  - Jekyll
tags:
    - Blog
    - Jekyll
    - MathJax
use_math : true
---  

## MathJax 이란
- Jekyll Github 블로그에 수식 표현이 아래와 같이 가능하다.

$f(x) = x^2 + x$

## MathJax 적용하기
- `_config.yml` 파일에서 markdown engine 이 `kramkdown` 인지 확인하고 아니라면 변경해 준다.

```markdown
markdown: kramdown
```  

- `_include/math_support.html` 파일을 만들고 아래 내용을 작성한다.

```html
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
    TeX: {
      equationNumbers: {
        autoNumber: "AMS"
      }
    },
    tex2jax: {
    inlineMath: [ ['$', '$'] ],
    displayMath: [ ['$$', '$$'] ],
    processEscapes: true,
  }
});
MathJax.Hub.Register.MessageHook("Math Processing Error",function (message) {
	  alert("Math Processing Error: "+message[1]);
	});
MathJax.Hub.Register.MessageHook("TeX Jax - parse error",function (message) {
	  alert("Math Processing Error: "+message[1]);
	});
</script>
<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
```  

- `_layout/default.html` 의 `head` 부분에 아래 내용을 추가한다.
{% raw %}
```markdown
{% if page.use_math %}
{% include mathjax_support.html %}
{% endif %}
```  
{% endraw %}

- 수식을 사용하는 포스트를 작성할때 아래와 같이 선언후 사용한다.
	- use_math : true

```markdown
---
title: "[Jekyll] Jekyll 에 MathJax 으로 수식 적용하기"
author: "window_for_sun"
header-style: text
categories :
  - Jekyll
tags:
    - Blog
    - Jekyll
    - MathJax
use_math : true
---
```  

## MakthJax 수식 표현하기
### 인라인 수식 표현
- `$ ~~~ $` ~~~ 부분에 표현할 수식을 넣어 준다.

```markdown
$f(x) = x^2 + x$
```  

$f(x) = x^2 + x$

### 수식 표현
- `$$ ~~~ $$` ~~~ 부분에 표현할 여러줄의 수식을 넣어 준다.

```markdown
$$
\begin{align}
\mbox{Union: } & A\cup B = \{x\mid x\in A \mbox{ or } x\in B\} \\
\mbox{Concatenation: } & A\circ B  = \{xy\mid x\in A \mbox{ and } y\in B\} \\
\mbox{Star: } & A^\star  = \{x_1x_2\ldots x_k \mid  k\geq 0 \mbox{ and each } x_i\in A\} \\
\end{align}
$$
```  

$$
\begin{align}
\mbox{Union: } & A\cup B = \{x\mid x\in A \mbox{ or } x\in B\} \\
\mbox{Concatenation: } & A\circ B  = \{xy\mid x\in A \mbox{ and } y\in B\} \\
\mbox{Star: } & A^\star  = \{x_1x_2\ldots x_k \mid  k\geq 0 \mbox{ and each } x_i\in A\} \\
\end{align}
$$

```markdown
$$
K(a,b) = \int \mathcal{D}x(t) \exp(2\pi i S[x]/\hbar)
$$
```  

$$
K(a,b) = \int \mathcal{D}x(t) \exp(2\pi i S[x]/\hbar)
$$

```markdown
$$
\lim_{x\to 0}{\frac{e^x-1}{2x}}
\overset{\left[\frac{0}{0}\right]}{\underset{\mathrm{H}}{=}}
\lim_{x\to 0}{\frac{e^x}{2}}={\frac{1}{2}}
$$
```  

$$
\lim_{x\to 0}{\frac{e^x-1}{2x}}
\overset{\left[\frac{0}{0}\right]}{\underset{\mathrm{H}}{=}}
\lim_{x\to 0}{\frac{e^x}{2}}={\frac{1}{2}}
$$

---
## Reference
[MathJax, Jekyll and github pages](http://benlansdell.github.io/computing/mathjax/)  



