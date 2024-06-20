---
layout: post
title: "Home Assistant: Fetching live energy tariff from electricity provider"
---
In my previous post I explained how I created a Home Assistant entity to track the daytime/nighttime costs of my electricity. However, that is only half the story. Let's back up to where this started.

I got into home automation a few years ago - I have a few Google Home devices, the IKEA TRÅDFRI gateway and a bunch of smart lightbulbs. The integration between all these was pretty good, so I didn't think I needed another system so I stayed away from Home Assistant - until I finally caved and gave it a try. I set it up on a dedicated raspberry Pi 4, and needless to say it is amazing. Not only does it have integrations for pretty much everything, but once I also added a [zigbee dongle](https://www.home-assistant.io/connectzbt1), this unlocked a bunch of workflows that were not available through the existing IKEA or Google Assistent apps.

I started being interested in monitoring my electricity once I discovered that my apartment's [electricity meter](https://www.nackaenergi.se/privat/elmatare) has a serial port that spits out live data called a P1/HAN port. Apparently all electricity meters in the nordic countries need to have it. Sweet! After months of deliberations I decided to buy the [official P1 to MQTT](https://energyintelligence.se/shop/product/p1-till-mqtt-adapter) adapter with WiFi support.
Connecting it to Home Assistant was straightforward - even though I did have to create an extra helper for the incoming data.

In any case, now that I had my electricity consumption in the dashboard, obviously I also wanted to track the costs.
As I mentioned in my previous post, my electricity supplier, Nacka Energi, has a fixed night/day rate, that was easy enough to hardcode into an entity.
However, my electricity "provider", GodEl has a variable electricity cost on top of the fixed rate.

Fun fact: GodEl is a great electricity company that donates all their profits to charity. Nice 👍 

In any case, the variable electricity rate doesn't seem to be available on any public endpoint, but I can view it in the GodEl app.
It was an easy few steps to fire up [HTTPToolkit](https://httptoolkit.com/), intercept all network requests, and see what's going on. It was easy enough to put that into a [python script](https://www.home-assistant.io/integrations/python_script/), and try to run it in HA. To my surprise, importing `requests` failed, and it took quite a bit of googling to figure out that what I needed was the [AppDaemon](https://github.com/hassio-addons/addon-appdaemon) integration, and then I could run the script in an isolated environment.

In any case, here's my script.

<details> <summary>Show me the Python code!</summary> 

<div markdown="1">

```python
import requests
from datetime import datetime, timezone, timedelta
import json
import os
import appdaemon.plugins.hass.hassapi as hass

class SpotPrice(hass.Hass):
	def initialize(self):
		self.run_every(self.get_spot_price, "now", 900)  # Run every 15 minutes

	def read_bearer_token(self, file_path):
		with open(file_path, 'r') as file:
			return file.read().strip()

	def read_json_data(self, file_path):
		if os.path.exists(file_path):
			with open(file_path, 'r') as file:
				return json.load(file)
		return None

	def save_json_data(self, file_path, data):
		with open(file_path, 'w') as file:
			json.dump(data, file, indent=4)

	def find_current_spot_price(self, spot_prices):
		current_time = datetime.now(timezone.utc)
		current_time_rounded = current_time.replace(minute=0, second=0, microsecond=0)
		for price in spot_prices:
			price_time = datetime.fromisoformat(price['date'][:-1]).replace(tzinfo=timezone.utc)
			if current_time_rounded == price_time:
				return price['value']
		return None

	def is_file_newer_than_one_hour(self, file_path):
		if os.path.exists(file_path):
			file_mod_time = datetime.fromtimestamp(os.path.getmtime(file_path), timezone.utc)
			return datetime.now(timezone.utc) - file_mod_time < timedelta(hours=1)
		return False

	def get_current_day_times(self):
		current_time = datetime.now(timezone.utc)
		start_time = current_time.replace(hour=0, minute=0, second=0, microsecond=0).isoformat()
		end_time = (current_time + timedelta(days=1)).replace(hour=0, minute=0, second=0, microsecond=0).isoformat()
		return start_time, end_time

	def get_spot_price(self, kwargs):
		bearer_token = self.read_bearer_token('/config/bearer_token.txt')
		json_file_path = '/config/spot_prices.json'
		current_time = datetime.now(timezone.utc)

		if self.is_file_newer_than_one_hour(json_file_path):
			data = self.read_json_data(json_file_path)
			if data is not None:
				spot_prices = data['data']['spotPrices']
				current_spot_price = self.find_current_spot_price(spot_prices)
				if current_spot_price is not None:
					self.set_state('sensor.spot_price', state=current_spot_price, attributes={
						'unit_of_measurement': 'SEK/kWh',
						'friendly_name': 'Spot Price',
						'value_with_vat': current_spot_price * 1.25
					})
					return

		start_time, end_time = self.get_current_day_times()
		url = "https://godel.production.getbright.se/v3/data/spotprices/delivery-area/SE3"
		params = {
			"startTime": start_time,
			"endTime": end_time,
			"interval": "hour"
		}
		headers = {
			"accept": "application/json",
			"Accept-Encoding": "gzip",
			"authorization": f"Bearer {bearer_token}",
			"Connection": "Keep-Alive",
			"content-type": "application/json",
			"Host": "godel.production.getbright.se",
			"User-Agent": "okhttp/4.9.2"
		}

		response = requests.get(url, headers=headers, params=params)
		if response.status_code == 200:
			data = response.json()
			self.save_json_data(json_file_path, data)
			spot_prices = data['data']['spotPrices']
			current_spot_price = self.find_current_spot_price(spot_prices)
			if current_spot_price is not None:
				self.set_state('sensor.spot_price', state=current_spot_price, attributes={
					'unit_of_measurement': 'SEK/kWh',
					'friendly_name': 'Spot Price',
					'value_with_vat': current_spot_price * 1.25
				})
			else:
				self.set_state('sensor.spot_price', state='unavailable', attributes={
					'unit_of_measurement': 'SEK/kWh',
					'friendly_name': 'Spot Price'
				})
		else:
			self.log(f"Request failed with status code {response.status_code}")
```

</div>
</details>
<br/>

The app also has a reauthentication workflow, but so far the script keeps working so I didn't need to worry about that until now. I'll try to fix it once it starts failing 🙂.

Here's the YAML for my new helper entity that includes both the fixed and spot price.

```yaml
{% raw %}
{% set current_time = now().time() %}
{% set start_time = now().replace(hour=6, minute=0, second=0, microsecond=0).time() %}
{% set end_time = now().replace(hour=22, minute=0, second=0, microsecond=0).time() %}
{% set spot_price = states('sensor.spot_price') | float %}
{% set daytime_price = 0.6550 %}
{% set nighttime_price = 0.4912 %}
{% if start_time <= current_time < end_time %}
  {{ spot_price + daytime_price }}
{% else %}
  {{ spot_price + nighttime_price }}
{% endif %}
{% endraw %}
```

- Unit of measurement: SEK/kWh
- Device class: Balance
- State class: Total

So here's the final chart of the electricity cost in my area. I find it quite useful for deciding when's the best time to run the dishwasher or even just to see how it affects the overall cost of my electric bill.

<img src="https://lh3.googleusercontent.com/pw/AP1GczOkbSA0_HXgfryp-35pAJvezTWTKL7xS1fb7gYmUeyvm9o5zwB5bEK0qW1LYSOxzG1_llRJnLbqEdyO7PkpJ75LxcUtosCeU54_mdl4VN0BC1W24UkvOFTUFR1irTXluehTRLkws6HyIdwG6DRLsedPdQ=w3102-h902-s-no"/>


