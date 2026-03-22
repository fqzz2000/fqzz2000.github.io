---
layout: homepage
---

## Blog

I write about AIOps, agent infrastructure, and research at the frontier of the agent era.

{% for post in site.posts %}
- **[{{ post.date | date: "%b %d, %Y" }}]** [{{ post.title }}]({{ post.url }})
{% endfor %}
