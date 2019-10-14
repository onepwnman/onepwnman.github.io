---
layout: post
title: "Disjoint-Set (Union-Find) Data Structure"
tags: [Algorithm & Data Structure, Tree, Python]
---

*Disjoint-Set* 자료구조는 많은 서로소 부분 집합들로 나눠진 원소들에 대한 정보를 저장하고 조작하는 자료 구조입니다.[^1]  *Disjoint-Set*은 **Union**과 **Find**연산을 제공하며 Union-Find Set이라 불리기도 합니다.  

**Union**과 **Find**연산은 Linkedlist와 Tree로 구현될 수 있으며 Linkedlist로 구현시에 시간복잡도는 *O(n)* 시간이지만 Tree로 구현시 최적화를 통해 최대 *$$O(\alpha(n))$$*시간으로 줄일 수 있습니다.      
$$O(\alpha(n))$$은 아커만 함수(Ackermann function)의 역함수로 5이하의 값을 가지기 때문에 실질적으로 $$O(1)$$시간으로 보아도 무방합니다.     
또한 Disjoint-Set 자료구조는 최소신장트리(Minimum Spanning Tree)를 찾는 쿠르스칼(Kruskal) 알고리즘에 사용됩니다.     
위의 설명만으로는 이해가 잘 안될 수 있습니다.  
이제부터 그림을 보며 설명 하겠습니다.    
  
<br> <br> 
다음 그림과 같이 7개의 부분 집합이 있고 그 밑의 array에는 각 부분집합의 부모의 값을 가지고 있다고 해보겠습니다.
![Disjoint-set](/assets/images/disjoint-set/disjoint-set1.jpg "Disjoint-Set")


초기의 부분집합은 원소가 각각 하나이기 때문에 각각의 부모는 자기자신이 됩니다.           


              













  
    
       
    

<br> <br> <br> <br> <br> <br>
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



  
[^1]: <https://en.wikipedia.org/wiki/Disjoint-set_data_structure>
