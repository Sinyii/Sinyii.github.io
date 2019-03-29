---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---
<html>
  <head>
    <meta charset="utf-8">
    <title>{{ page.title }}</title>
  </head>
  <body>
    {% for post in site.categories.blog %}
        <div>
            <a href="{{ post.url }}" ><img src="{{ post.thumbnail }}" />
            <a href="{{ post.url }}" >{{ post.title }}</a>
        </div>
    {% endfor %}
  </body>
</html>