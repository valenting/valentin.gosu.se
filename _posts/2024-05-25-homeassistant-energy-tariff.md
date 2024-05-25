---
layout: post
title: "Home Assistant: adding the electricity tariff/cost to your energy usage dashboard"
---
The energy dashboard in Home Assistant supports adding tracking the cost of your electricity consumption. However, being a HA noob myself, it's been slightly tricky figuring out how to do that, so here's how I did it, in case someone else is also struggling with this.
In the `Energy` dashboard, you edit your `Grid consumption` entity (I'm assuming you've already set it up).
Under `Select how Home Assistant should keep track of the costs of the consumed energy.` you choose `Use an entity with current price`.
Now, you just have to define it.

- Go to your helpers page - mine is at http://homeassistant:8123/config/helpers
- Create Helper > Template > Template a sensor
- Give it a name: `Current electricity cost`
- Unit of measurement: mine is `SEK/kWh`, but it could also be `€/kWh` or `$/kWh`
- Device class: `Balance`
- State class: `Total`
- State template:

```yaml
{% raw %}
{% set current_time = now().time() %}
{% set start_time = now().replace(hour=6, minute=0, second=0, microsecond=0).time() %}
{% set end_time = now().replace(hour=22, minute=0, second=0, microsecond=0).time() %}
{% if start_time <= current_time < end_time %}
  {{ 0.6550 }}
{% else %}
  {{ 0.4912 }}
{% endif %}
{% endraw %}
```

This template specifies the cost per kWh. Mine has two levels, depending on the hour.
If you have a fixed cost, your could also look something like {% raw %} `{{ 0.1 }}` {% endraw %}.

And now go back to the Energy dashboard and use the new entity.

Tested on:
- Home Assistant Core 2024.5.5
- Home Assistant Operating System 12.3