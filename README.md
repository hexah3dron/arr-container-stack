# arr-container-stack

A complete containerized stack of serverr applications, Plex streaming, and both Usenet and Torrent download functionality.

The applications included are:
- Radarr
- Sonarr
- Overseerr
- Prowlarr
- SABnzbd
- Qbittorrent
- Gluetun (VPN)
- Plex

## Requirements

- Docker version 24.10 and above
- Docker compose version v2.15 and above

Copy the `docker-compose.example.yml` file and rename it `docker-compose.override.yml`. This will ensure you still have the original file to reference, and allow you to make your modifications without losing the original configuration.

## Deploy Containers

There are two ways this stack can be deployed.

1. With a VPN using gluetun to deploy a VPN of your choice
2. Without a VPN (not recommended)

### Network Set Up

On your host machine, create the docker network that will be used by the entire application stack. 

```bash
docker network create --subnet 172.20.0.0/16 mynetwork
# use any subnet that does not conflict with existing docker networks on your host machine
```

**VPN Deployment Method**

If VPN is enabled, both qBittorrent and Prowlarr will be put behind VPN.

In the `docker-compose.yml` file under services > vpn > environment, you will find a VPN_SERVICE_PROVIDER variable. This can be modified to enable use of ExpressVPN, SurfShark, NordVPN, Custom OpenVPN or Wireguard VPN. It uses OpenVPN type for all providers, and can be configured to use a large number of VPN systems. 

Follow https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers to find the gluetun settings needed for your particular VPN provider.

By default, VPN is enabled in `docker-compose.yml`. The `docker-compose.yml` file has comments describing how to configure with or without the VPN deployment. Comment and uncomment as described throughout the *entire* compose file.

To deploy the stack with VPN, use the following command:

```bash
docker compose --profile vpn up -d

# docker compose -f docker-compose-nginx.yml up -d # OPTIONAL to use Nginx as reverse proxy
```

*Static container IP addresses are needed when Prowlarr is behind VPN. Prowlarr is configured to accesss both Radarr and Sonarr via IP, and will help prevent the need to reconfigure settings when containers restart.*

*This is set using RADARR_STATIC_CONTAINER_IP and SONARR_STATIC_CONTAINER_IP variables in your env file.*

**Without VPN Deployment Method**

To deploy the stack without VPN (highly discouraged if using torrents), run the following command:

```bash
docker compose up -d
# docker compose -f docker-compose-nginx.yml up -d # OPTIONAL to use Nginx as reverse proxy
```

## Configure qBittorrent

- Open qBitTorrent at http://\<*server-ip*>:5080. Default username is `admin`. The temporary password can be collected from the container logs via the command `docker logs qbittorrent` from the shell.
- On the webpage, navigate to  Tools > Options > WebUI > Change password and modify to your own unique password.
- To build your download directories, run the following commands:

```bash
docker exec -it qbittorrent bash # Get inside qBittorrent container

# Once inside the terminal, run the following to build the downloads directories. Modify/Add to your needs
mkdir /downloads/movies /downloads/tvshows
chown 1000:1000 /downloads/movies /downloads/tvshows
```
## Configure Prowlarr

- Before configuring Prowlarr, log into your Radarr and Sonarr instances (detailed in the [Configure Radarr](#configure-radarr) and [Configure Sonarr](#configure-sonarr) sections) and grab the API keys from the Settings > General API Key sections and note them. They will be used to set up the connection between Prowlarr, Sonarr, and Radarr.
- Open Prowlarr at http://\<*server-ip*>:9696
- Under Settings > General, select "Show Advanced". Navigate to the Security section and set Authentication method. Set Authentication Required to Enabled and add your unique username and password.
- On the Idexer tab, select Add Indexer. Search for your Indexers and fill out the required authentication info. Test and save for all indexers.
- Under Settings > Apps, select the `+` and set up Radarr and Sonarr. 
- For Radarr settings, set the Prowlarr server as http://vpn:9696 if configured for VPN, or http://\<*server-ip*>:9696 if configured without. Set the Radarr server as http://\<*RADARR_STATIC_CONTAINER_IP*>:7878. **Note: This is the same IP you set in the env file**.
- Set the Prowlarr server as http://vpn:9696 if configured for VPN, or http://\<*server-ip*>:9696 if configured without. Set the Sonarr server as http://\<*SONARR_STATIC_CONTAINER_IP*>:7878. **Note: This is the same IP you set in the env file**.
- This will automatically add indexers in the respective apps automatically.

**Note: If VPN is enabled, Prowlarr will not be able to reach Radarr and Sonarr with \<*server-ip*> or the container service name. Prowlar will also not be reachable with its container service name. Use `http://vpn:9696` instead in the Prowlarr server field.**

## Configure Radarr

- Open Radarr at http://\<*server-ip*>:7878
- Under Settings > Media Management enable `Unmonitor Deleted Movies` if you want to prevent Radarr from auto searching for the same media immediately after deletion. You will need to re-monitor it again if you want Radarr to search for it.
- Add the Root folder, and set to /data/media/movies
- Under Settings > Download clients, select qBittorrent. Configure the Name, Host, Port, Username and Password to match your configuration and save. **Note: If VPN is enabled, qbittorrent will be reachable inside the vpn service network. In this case use `vpn` in Host field.**
- Under Settings > General, select "Show Advanced". Navigate to the Security section and set Authentication method. Set Authentication Required to Enabled and add your unique username and password.

**Adding a movie**

- Under Movies, Select Add New and search for a movie. When selected, ensure root folder is set to /data/media/movies if following guide exactly. Select your Quality profile and ensure `Start searching for missing movie` is enabled, then hit `Add Movie`.
- All queued movies download can be checked under Activities > Queue 
- If the movie is found as a torrent, go to your qBittorrent instance and confirm the torrent is in the queue. 

## Configure Sonarr

- Open Sonarr at http://\<*server-ip*>:8989
- Under Settings > Media Management enable `Unmonitor Deleted Episodes` if you want to prevent Sonarr from auto searching for the same media immediately after deletion. You will need to re-monitor it again if you want Sonarr to search for it.
- Add the Root folder, and set to /data/media/tv
- Under Settings > Download clients, select qBittorrent. Configure the Name, Host, Port, Username and Password to match your configuration and save. **Note: If VPN is enabled, qbittorrent will be reachable inside the vpn service network via container name. In this case use `vpn` in Host field.**
- Under Settings > General, select "Show Advanced". Navigate to the Security section and set Authentication method. Set Authentication Required to Enabled and add your unique username and password.

**Add a TV show**

- Under Series, Select Add New and search for a TV Series. When selected, ensure root folder is set to /data/media/tv if following guide exactly. Select your Quality profile and ensure `Start searching for missing episodes` is enabled, then hit `Add`.
- All queued TV Series downloads can be checked under Activities > Queue 
- If the series is found as a torrent, go to your qBittorrent instance and confirm the torrent is in the queue. 

## Configure SABnzbd

- Connect to SABnzbd at http://\<*server-ip*>:8182
- To setup your Usenet providers, navigate to Servers and select Add Server. Follow the simple authentication setup according to your own Usenet providers and hit `Add Server` after testing.
- To setup your download folders, navigate to the Folders tab. Set the temporary download folder to `/config/Downloads/incomplete`, and the completed download folder to `/data/usenet/complete`. You can modify the location of the incomplete downloads to meet your particular system needs in terms of r/w capabilities, but the completed download folder should always exist in the `/data/usenet/complete`. The main idea here is that SABnzbd is used as a download client for Sonarr and Radarr requests, and when it has finished it will alert either Sonarr or Radarr that it is complete, where they will then sort your media into the correct directories to be picked up by Plex.
- You will now set up the download Categories, which are used by SAB to sort incomplete and complete downloads automatically, tag media sort by priority, as well as for Sonarr and Radarr to pick up the correct media type.
- For TV, create a new Category named `tv`, and set Folder/Path to `/data/usenet/complete/tv`
- For Movies, create a new Category named `movies`, and set Folder/Path to `/data/usenet/complete/movies`



## Configure Plex

- Connect to Plex at http://\<*server-ip*>:32400 and follow the guided setup.
- If you run into a **Not Authorized** error, ensure the account you are using to set up the Plex instance has the generate Plex Claim Token in the .env file. If you run into further issues on a QNAP system, see the [following guide](https://www.reddit.com/r/PleX/comments/pjtl5m/qnap_cant_launch_setup_not_authorized/).
- Add your media via Add Library, and set the library location to the **final** directories for both Sonarr and Radarr. If you are following the guide, these should be in your `/path/to/appdata/media-root` directory.
**Note: If you are intending to use hardware for transcoding, check the `docker-compose.example.yml` file for the services > plex > devices section and ensure your unique GPU is /dev is added**


## Configure Overseerr

- Now that Sonarr and Radarr are set up, you can finally hook Overseer in to manage downloads of both types of media from one location
- Connect to Overseerr at http://\<*server-ip*>:8182
- Login using the same Plex account you used for authorization.
- Configure your Plex server settings as indicated in the [Configure Plex](#configure-plex) section above.
- Add your Plex media libraries and allow them to sync with Overseerr.
- For Radarr setup, set Hostname to `http://radarr:7878` and enter the Radarr API key collected earlier. Set your Root folder as `/data/media/movies` and ensure the `movies` tag is being applied. Configure the rest of the settings as you see fit, and save.
- For Sonarr setup, set Hostname to `http://sonarr:8989` and enter the Sonarr API key collected earlier. Set your Root folder as `/data/media/tv` and ensure the `tv` tag is being applied. Configure the rest of the settings as you see fit, and save.


### Congratulations! You have completed the intial ARR stack setup!

## Application Set Up
The complete configuration of the various systems is outside of the scope of this guide, but [here](https://trash-guides.info/Hardlinks/Examples/) is a fantastic guide explaining various things like hardlinking and how to properly sync the arr container data.
