6/2/2022 Update - As it turns out, running the containers on a Docker-Compose network causes issues with the network interface data gathered by Telegraf. I looked into a few possible solutions, but it seemed like the best option was to host everything in Bridge mode, and Telegraf in Host mode, in order to gather data properly. Config files and database connections need to be updated to use YOURSERVERIP instead of container names. The four main places this will happen is:
- Your telegraf.conf needs to update the [outputs.influx] URL field to match your server IP instead of 'http://influxdb:8086'
- Your varken.ini needs to update the [influxdb] url to be YOURSERVERIP
- Your Data Sources in grafana need to be updated to use your server IP, instead of container names (Telegraf, UnraidAPI, and Varken data sources)

If you want to clean your influx database as well, you can enter the container with the 'console' option, and run these commands:
- `influx`
- `use telegraf`
- `DROP SERIES FROM net WHERE ("host"!='YOURHOSTNAME')`
  - *Your hostname is the value you set in your environment variables*
- `drop series from net WHERE ("interface"!='br0' OR "interface"!='wg0' OR "interface"!='eth0' OR "interface"!='bond0' OR "interface"!='docker0' OR "interface"!='virbr0')`
-------------

# Ultimate Unraid Dashboard Docker-Compose

Community Developer 'falconexe' over at the unraid forums has built and maintained the amazing [Ultimate Unraid Dashboard](https://forums.unraid.net/topic/96895-ultimate-unraid-dashboard-uud/), which is a stack of applications to gather and display data for the server admin. It consists of:

- [InfluxDB](https://www.influxdata.com/) - A time series database for storing data
- [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) - An Agent for collections server metrics and storing them in influxdb
- [UnraidAPI](https://github.com/ElectricBrainUK/UnraidAPI) - A WIP open source api for the UnRaid operating System
- [Varken](https://github.com/Boerderij/Varken) - A plex agent for collection stats from common Plex apps and storing them in InfluxDB
- [Grafana](https://grafana.com/) - An incredible data visualization tool

I made the addition of [Chornograf](https://www.influxdata.com/time-series-platform/chronograf/) to the stack, which is the InfluxDB UI Explorer Replacement. As of 1.3, InfluxDB no longer supports its built in WebUI Database exploration tool, so this takes its place instead. Nice for debugging, but totally optional for most users. The real meat of the project is in the Grafana Dashboard - This tool is here for database administration.

Since this setup relies on so many different containers and configuration files, there have been attempts to 'ease' users into it with 'all-in-one' solutions (like testdasi's abandoned [grafana-unraid-stack](https://github.com/testdasi/grafana-unraid-stack) project), but it felt more realistic to use Docker-Compose for the job. 

## Prerequisites:
- A Functioning Unraid Operating System on 6.10.2
  - This requirement is based on the new 'labels' provided for docker compose, which lets us set icons and 'webui' parameters for compose-built containers
- Docker Compose Manager plugin
- Usable Ports:
  - 8086 (influxdb)
  - 7889 (default chronograf ui port)
  - 6379 (telegraf)
  - 3005 (default unraidapi ui port)
  - 3000 (default grafana ui port)

**Note**: The default ports can be changed, but it is highly recommended not to. They are hardcoded into the 'labels' section of related containers to add a webui button. If you are comfortable editing the dockercompose.yml file, and can map your configs to the proper ports, then go for it.

### Installation

- With the Docker-Compose-Manager plugin installed, scroll to the bottom of the 'Docker' tab and press 'Add New Stack'
- Enter a name, and hit okay
- Scroll back down, and Press the Gear next to the Stack you just created

### IMPORTANT NOTE: 
**These default environment variables are mapped in the docker compose to the webui labels. If you change a webui port here, you HAVE TO change it in the yml for the webui button to function properly. As far as I know, theres no way to parse docker-compose labels in the label string, otherwise it would be able to use the ports you set in the env**

- Under 'Edit env', copy the .env variables from this repository, and paste them into the textbox. Replace any of the variables with data according to your UnRaid Setup
  - **HOSTNAME** - The hostname of your server. Usually its 'Tower'
  - **TIMEZONE** - Your Timezone
  - **APPDATA_PATH** - The path to your appdata share
    - If you want to bypass the unraid fs layer, you can put '/mnt/YOURCACHEDISKNAME/appdata'
  - **UNRAID_API_UI_PORT** - The Port you will access your 'UnraidAPI' container through
  - **CHRONOGRAF_UI_PORT** - The port you will access your Chronograf instance from.
  - **GRAFANA_UI_PORT** = The Port you will access your 'Grafana' container through
  - **GF_SERVER_ROOT_URL** - The Grafana Server Root url. 
    - Usually this is your IP address, since you'll be accessing it via 'http://YOURIP:YOURPORT
  - **GF_SECURITY_ADMIN_PASSWORD** - Your Secret Admin Password for Grafana



- Scroll down, press the gear again, and hit 'Edit Stack'
- Copy and paste the 'Ultimate_Unraid_Dashboard.yml' contents into your text box. 
  - Since Docker-Compose uses its own network, only ports set in the .env will clash with your unraid docker ports. Other ports are seen by other containers in the network with the 'expose' setting 
- **This is not the end of your setup. Some containers require a Config file to be made, and will not start without them. Varken and Telegraf are the main two.**

### Config Files

#### Varken
- In your /mnt/user/appdata, create a folder called 'GUS-varken'
  - You can change this, but you'll need to edit the Docker-Compose stack if you do.
- Download the 'varken.ini' file in the /appdata/GUS-varken folder of this repository
- Set your INFLUXDB ip:port to your influxdb containers ip:port
- The Varken.ini file consists of individual 'blocks' for each service you configure with it. You will need to go into each instance of tautulli/sonarr/radarr/lidarr/sickchill/ombi in order to set up the information like URLs and API Keys for each individual block
  - Read carefully, because there's a few ways to format things like ips/ports
- Once all of your services you want to connect are configured in the varken.ini, place it into your appdata/GUS-varken folder on your server

#### Telegraf
*The .conf file was created by [skaterpunk](https://github.com/skaterpunk/UUD). All credit goes to him for trimming it down to what's required*

- In your /mnt/user/appdata, create a folder called 'GUS-telegraf'
- Download the 'telegraf.conf' file found in the /appdata/GUS-telegraf folder of this repository
- Most of the config should be done. Set your INFLUXDB ip:port to your influxdb containers ip:port
- You may need to change is the influx username/password, if you set them, but the Docker-Compose doesn't specify any.
- In the 'telegraf.conf, uncomment either the [[inputs.hddtemp]\] or the [[inputs.ipmi_sensor]\] line, depending on your method of gathering data. More often than not, this is not required, but if you run the HDDTemp container, you can connect it here.
  - If you use server hardware, it will likely be the ipmi line, and you'll need the plugin
  - If you use consumner hardware, hddtemp will likely suffice
- Move your telegraf.conf to the appdata/GUS-telegraf folder on your server

*Again, massive thanks to skaterpunk for doing the hard config work*

#### Grafana

##### If you already have set up the UUD and are transitioning to Docker-Compose there are a few things that need to be changed in the Grafana WebUI configuration. Your data sources are referred to by IP address,but you need to change them to refer by container names.

- Hit the 'Compose Up' button in the 'Docker' UnRaid tab, the 'Docker Compose Down', then 'Docker Compose Up' again
  - This is to ensure the Grafana Plugins are installed/recognized properly
- ~~The Actual Grafana Dashboard is found in the [official forum post](https://forums.unraid.net/topic/96895-ultimate-unraid-dashboard-uud/). Scroll down to the attachments in the main post, and download the 'Current' version.~~
  - As of 12/1/2022, the forum post outlining the UUD seems to have been taken down. Its unclear exactly why it was removed, but Ive added my Dashboard JSON to the repository.
  - Download the 'GUS-Dashboard.json' file and import it into Grafana under the '4 box' button on the left menu   
- Log into Grafana, and hit the 'Configuration' gear icon on the left side panel
- In the configuration, you will need to edit the Telegraf, UnraidAPI, and Varken data sources

##### Telegraf 
- Hit 'Add Data Source' and choose an 'InfluxDB' data type
- With the containers being on the same docker compose network, we can refer to them by container name in URLs.
- Top fields:
  - URL - http://YOURSERVERIP:8086
- Bottom Fields:
  - Database - telegraf
  - User - telegraf


##### UnraidAPI
- Hit 'Add Data Source' and choose an 'JSON API' data type
- With the containers being on the same docker compose network, we can refer to them by container name in URLs.
- Top fields:
  - URL - http://YOURSERVERIP:YOURUNRAIDAPIPORT/api/getServers
  
**Note**: The default unraidapiport is 3005. If you did not edit the docker-compose.yml, set the port to 3005

##### Varken
- Hit 'Add Data Source' and choose an 'InfluxDB' data type
- With the containers being on the same docker compose network, we can refer to them by container name in URLs.
- Top fields:
  - URL - http://YOURSERVERIP:8086
- Bottom Fields:
  - Database - telegraf
  - User - telegraf





If you've uploaded the dashboard found at falconexe's forum page, then you should be close to finished. You can close the configuration screen, and go to your dashboard. You will need to set the variables at the top of your dashboard to match the information related to your setup. 

Since there are a few final things you can do (like add in a location api to visualize where plex streams originate), I highly recommend checking final configs with the official setup instructions found in the Unraid Forums as I just wanted to get you up and running with docker-compose

With the removal of the original forum post, you can view those instructions here on Internet Archive's WayBackMachine: https://web.archive.org/web/20220616040234/https://forums.unraid.net/topic/96895-ultimate-unraid-dashboard-uud/