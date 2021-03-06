# nord-sirius - An alternative backend for your Little Printer

![CloudBerg Little Printer](https://i.vimeocdn.com/video/222115839_1280x720.jpg)

This repository is a fork of [genmon/sirius](https://github.com/genmon/sirius).

Want to try it out? We have an instance running at http://littleprinter.nordprojects.co which all are welcome to use!

### Interesting links

Before everything else, you might want to understant how to update your CloudBerg Little Printer to use it with Sirius. It seems that before you just had to give your CloudBerg Bridge MAC address and someone pushed the update on your device. Now, it appears that the only option to update your CloudBerg Bridge to use it with Sirius is to root your device. The following links might help you :

  * [Updating the Bridge](https://github.com/genmon/sirius/wiki/Updating-the-Bridge)
  * [Rooting Your BERG Cloud Bridge](http://pipt.github.io/2013/04/15/rooting-berg-cloud-bridge.html)
  * [Getting Little Printer online](http://joerick.me/hardware/2017/03/09/little-printer)
  * [Installing Sirius on Mac / Linux / Raspberry Pi](https://gist.github.com/hako/f8944cfa7b8fb8115f6d)

### Installation

Install and run Sirius on Debian:

```
apt install python python-pip libpq-dev postgresql phantomjs libfreetype6-dev fontconfig libgstreamer1.0-dev python-dev wget libjpeg-dev zlib1g-dev
git clone https://github.com/genmon/sirius
cd sirius

pip install -r requirements.txt

wget https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-2.1.1-linux-x86_64.tar.bz2
tar jxf phantomjs-2.1.1-linux-x86_64.tar.bz2
sudo mv phantomjs-2.1.1-linux-x86_64/bin/phantomjs /usr/local/bin/phantomjs

./manage.py db upgrade head

~/.local/bin/gunicorn -k flask_sockets.worker manage:app -b 0.0.0.0:5002 -w 1
```
or use Docker Compose, just type:

```
docker-compose up
```

*With PhantomJS 1.9.8 I can't print pictures but version 2.1.1 seems to fix the problem*.

Navigate browser to http://127.0.0.1:5002/

You should configure your BergCloud Bridge to point to your Sirius instance.

If you want, you can create a systemd service, feel free to update this skeleton to fit your needs:

```
[Unit]
Description=Sirius, webapp for Little Printer

[Service]
WorkingDirectory=/home/berg/sirius
ExecStart=/home/berg/.local/bin/gunicorn -k flask_sockets.worker manage:app -b 0.0.0.0:5002 -w 1
User=berg

[Install]
WantedBy=multi-user.target
```

### Using the external API

If you want to print messages from your application, you can use the external API.


#### With curl

```bash
curl \
  --data '{"message": "<h1>hello</h1>", "face": "noface"}' \
  http://127.0.0.1:5000/ext_api/v1/printer/1/print_html?api_key=<key>
```

#### With Processing

You'll need an external library named unirest and its dependencies.
You can download it directly from my server : [unirest-java-1.4.10.jar](https://cloud.deuxfleurs.fr/f/685c77efbd/?raw=1)
Or, if you prefer, you can compile it from the source by following this [guide](http://blog.mashape.com/installing-unirest-java-with-the-maven-assembly-plugin/).

Once you have the `unirest-java-1.4.10.jar` (or another version):

  1. Create a folder named `code` inside your sketchbook folder, next to your Processing files (use File explorer on Windows, Finder on Mac OS, etc.)
  2. Put `unirest-java-1.4.10.jar` in this folder

Now restart processing and add the following in your project:

```processing
import com.mashape.unirest.http.*;
import com.mashape.unirest.http.exceptions.UnirestException;

boolean little_print(String face, String message, String endpoint, String api_key) {
  try {
    com.mashape.unirest.http.HttpResponse<String> request = Unirest.post(endpoint+"?api_key="+api_key)
      .body("{\"message\": \"" + message.replace("\"", "\\\"") +"\", \"face\": \"" + face.replace("\"", "\\\"") + "\"}")
      .asString();

    println(request.getBody());
    return true;
  } catch (Exception e) {
    println(e);
    return false;
  }
}

```

Each time you want to use your little printer to print something, you should write something like this (don't forget to put your API key and update the URL):

```processing
little_print("noface", "<h1 style=\"margin-top: 100px\"> PROOOOCESSING </h1>", "http://127.0.0.1:5000/ext_api/v1/printer/1/print_html", "xxxxxxxxx");
```

### Environment variables

The server can be configured with the following variables:

```
TWITTER_CONSUMER_KEY=...
TWITTER_CONSUMER_SECRET=...
FLASK_CONFIG=...
```

For dokku this means using e.g.:

```
dokku config:set sirius FLASK_CONFIG=heroku
dokku config:set sirius TWITTER_CONSUMER_KEY=DdrpQ1uqKuQouwbCsC6OMA4oF
dokku config:set sirius TWITTER_CONSUMER_SECRET=S8XGuhptJ8QIJVmSuIk7k8wv3ULUfMiCh9x1b19PmKSsBh1VDM
```

### Creating fake printers and friends

Resetting the actual hardware all the time gets a bit tiresome so
there's a fake command that creates unclaimed fake little printers:

```console
$ ./manage.py fake printer
[...]
Created printer
     address: 602d48d344b746f5
       DB id: 8
      secret: 66a596840f
  claim code: 5oop-e9dp-hh7v-fjqo
```

Functionally there is no difference between resetting and creating a new printer so we don't distinguish between the two.

To create a fake friend from twitter who signed up do this:

```console
$ ./manage.py fake user stephenfry
```

You can also claim a printer in somebody else's name:

```console
$ ./manage.py fake claim b7235a2b432585eb quentinsf 342f-eyh0-korc-msej testprinter
```

## Sirius Architecture

### Layers

The design is somewhat stratified: each layer only talks to the one
below and above. The ugliest bits are how database and protocol loop
interact.

```
UI / database
----------------------------
protocol_loop / send_message
----------------------------
encoders / decoders
----------------------------
websockets
----------------------------
```

### Information flow (websockets)

The entry point for the bridge is in `sirius.web.webapp`. Each new
websocket connection spawns a gevent thread (specified by running the
flask_sockets gunicorn worker) which runs
`sirius.protocol.protocol_loop.accept` immediately. `accept` registers
the websocket/bridge_address mapping in a global dictionary; it then
loops forever, decoding messages as they come in.


### Claim codes

Devices are associated with an account when a user enters a "claim
code". This claim code contains a "hardware-xor" which is derived via
a lossy 3-byte hash from the device address. The XOR-code for a device
is always the same even though the address changes!

The claim codes are meant to be used "timely", i.e. within a short
window of the printer reset. If there are multiple, conflicting claim
codes we always pick the most recently created code.

We are also deriving this hardware xor when a device calls home with
an "encryption_key_required". In that case we connect the device to
the claim code via the hardware-xor and send back the correct
encryption key.
