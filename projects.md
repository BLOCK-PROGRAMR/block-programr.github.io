---
layout: default
title: Projects
permalink: /projects/
---

<div class="home">
  <h1>Projects</h1>
  <div class="project-grid">
    {% if site.projects.size == 0 %}
      <p>No projects added yet. Coming soon!</p>
    {% endif %}
    {% assign sorted_projects = site.projects | sort: "date" | reverse %}
    {% for project in sorted_projects %}
      <a href="{{ project.url | relative_url }}" class="project-box">
        <h2>{{ project.title }}</h2>
        <p>{{ project.date | date: "%B %Y" }}</p>
        <p>{{ project.description }}</p>
        {% if project.tech %}
          <small><strong>Tech:</strong> {{ project.tech | join: ", " }}</small>
        {% endif %}
      </a>
    {% endfor %}
  </div>
</div>
