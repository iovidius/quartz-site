---
title: Plex / Tautulli Push Notifications on Android
draft: false
description: Learn how to send Plex updates to your Android phone.
date: 2021-03-16
tags:
  - "#plex"
  - "#tautulli"
  - "#selfhosting"
---
![[plex-1140x560.png]]
[Plex Media Server](https://www.plex.tv/nl/media-server-downloads/) is an outstanding solution for managing and sharing your own collection of films and (tv) series. Additionaly, [Tautulli](https://tautulli.com/) offers all the statistics you need about your Plex server. There are systems to automatically add films or series to the collection. Whenever a download is completed and an item is succesfully added to my collection I'd like to be notified, preferably with a notification on my phone.

Tautulli offers this feature. Under **Settings > Notification Agents > Plex Android / iOS App** or **Tautulli Remote Android App** you can hookup your phone and everything should work fine. Except it didn't.

I found that more people on the internet had this problem, so I assumed it wasn't my fault. After a bit of experimenting, I noticed that Tautulli was able to send a message to a MQTT Broker. Now I had someplace to start!

Note that getting this to work (MQTT → phone notification) is probably way easier with [HomeAssistant](https://www.home-assistant.io/hassio/) or some platform like that, but I wanted it to get working with the platform that I was already using: Node-RED.


# MQTT Broker

For this to work, you will be needing an MQTT Broker. The best course of action is installing [Mosquitto](https://mosquitto.org/) on your own server, which I assume you have since you're using Plex/Tautulli. [Here](https://tewarid.github.io/2019/04/03/installing-and-configuring-the-mosquitto-mqtt-broker.html) is a quick guide on how to install Mosquitto using  [Docker](https://nodered.org/docs/getting-started/docker).

# Tautulli

First I had to add a Tautulli MQTT notifier. Go to **Settings > Notification Agents > MQTT**. Then fill in the details of your broker in **Configuration**. Go to **Triggers** and select **Recently Added**. Then go to **Text** and write the following under **Recently Added**.

Subject Line:

```
Plex ({server_name})
```

Message body:

```
{title} was recently added to Plex.
${imdb_id}
```

This will be the message that will be displayed on your phone. You can obviously use variations that fit more to your wishes. The most important think is that you add `\n${imdb_id}` to the message body, as we're going to use that later.

# Firebase / FireNotify

Notifications on Android can only be displayed with the help of an app. A free app for this purpose is [FireNotify](https://github.com/eschava/FireNotify-Android). It is based on the FireBase JSON API and basically enables you to send any message to your phone through the FireBase Service ([FCM](https://firebase.google.com/docs/cloud-messaging)). Install the Android application on your phone. The app yields an API key that you will have to use later.

# OMDb API

Personally, I also want some kind of image that refers to the downloaded item to be displayed in the notification. In order to retrieve this information, we can use the [Open Movie Database (OMDb) API](http://www.omdbapi.com/). That is a free, RESTful API service. Please consider donating. Go to their website, register and generate an API key.

Now, for retrieving the image of a certain movie, we just make a simple HTTP request to `http://img.omdbapi.com/?apikey=[yourkey]&i=[imdb_id]`

# Node-RED

I used [Node-RED](https://nodered.org/) to create a flow that handles (1) the MQTT communication, (2) handles the request to OMDb and (3) pushes a notification to your phone with FCM. If you haven't already have Node-RED installed, I recommend installing it using Docker.

## FCM Environment Variable

One important step in getting FCM to work is adding an environment variable to your Node-RED's **settings.js** file. This file can be found in your configuration folder. Get the **API Key** from your FireNotify app and add the following to your settings file, right above `module.exports = {`.

```
process.env.FCM_SERVER_KEY = "your API key";
```

## Flow Configuration

I created the following flow. Import it to Node-RED (**Menu > Import)** by copy-pasting the code below.

<figure>
  <img
  src="../../imgs/nodered-flow-1.png"
  alt="Alt.">
  <figcaption><center><small>Fig 1. Flow in Node-RED.</center></small></figcaption>
</figure>

```json
[{"id":"2fbdb961.3c26b6","type":"tab","label":"Plex notification","disabled":false,"info":""},{"id":"6c6e0dde.1d9454","type":"mqtt in","z":"2fbdb961.3c26b6","name":"","topic":"plex/new","qos":"2","datatype":"auto","broker":"91478dbd.3a542","x":250,"y":180,"wires":[["25e54580.661c5a"]]},{"id":"89cc8b52.9ba8f8","type":"fcm-push","z":"2fbdb961.3c26b6","name":"","x":1170,"y":180,"wires":[]},{"id":"25e54580.661c5a","type":"json","z":"2fbdb961.3c26b6","name":"","property":"payload","action":"","pretty":false,"x":390,"y":180,"wires":[["4040a909.dce6c8"]]},{"id":"4040a909.dce6c8","type":"function","z":"2fbdb961.3c26b6","name":"","func":"msg.to=\"your_token\";\nmsg.priority=\"high\";\nvar text = msg.payload.body.split(\"\\n$\")[0];\nvar imdb_id = msg.payload.body.split(\"\\n$\")[1];\n\nmsg.title = msg.payload.subject;\nmsg.text = text;\n\nmsg.payload = {i : imdb_id}\nreturn msg;","outputs":1,"noerr":0,"initialize":"","finalize":"","x":520,"y":180,"wires":[["cb663c57.9c558"]]},{"id":"cb663c57.9c558","type":"http request","z":"2fbdb961.3c26b6","name":"","method":"GET","ret":"txt","paytoqs":"query","url":"http://www.omdbapi.com/?apikey=?????","tls":"","persist":false,"proxy":"","authType":"","x":670,"y":180,"wires":[["166ade94.244051"]]},{"id":"e26c9c71.56c17","type":"function","z":"2fbdb961.3c26b6","name":"","func":"\nvar newMsg = {payload :{to : msg.to, priority : msg.priority,\n data : {text : msg.text, title : msg.title,\n image : msg.payload.Poster\n }}};\n\nreturn newMsg;","outputs":1,"noerr":0,"initialize":"","finalize":"","x":940,"y":180,"wires":[["89cc8b52.9ba8f8"]]},{"id":"166ade94.244051","type":"json","z":"2fbdb961.3c26b6","name":"","property":"payload","action":"","pretty":false,"x":810,"y":180,"wires":[["e26c9c71.56c17"]]},{"id":"91478dbd.3a542","type":"mqtt-broker","name":"","broker":"???","port":"8883","tls":"","clientid":"???","usetls":true,"compatmode":false,"keepalive":"60","cleansession":true,"birthTopic":"","birthQos":"0","birthPayload":"","closeTopic":"","closeQos":"0","closePayload":"","willTopic":"","willQos":"0","willPayload":""}]
```

Now you'll have to configure the following things:

1. Add your MQTT credentials to the **mqtt in** node. If you've already configured a MQTT server there is a big chance Node-RED automatically fills this in for you.
2. Add your FireNotify Token. Go to your Android app and copy the **Token**. Now go back to your flow and open the first **function** node. Fill in your token in the following line: `msg.to="your_token";`
3. Finally, fill in your OMDb API key by going to the **http request** node and filling in the **URL** field, which says: `http://www.omdbapi.com/?apikey=?????`

Now you should be good to go! Deploy your flow and add something to your Plex.

<figure>
  <img
  src="../../imgs/notification.jpeg"
  alt="Alt.">
  <figcaption><center><small>Fig 2. A sample notification of <a href="https://www.imdb.com/title/tt3977848/">a movie</a> that's been added.</center></small></figcaption>
</figure>
