# Overview

This repository explains how to install, configure and run OSRM (open source routing machine) for
use as a **map matching** server. This system takes raw, noisy GPS readings, and snaps them to the
nearest road. Without this functionality, some vehicles might appear to move off their designated
routes. This image illustrates the concept:

![Route-matching illustrated](images/raw-matched.png "Example of route matching")


## Running OSRM with Docker
The cartographic data must be prepared in advance, [according to these instructions](https://github.com/Project-OSRM/osrm-backend)

For convenience, an abridged version is given, operating under the assumption that we're running it
in `/home/gps/osrm` under the user `gps`:

0. `mkdir ~/osrm  && cd ~/osrm`
1. Download the latest map for Moldova from http://download.geofabrik.de/europe/moldova.html,
   typically it is a matter of `wget http://download.geofabrik.de/europe/moldova-latest.osm.pbf`
2. Trim down the map (see next section; this optional, but strongly recommended)
3. Pre-process the data `docker run -t -v "${PWD}:/data" --rm osrm/osrm-backend osrm-extract -p /opt/car.lua /data/moldova-latest.osm.pbf`
4. `docker run -t -v "${PWD}:/data" --rm osrm/osrm-backend osrm-partition /data/moldova-latest.osrm`
5. `docker run -t -v "${PWD}:/data" --rm osrm/osrm-backend osrm-customize /data/moldova-latest.osrm`
6. After running these commands, make sure all the files are owned by the user `gps`, run `chown gps:gps ./*.*`


### Trim down the map
The map provided by Geofabrik includes all the roads in the entire country, even the ones in other
cities, and the ones we know fore sure are not a part of the pubic transport network. This means
that whenever OSRM performs a match, it will have more work to do, because the search-space is
larger. It also implies that sometimes, when the coordinates are particularily noisy, OSRM might
"snap" them to the wrong road.

Thus, by removing unnecessary roads, we make map matching more efficient. The raw `moldova-latest.osm.pbf`
must be processed, such that it only includes the public transport grids that meet the following
criteria:
- are a part of Chișinău, Ialoveni, Trușeni, Stăuceni or Codru (this list is to be extended, when
RTEC adds new routes, or when other transport systems join Roataway)
- are a trolleybus or a bus route (i.e. we omit minibuses)


This is how it is done (the easy way):

1. Install `osmium`. on Debian systems it is `apt install osmium-tool`
2. Run `osmium getid -r moldova-latest.osm.pbf r6726484 -o chisinau-public-transport.osm.obf`
   (`6726484` is a relation that includes all the data of all public transport networks in
   Chisinau)
3. Alternatively, if you want to include trolleybus routes and nothing else, use this command:
   `osmium getid -r moldova-latest.osm.pbf r7390177 r8649765 r9478330 r9478295 r9478243 r9220092 r8219936 r6768372 r6768261 r6768260 r6768231 r6768200 r6768158 r6768073 r6768055 r6767907 r6767859 r6755788 r9220024 r6862391 r6755732 r6754654 r6754630 r6754620 r6754537 r6754503 r6754451 r6754453 r6768282 -o trolley-only.osm.pbf` (note that the magic numbers are the relation
   IDs of each route, you can find them in `routes.csv` inside the `infrastructure-data` repo)
3. Use the output file as an input for the previous section.

This is an alternative way to accomplish the same thing. It takes more steps, but it is a
point-and-click approach that some might prefer:

1. Go to https://overpass-turbo.eu/
2. Type the following query: `relation (6726484); >>; (._;>;); out meta;` (the relation ID corresponds
   to a relation that includes all public transport in Chisinau and adjacent cities) and press `Run`
3. The system will warn you that the result might be a large data set, press `continue anyway`
4. When the query is complete, you'll see all the roads on the map, you can visually inspect them
   to make sure everything is alright, then press `Export`.
5. Choose `download as raw OSM data` and save it to a local directory.
6. Open this file in `josm`
7. Then go to `File\Save as`, choose the `osm pbf` format, and save it.
8. Use this file as the input data for the pre-processing steps described above, it might be convenient
   to save it as `moldova-latest.osm.pbf` so all the commands in the previous section will work "as is".


### Check if all is well

1. Run the server itself `docker run -t -i --rm -p 5000:5000 -v "${PWD}:/data" osrm/osrm-backend osrm-routed --algorithm mld /data/moldova-latest.osrm` (note that the `--rm` flag ensures the container will be deleted after use; no worries - a new one will be created if you start `osrm` again via docker)
2. Send a test request `curl 'http://127.0.0.1:5000/match/v1/driving/28.8260977,47.0231816;28.8260977,47.0231816?radiuses=15;15'`, you should get a JSON response back.

### Production deployment

This is done with `systemd`, see `res/systemd-osrm.template` for details. Pay attention to several
sneaky differences between what is in the service template file and the commands above:

- absolute paths are used, so no `${PWD}` in the service
- no `-i` either, because the service doesn't provide an interactive console

### Controlling the service
- `sudo systemctl osrm start|stop|restart`
- `sudo journalctl -u osrm -f`

