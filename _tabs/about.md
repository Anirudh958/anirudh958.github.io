---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

{% assign profile = site.data.profile %}

> Professional photo and bio will be updated soon.
{: .prompt-info }

## Bio

Will be updated.

## Key Achievements

- **Published CVE:** Will be updated.
- **Let Us Hack:** Will be updated.

## Creator Profile

| Rooms built | Topics |
| :---------- | :----- |
| {{ profile.creator.rooms_built }} | {{ profile.creator.topics | join: ', ' }} |

## Links

{% for link in profile.links %}
{%- if link.url -%}
- <i class="{{ link.icon }} fa-fw"></i> [{{ link.title }}]({{ link.url | relative_url }})
{% else -%}
- <i class="{{ link.icon }} fa-fw"></i> {{ link.title }} — will be updated
{% endif -%}
{% endfor %}

## Contact

Reach me by email at [{{ site.social.email }}](mailto:{{ site.social.email }}).
