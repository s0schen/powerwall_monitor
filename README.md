# Powerwall_Monitor
Monitoring Dashboard for the Tesla Powerwall using Grafana, InfluxDB and Telegraf.

![Dashboard](https://user-images.githubusercontent.com/836718/144769680-78b8abf4-4336-4672-9483-896b0476ec44.png)
![Strings](https://user-images.githubusercontent.com/836718/146310511-7863e4bb-7e43-40b9-9790-65c1d6ce24ba.png)

This is based on the great work by [mihailescu2m](https://github.com/mihailescu2m/powerwall_monitor) but has been modified to use pypowerwall as a proxy to the Powerwall and includes solar String graphs for Powerwall+ systems.

## Requirements
* docker
* docker-compose

## Installation

Clone this repo to your local host that will run the dashboard:

```bash
git clone https://github.com/jasonacox/powerwall_monitor.git
```

You will want to set your local timezone by editing `powerwall.yml`, `influxdb.sql` and `dashboard.json` or you can use this handy `tz.sh` update script.  A list of timezones is available [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

```bash
# Replace with your timezone
bash tz.sh "America/Los_Angeles"
```

### Docker Containers

* Edit `powerwall.yml` and look for the section under `pypowerall` and update the following details for your Powerwall:
```yml
            PW_PASSWORD: "password"
            PW_EMAIL: "email@example.com"
            PW_HOST: "192.168.91.1"
            PW_TIMEZONE: "America/Los_Angeles"
			PW_DEBUG: "yes"

```

* Start the docker containers

```bash
	docker-compose -f powerwall.yml up -d
```

### InfluxDB

* Connect to the Influx database shell: `docker exec -it influxdb influx`
* At the database prompt, you will need to enter (copy/paste) the following commands after you adjust the timezone (tz) as appropriate.  If you are on a Mac you can `pbcopy < influxdb.sql` to grab the edited version into your clipboard for pasting or you can grab the commands below (also in [influxdb.sql](influxdb.sql)):
	```sql
	USE powerwall
	CREATE RETENTION POLICY raw ON powerwall duration 3d replication 1
	ALTER RETENTION POLICY autogen ON powerwall duration 365d
	CREATE RETENTION POLICY kwh ON powerwall duration INF replication 1
	CREATE RETENTION POLICY daily ON powerwall duration INF replication 1
	CREATE RETENTION POLICY monthly ON powerwall duration INF replication 1
	CREATE CONTINUOUS QUERY cq_autogen ON powerwall BEGIN SELECT mean(home) AS home, mean(solar) AS solar, mean(from_pw) AS from_pw, mean(to_pw) AS to_pw, mean(from_grid) AS from_grid, mean(to_grid) AS to_grid, last(percentage) AS percentage INTO powerwall.autogen.:MEASUREMENT FROM (SELECT load_instant_power AS home, solar_instant_power AS solar, abs((1+battery_instant_power/abs(battery_instant_power))*battery_instant_power/2) AS from_pw, abs((1-battery_instant_power/abs(battery_instant_power))*battery_instant_power/2) AS to_pw, abs((1+site_instant_power/abs(site_instant_power))*site_instant_power/2) AS from_grid, abs((1-site_instant_power/abs(site_instant_power))*site_instant_power/2) AS to_grid, percentage FROM raw.http) GROUP BY time(1m), month, year fill(linear) END
	CREATE CONTINUOUS QUERY cq_kwh ON powerwall RESAMPLE EVERY 1m BEGIN SELECT integral(home)/1000/3600 AS home, integral(solar)/1000/3600 AS solar, integral(from_pw)/1000/3600 AS from_pw, integral(to_pw)/1000/3600 AS to_pw, integral(from_grid)/1000/3600 AS from_grid, integral(to_grid)/1000/3600 AS to_grid INTO powerwall.kwh.:MEASUREMENT FROM autogen.http GROUP BY time(1h), month, year tz('America/Los_Angeles') END
	CREATE CONTINUOUS QUERY cq_daily ON powerwall RESAMPLE EVERY 1h BEGIN SELECT sum(home) AS home, sum(solar) AS solar, sum(from_pw) AS from_pw, sum(to_pw) AS to_pw, sum(from_grid) AS from_grid, sum(to_grid) AS to_grid INTO powerwall.daily.:MEASUREMENT FROM powerwall.kwh.http GROUP BY time(1d), month, year tz('America/Los_Angeles') END 
	CREATE CONTINUOUS QUERY cq_monthly ON powerwall RESAMPLE EVERY 1h BEGIN SELECT sum(home) AS home, sum(solar) AS solar, sum(from_pw) AS from_pw, sum(to_pw) AS to_pw, sum(from_grid) AS from_grid, sum(to_grid) AS to_grid INTO powerwall.monthly.:MEASUREMENT FROM powerwall.daily.http GROUP BY time(365d), month, year END
	```

Note: the database queries are set to use `America/Los_Angeles` as timezone. Use the `tz.sh` script or manually update the database commands above to replace `America/Los_Angeles` with your own timezone.

### Grafana Setup

* Open up Grafana in a browser at `http://<server ip>:9000` and login with `admin/admin`

* From `Configuration\Data Sources` add `InfluxDB` database with:
  - Name: `InfluxDB`
  - URL: `http://influxdb:8086`
  - Database: `powerwall`
  - Min time interval: `5s`
  - Click "Save & test" button

* From `Configuration\Data Sources` add `Sun and Moon` database with:
  - Name: `Sun and Moon`
  - Enter your latitude and longitude (some browsers will use your location)
  - Click "Save & test" button

* From `Dashboard\Manage` (or `Dashboard\Browse`), select `Import`, and upload `dashboard.json`

### Notes

* The database queries and dashboard are set to use `America/Los_Angeles` as the timezone. Remember to edit the database commands [influxdb.sql](influxdb.sql), [powerwall.yml](powerwall.yml), and [dashboard.json](dashboard.json) to replace `America/Los_Angeles` with your own timezone.

* InfluxDB does not run reliably on older models of Raspberry Pi, resulting in the Docker container terminating with `error 139`.  

### Troubleshooting Tips

Check the logs of the services using:
```bash
	docker logs -f pypowerwall
	docker logs -f telegraf
	docker logs -f influxdb
	docker logs -f grafana
```

### Credits

* This is based on the great work by mihailescu2m at [https://github.com/mihailescu2m/powerwall_monitor](https://github.com/mihailescu2m/powerwall_monitor).

