# GoSungrow2MQTT

The current GoSungrow API/addon, created by [MickMake](https://github.com/MickMake/GoSungrow), is no longer being maintained for some reason and has several connection issues. While there are some patches available, such as the one by [triamazikamno](https://github.com/triamazikamno/GoSungrow), they don’t fully address the problem. Specifically, the addon still generates multiple unnecessary entities, complicating the selection of data to be returned and causing frequent connection flooding errors.

In response to this, I decided to develop a simple script that encapsulates the GoSungrow API along with a Node.js app running in Docker. This script periodically executes user-defined GoSungrow endpoints and sends the results filtered to an MQTT broker. This way, in Home Assistant, we can create sensors that subscribe to these topics and receive only the information that truly matters.

You can run this container directly on your Home Assistant setup or locally.

## Prerequisites

- MQTT Broker
- Docker (if used locally)

## How to use

1. Clone this repository:
```bash
git clone https://github.com/gR-xbY/GoSungrow-MQTT-Bridge.git
cd GoSungrow-MQTT-Bridge
```

2. Configure environment variables in the `.env` file:
   
Edit the `.env` file and set the necessary environment variables:

```bash
cp .env.example .env
```

```
# MQTT
MQTT_BROKER=mqtt://<your_broker_ip>:<your_broker_port>
MQTT_USERNAME=your_mqtt_username
MQTT_PASSWORD=your_mqtt_password
MQTT_TOPIC=your_mqtt_topic
MQTT_CLIENT_ID=your_mqtt_client_id

# SUNGROW
SUNGROW_PS_ID=your_sungrow_ps_id (log in to the old sungrow dashboard and search for the query param psId)

# APP (Change for whatever you want)
FETCH_INTERVAL=900000
CONNECT_TIMEOUT=10000
RECONNECT_PERIOD=60000
## Here you can define the interval at which the API calls will be performed, usually during the day to avoid entering repeated values.
APP_START_HOUR=5
APP_END_HOUR=20
```

3. Configure GoSungrow variables in the `GoSungrow/config.json` file:
   
Edit the `config.json` file and set the necessary GoSungrow variables:

```bash
cp GoSungrow/config.example.json GoSungrow/config.json
```
Here you basically need to define the `host`, `appkey`, `user`, and `password`.

For `host`, select based on your account location:
"Server"      | Web URL                                                  | Host URL
--------------|----------------------------------------------------------|--------------
Chinese       | [www.isolarcloud.com.cn](https://www.isolarcloud.com.cn) | https://gateway.isolarcloud.com.cn
European      | [www.isolarcloud.eu](https://www.isolarcloud.eu)         | https://gateway.isolarcloud.eu
International | [www.isolarcloud.com.hk](https://www.isolarcloud.com.hk) | https://gateway.isolarcloud.com.hk
Australian    | [au.isolarcloud.com](https://au.isolarcloud.com)         | https://augateway.isolarcloud.com

For `appkey`, try one of these:
* `B0455FBE7AA0328DB57B59AA729F05D8`
* `ANDROIDE13EC118BD7892FE7AB5A3F20`

If you have any questions, please consult the references at the end of this document.

5. Setup GoSungrow endpoints call in the `api.json` file:

### Example Usage
```json
[
  {
    "endpoint": "getPowerStatistics",
    "topic": "inverter1/powerStatistics"
  },
  {
    "endpoint": "getPsDetail",
    "filter": ["actual_energy", "day_eq_hours", "month_energy_virgin"],
    "topic": "inverter1/psDetail",
    "params": {"ps_id": "your_custom_psId_1"}
  },
  {
    "endpoint": "getPowerStatistics",
    "filter": ["dayPower", "nowCapacity"],
    "topic": "inverter2/powerStatistics",
    "params": {"ps_id": "your_custom_psId_2"}
  },
  {
    "endpoint": "queryMutiPointDataList ",
    "topic": "inverter2/mutiPointDataList",
    "params": {
                "ps_key":"1129147_14_1_1,1129147_14_1_1,1129147_14_1_1,1129147_14_1_1,1129147_14_1_1,1129147_14_1_1,1129147_14_1_1,1129147_14_1_1,1129147_14_1_1,1129147_11_0_0",
                "points":"p13150,p13126,p13142,p13143,p13019,p13141,p13121,p13003,p13149,p83106",
                "minute_interval":"5",
                "start_time_stamp":"20220215000000",
                "end_time_stamp":"20220215235900",
                "ps_id":"your_custom_psId_2"
              }
  }
]
```

- **endpoint**: Specifies the GoSungrow API endpoint to be called. Take a look at the GoSungrow API documentation for more information.
- **filter**: Defines the specific data fields to be included in the MQTT message. If left blank, all data fields will be returned.
- **topic**: Specifies the MQTT topic where the data will be published. The final topic will be a sufix from MQTT_TOPIC defined on `.env`: `MQTT_TOPIC/topic`
- **params**: Specifies custom params that will bee appended to api call. If left blank, the api will use by default SUNGROW_PS_ID.

6. Build and run the Docker image:
   
```bash
docker build -t gosungrow2mqtt .
docker run -d --name gosungrow2mqtt gosungrow2mqtt
```

The application will start and begin consuming data from the GoSungrow API and publishing it to the Home Assistant MQTT in a loop with the time defined in `.env`.

## Configuring New Sensor on HA

After successfully sending the data to your broker, you can set up a new sensor in your Home Assistant by editing the `configuration.yaml` file. Simply add an entry like the one below and modify it according to your needs.

```yaml
mqtt:
  sensor:
    - name: "Daily Generation"
      unique_id: "sungrow_daily_generation"
      state_topic: "sungrow/inverter/powerStatistics"
      unit_of_measurement: "kWp"
      value_template: "{{ value_json.dayPower.value | float }}"
```

## How it works

The Node.js application inside the Docker will make requests to the GoSungrow API using the provided credentials and publish the data to MQTT. The `index.js` script contains the logic for making requests and publishing messages to MQTT.

## References
- [Great @Paraphraser Gist for common doubts](https://gist.github.com/Paraphraser/cad3b0aa6428c58ee87bc835ac12ed37)
- [MickMake GoSungrow API](https://github.com/MickMake/GoSungrow)
