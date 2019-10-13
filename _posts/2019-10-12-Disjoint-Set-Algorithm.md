---
layout: post
title: "Disjoint-Set (Union-Find) Data Structure"
tags: [Algorithm & Data Structure, Tree, Python]
---

*Disjoint-Set* 자료구조는 많은 서로소 부분 집합들로 나눠진 원소들에 대한 정보를 저장하고 조작하는 자료 구조입니다.[^1]  *Disjoint-Set*은 **Union**과 **Find**연산을 제공하며 Union-Find Set이라 불리기도 합니다. **Union**과 **Find**연산은 Linkedlist와 Tree로 구현될 수 있으며 Linkedlist로 구현시에 	 앞의 설명만으로는 Disjoint-Set이 무엇인지 잘 이해가 가지 않을 것이다. 예를 통해 하나씩 살펴보자.   


[^1]: <https://en.wikipedia.org/wiki/Disjoint-set_data_structure>

![Disjoint-set](/assets/images/disjoint-set/disjoint-set1.jpg "Disjoint-Set")

|  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center> |  <center>1</center>  |




### Highlighted Code Blocks

To modify styling and highlight colors edit `/_sass/_highlighter.scss`.


```css
#container {
    float: left;
    margin: 0 -240px 0 0;
    width: 100%;
}
```

```html
{% raw %}<nav class="pagination" role="navigation">
    {% if page.previous %}
        <a href="{{ site.url }}{{ page.previous.url }}" class="btn" title="{{ page.previous.title }}">Previous article</a>
    {% endif %}
    {% if page.next %}
        <a href="{{ site.url }}{{ page.next.url }}" class="btn" title="{{ page.next.title }}">Next article</a>
    {% endif %}
</nav><!-- /.pagination -->{% endraw %}
```

```ruby
module Jekyll
  class TagIndex < Page
    def initialize(site, base, dir, tag)
      @site = site
      @base = base
      @dir = dir
      @name = 'index.html'
      self.process(@name)
      self.read_yaml(File.join(base, '_layouts'), 'tag_index.html')
      tag_title_suffix = site.config['tag_title_suffix'] || '&#8211;'
      self.data['title'] = "#{tag_title_prefix}#{tag}"
      self.data['description'] = "An archive of posts tagged #{tag}."
    end
```


### Standard Code Block

    {% raw %}<nav class="pagination" role="navigation">
        {% if page.previous %}
            <a href="{{ site.url }}{{ page.previous.url }}" class="btn" title="{{ page.previous.title }}">Previous article</a>
        {% endif %}
        {% if page.next %}
            <a href="{{ site.url }}{{ page.next.url }}" class="btn" title="{{ page.next.title }}">Next article</a>
        {% endif %}
    </nav><!-- /.pagination -->{% endraw %}

### GitHub Gist Embed

An example of a Gist embed below.

<script src="https://gist.github.com/mmistakes/43a355923921d22cd993.js"></script>




