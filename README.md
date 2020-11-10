<style>
.root-flex {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  alight-items: center;
}
.centered-flex {
  display: flex;
  flex-flow: column nowrap;
  width: 100%;
  max-width: 900px;
}
.centered-flex > * {
  margin: .5em 0;
  padding: 1em;
  border-radius: 7px;
  border: 1px solid #fff;
  cursor: pointer;
  text-decoration: none;
  display: grid;
  grid-template-columns: auto 1fr;
  grid-gap: .5em;
  color: #fff;
  text-shadow: none;
}
.centered-flex > *:hover {
  text-decoration: underline;
}
* {
  box-sizing: border-box;
}
</style>
<div class="root-flex">
<div class="centered-flex">
  {% for post in site.posts %}
  <a href="{{ post.url }}"><span>{{post.date | date: "%Y-%m-%d"}}</span><span>{{ post.title }}</span></a>
  {% endfor %}
</div>
</div>
