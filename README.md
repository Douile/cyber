# Cyber
<style>
.centered-flex {
  display: flex;
  flex-flow: column nowrap;
  width: 100%;
  max-width: 900px;
}
.centered-flex * {
  margin: 2em;
  padding: 1em;
  border-radius: 7px;
  border: 1px solid #fff;
  cursor: pointer;
}
* {
  box-sizing: border-box;
}
</style>
<ul>
  {% for post in site.posts %}
    <div class="centered-flex">
      <a href="{{ post.url }}">{{ post.title }}</a>
    </div>
  {% endfor %}
</ul>
