---
layout: default
title: Notes
permalink: /notes/
---
<div class="blog-index">
  <h1>Notes</h1>
  <p class="subtitle">Development notes from across the CaseHub ecosystem.</p>

  {% assign notes = site.notes | sort: 'date' | reverse %}
  {% if notes.size == 0 %}
  <p style="color:var(--text-muted);font-size:14px;margin-top:32px;">No notes yet — repos are just getting started.</p>
  {% else %}
  <ul class="post-list">
    {% for note in notes %}
    <li class="post-list-item">
      <div class="meta">
        {{ note.date | date: "%b %-d, %Y" }}
        {% if note.repo %}&nbsp;·&nbsp;{{ note.repo }}{% endif %}
      </div>
      <a href="{{ note.url }}" class="title">{{ note.title }}</a>
      {% if note.excerpt %}
      <p class="excerpt">{{ note.excerpt | strip_html | truncate: 220 }}</p>
      {% endif %}
      {% if note.tags and note.tags.size > 0 %}
      <div class="tags">
        {% for tag in note.tags %}<span class="tag">{{ tag }}</span>{% endfor %}
      </div>
      {% endif %}
    </li>
    {% endfor %}
  </ul>
  {% endif %}
</div>
