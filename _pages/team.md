---
layout: collection
entries_layout: grid
title: "Group"
permalink: /team/
author_profile: true
---

## Staff

{% for member in site.data.team_members %}
  <h4>{{ member.name }}</h4>
  <i>{{ member.info }} <!--<br>email: <{{ member.email }}></i> -->
  <ul style="overflow: hidden">
{% endfor %}

