# QuakeJS

QuakeJS is a port of [ioquake3](http://www.ioquake3.org) to JavaScript with the help of [Emscripten](http://github.com/kripken/emscripten).

To see a live demo, check out [http://www.quakejs.com](http://www.quakejs.com).

This project is a fork of [https://github.com/inolen/quakejs/](https://github.com/inolen/quakejs/) and makes QuakeJS fully locally hostable with your own QuakeJS server, local Play page, and local content server.


## Building binaries

As a prerequisite, you'll need to have a working build of [Emscripten](http://github.com/kripken/emscripten), then:

```shell
cd quakejs/ioq3
make PLATFORM=js EMSCRIPTEN=<path_to_emscripten>
```

Binaries will be placed in `ioq3/build/release-js-js/`.

To note, if you're trying to run a dedicated server, the most up to date binaries are already included in the `build` directory of this repository.


## Running locally

Install the required node.js modules:

```shell
npm install
```

Set `content.quakejs.com` as the content server:

```shell
echo '{ "content": "content.quakejs.com" }' > bin/web.json
```

Note: you can _optionally_ also specify a master server:

```shell
cat <<EOF > bin/web.json
{
  "content": "content.quakejs.com",
  "master": "master.quakejs.com"
}
EOF
```

Run the server:

```shell
node bin/web.js --config ./web.json
```

Your server is now running on: [http://0.0.0.0:8080](http://0.0.0.0:8080)


## Running a dedicated server

If you'd like to run a dedicated server, the only snag is that unlike regular Quake 3, you'll need to double check the content server to make sure it supports the mod / maps you want your server to run (which you can deduce from the [public manifest](http://content.quakejs.com/assets/manifest.json)).

Also, networking in QuakeJS is done through WebSockets, which unfortunately means that native builds and web builds currently can't interact with eachother.

Otherwise, running a dedicated server is similar to running a dedicated native server command-line wise.

Setup a config for the mod you'd like to run, and startup the server with `+set dedicated 2`:

```shell
node build/ioq3ded.js +set fs_game <game> +set dedicated 2 +exec <server_config>
```

If you'd just like to run a dedicated server that isn't broadcast to the master server:

```shell
node build/ioq3ded.js +set fs_game <game> +set dedicated 1 +exec <server_config>
```

### baseq3 server, step-by-step

*Note: for the initial download of game files you will need a server wth around 1GB of RAM. If the server exits with the message `Killed` then you need more memory. You can try getting around this by downloading the ``/quakejs/base/baseq3`` files on another, more powerful, machine and copying them to your low-RAM server.  I've heard a report of this being successfully completed on a machine with 512MB of RAM*

On your server clone this repository. `cd` into the `quakejs` clone and run the following commands:

```
git submodule update --init
npm install
node build/ioq3ded.js +set fs_game baseq3 +set dedicated 2
```

After running the last command continue pressing Enter until you have read the EULA, and then answer the `Agree? (y/n)` prompt. The base game files will download. When they have finished press Ctrl+C to quit the server.

In the newly created `base/baseq3` directory add a file called `server.cfg` with the following contents (adapted from [Quake 3 World](http://www.quake3world.com/q3guide/servers.html)):

```
seta sv_hostname "CHANGE ME"
seta sv_maxclients 12
seta g_motd "CHANGE ME"
seta g_quadfactor 3
seta g_gametype 0
seta timelimit 15
seta fraglimit 25
seta g_weaponrespawn 3
seta g_inactivity 3000
seta g_forcerespawn 0
seta rconpassword "CHANGE_ME"
set d1 "map q3dm7 ; set nextmap vstr d2"
set d2 "map q3dm17 ; set nextmap vstr d1"
vstr d1
```

replacing the `sv_hostname`, `g_motd` and `rconpassword`, and any other configuration options you desire.

You can now run the server with 

```
node build/ioq3ded.js +set fs_game baseq3 +set dedicated 2 +exec server.cfg
```

and you should be able to join at http://www.quakejs.com/play?connect%20SERVER_IP:27960, replacing `SERVER_IP` with the IP of your server.

## Running a content server

QuakeJS loads assets directly from a central content server. A public content server is available at `content.quakejs.com`, however, if you'd like you run your own (to perhaps provide new mods) you'll need to first repackage assets into the format QuakeJS expects.

### Repackaging assets

When repackaging assets, an asset graph is built from an incoming directory of pk3s, and an optimized set of map-specific pk3s is output to a destination directory.

To run this process:

```shell
node bin/repak.js --src <assets_src> --dest <assets>
```

And to launch the content server after the repackaging is complete:

```shell
node bin/content.js
```

Note: `./assets` is assumed to be the default asset directory. If you'd like to change that, you'll need to modify the JSON configuration used by the content server.

Once the content server is available, you can use it by launching your local or dedicated server with `+set fs_cdn <server_address>`.

## Running a local dedicated server, content server, and play page

It is possible to run a QuakeJS server, Content Server, and Play Page entirely locally (for use on a LAN with no external internet connection required).

Configure a dedicated local server as described above in **Running a dedicated server** and **baseq3 server, step-by-step**.

Copy the files from `html` to your web server's root folder.  Run `html/get_assets.sh` to download files from http://content.quakejs.com.  Rename *quakejs* on line 77 of `html/index.html` to your server's hostname.

Copy `init.d/quakejs` to `/etc/init.d/`, make it executable, and enable it by running `sudo update-rc.d quakejs defaults` (under Debian).

*Step by step instructions can be found at https://steamforge.net/wiki/index.php/How_to_setup_a_local_QuakeJS_server_under_Debian_9_or_Debian_10*

## Running Secure Servers (Content, Dedicated, and Web) Quick-Start

1. Follow the [baseq3 server, step-by-step](#baseq3-server-step-by-step) instructions to initialize your repo, get `base/` assets and start a dedicated server config.
2. Set up your `./assets` directory for the content server using the instructions in [Running a content server](#running-a-content-server).
3. Point all the `bin/*_secure.json` configs to your key and cert (if you don't already have some, you can get some for free using [certbot](https://certbot.eff.org/)). Also make sure to point the `web_secure.json` to your own domain for the content server.
4. `node bin/content.js --config ./content_secure.json`
5. `node build/ioq3ded_secure.js +set dedicated 1 +set fs_cdn <your_domain>:9000 +set fs_game baseq3 +exec server.cfg`
6. `node bin/wssproxy.js --config ./wssproxy.json`
7. `node bin/web.js --config ./web.json` (you may want to customize this server config so it serves on port 443)

You now have a content server running securely on port 9000, a dedicated server on (insecure) port 27960 (this one should be kept private, behind a firewall), a wss:// proxy running securely on port 27961, and a secure web server running on (ideally) port 443. Make sure ports 443, 9000, and 27961 are forwarded through your firewall so clients can connect to them.

Now you (and others) can connect to your secure dedicated server at `https://<SERVER_DOMAIN>/play?connect%20<SERVER_DOMAIN>:27961`, replacing `<SERVER_DOMAIN>` with the domain of your server (must match your SSL certificate).

### Notes

* Secure and insecure servers are incompatible. A secure web server cannot talk to an insecure content server, a secure content server cannot talk to an insecure dedicated server, etc.
* Master servers cannot work securely since they use IP addresses directly, so the browser would be unable to validate the SSL certificate. You can only connect directly to a known secure dedicated server using the URL above.

## Adding custom maps & content

[digidigital](https://github.com/digidigital/) has written a guide on adding custom content under Linux.  It is part of the repo and can be found at https://github.com/begleysm/quakejs/blob/master/cctools/README.md

## Windows

Step by step instructions on how to setup a server under Windows 10 can be found at https://steamforge.net/wiki/index.php/How_to_setup_a_local_QuakeJS_server_under_Windows_10#Adding_your_own_Maps

A video guide on setting up a Windows 10 server, and adding custom content, under Windows 10 has been developed by [grabisoft](https://github.com/grabisoft) and can be viewed at https://www.youtube.com/watch?v=m57rMXASWms

## Other QuakeJS Implementations
* [quakejs-installer](https://github.com/digidigital/quakejs-installer): a simple and configurable Linux-based QuakeJS "one step" Installer with some common pre-included addons. 
* [quakejs-docker](https://github.com/treyyoder/quakejs-docker): a fully local and Dockerized quakejs server. Independent, unadulterated, and free from the middleman.
* [quake-kube](https://github.com/criticalstack/quake-kube): a Kubernetes-ified version of QuakeJS that runs a dedicated Quake 3 server in a Kubernetes Deployment.

## License

MIT
