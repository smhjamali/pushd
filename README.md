Universal Mobile Push Daemon
============================

*Pushd* is a free and open source software which provides a unified push service for server-side notification to apps on mobile devices. With pushd you can push notification to any supported mobile platform. Pushd takes care of which device is subscribed to which event and is designed to support an unlimited amount of subscribable events.

![Architecture Overview](https://github.com/rs/pushd/raw/master/doc/overview.png)

Features
--------

- Multi protocols (APNs (iOS), C2DM (Android), MPNS (Windows Phone)
- Register unlimited number of subscribers (device)
- Subscribe to unlimited number of events
- Automatic badge increment for iOS
- Silent subscription mode (no alert message, only data or badge increment)
- Server side message translation
- Message template
- Broadcast
- Events statistics
- Automatic failing subscriber unregistration
- Apple Feedback API support
- Redis backend
- Fracking fast!

Installation
------------

- Install [redis](http://redis.io/), [node.js](http://nodejs.org/), [npm](http://npmjs.org/) and [coffeescript](http://coffeescript.org/).
- Clone the repository: `git clone git://github.com/rs/pushd.git && cd pushd`
- Install dependancies: `npm install`
- Configure the server: `cp settings-sample.coffee settings.coffee && vi settings.coffee`
- Start redis: `redis-server`
- Start the server: `sudo coffee pushd.coffee`

Getting Started
---------------

### Register

At first launch, your app must register with the push notification service to get a registration id. It then provides this registration id to pushd in exchange for a subscriber id (This subscriber id will be used with all further communications with pushd). Some informations can be sent with the request to pushd like: subscriber language, version or current badge value.

subscriber registration is performed through a HTTP REST API (see later for more details). Here is an example of a subscriber registration simulated using the curl command. As an example, we will register the iOS device with the registration id `FE66489F304DC75B8D6E8200DFF8A456E8DAEACEC428B427E9518741C92C6660`. For iOS, we have to specify the `apns` protocol. We also set the subscriber language to `fr` for French and init the badge to `0`. We suppose the command is run on the same machine as pushd:

    $ curl -d proto=apns \
           -d token=FE66489F304DC75B8D6E8200DFF8A456E8DAEACEC428B427E9518741C92C6660 \
           -d lang=fr \
           -d badge=0 \
           http://localhost/subscribers

In reply, we get the following JSON structure:

    {
        "proto":"apns",
        "token":"fe66489f304dc75b8d6e8200dff8a456e8daeacec428b427e9518741c92c6660",
        "lang":"fr",
        "badge":0,
        "updated":1332953375,
        "created":1332953375,
        "id":"J8lHY4X1XkU"
    }

Your app must save the `id` field value, it will be used for all further communication with pushd.

### Ping

Once the app is registered, it has to ping the pushd server each time the app is launched to let pushd know the subscriber still exists. The subscriber may have been unregistered automatically in case of repeated errors for instance. To ping pushd, you perform a POST on the `/subscriber/subscriber_ID` url as follow:

    $ curl -d lang=fr -d badge=0 http://localhost/subscriber/J8lHY4X1XkU

On iOS, you must update the badge value to inform pushd the user read the pending notifications. You may call this URL several times, each time the badge is updated, so the next notification will still increment the badge with the correct value.

### Subscriptions

Depending on your service, your app may auto-subscribe the subscriber to some events or ask the user which events he wants to be subscribed to (an event is identified as an arbitrary string meaningful for you service). For each event your app wants to be subscribed to, a call to the pushd API must be performed.

For instance, if your app is news related, you may want to create one subscriptable event for each news category. So if your user wants to subscribe to `sport` events, the following call to pushd has to be performed:

    $ curl -X POST http://localhost/subscriber/J8lHY4X1XkU/subscriptions/sport

You may later unsubscribe by switching from the `POST` to the `DELETE` method.

We recommend to auto-subscribe your users to some global event like for instance a country event if your app is international. This will let you send targeted messages to all of a given country’s users.

### Event Ingestion

Once subscribers are registered, your service may start to send events. Events are composed of a message, optionally translated in several languages and some additional data to be passed to your application. To send an event, you may either use the HTTP REST API or send UDP datagrams.

You don't need to create events before sending them. If nobody is subscribed to a given event, it will be simply ignored. It's thus recommended to send all the possible types of events and let your application choose which to subscribe to.

Here we will send a message to all subscribers subscribed to the `sport` event:

    $ curl -d msg=Test%20message http://localhost/event/sport

API
---

### Subscriber Registration

#### Register a subscriber ID

Register a subscriber by POSTing on `/subscribers` with some subscriber information like registration id, protocol, language, OS version (useful for Windows Phone OS) or initial badge number (only relevant for iOS, see bellow).

    > POST /subscribers HTTP/1.1
    > Content-Type: application/x-www-form-urlencoded
    > 
    > proto=apns
    > token=FE66489F304DC75B8D6E8200DFF8A456E8DAEACEC428B427E9518741C92C6660&
    > lang=fr&
    > badge=0
    > 
    ---
    < HTTP/1.1 201 Created
    < Location: /subscriber/JYJ1ehuEHbU
    < Content-Type: application/json
    < 
    < {
    <   "created":1332638892,
    <   "updated":1332638892,
    <   "proto":"apns",
    <   "token":"FE66489F304DC75B8D6E8200DFF8A456E8DAEACEC428B427E9518741C92C6660",
    <   "lang":"fr",
    <   "badge":10
    < }

*Carriage returns are added for readability*

##### Mandatory parameters:

- `proto`: The protocol to be used for the subscriber. Use one of the following values:
	- `apns`: iOS (Apple Push Notification service)
	- `c2dm`: Android (Cloud to subscriber Messaging)
	- `mpns` Window Phone (Microsoft Push Notification Service)
- `token`: The device registration id delivered by the platform's push notification service 

##### Allowed parameters:

- `lang`: The language code for the of the subscriber. This parameter is used to determine which message translation to use when pushing text notifications. You may use the 2 chars ISO code or a complete locale code (i.e.: en_CA) or any value you want as long as you provide the same values in your events. See below for info about events formatting.
- `badge`: The current app badge value. This parameter is only applicable to iOS for which badge counters must be maintained server side. On iOS, when a user read or loads more unread items, you must inform the server of the badge's new value. This badge value will be incremented automatically by pushd each time a new notification is sent to the subscriber.
- `version`: This is the OS subscriber version. This parameter is only needed by Windows Phone OS. By setting this value to 7.5 or greater an `mpns` subscriber ids will enable new MPNS push features.

##### Return Codes

- `200` subscriber previously registered
- `201` subscriber successfully registered
- `400` Invalid specified registration id or protocol

#### Update subscriber Registration Info

On each app launch, it is highly recommended to update your subscriber information in order to inform pushd your subscriber is still alive and registered for notifications. Do not forget to check if the app notifications hasn't been disabled since the last launch, and call `DELETE` if so. If this request returns a 404 error, it means your subscriber registration has been cancelled by pushd. You must then delete the previously obtained subscriber id and restart the registration process for this subscriber. Registration can be cancelled after pushd error count for the subscriber reached a predefined threshold or if the target platform push service informed pushd about an inactive subscriber (i.e. Apple Feedback Service).

    > POST /subscriber/subscriber_ID HTTP/1.1
    > Content-Type: application/x-www-form-urlencoded
    >
    > lang=fr&badge=0
    >
    ---
    < HTTP/1.1 204 No Content

##### Allowed parameters:

- `lang`: The language code for the of the subscriber. This parameter is used to determine which message translation to use when pushing text notifications. You may use the 2 chars ISO code or a complete locale code (i.e.: en_CA) or any value you want as long as you provide the same values in your events. See below for info about events formatting.
- `badge`: The current app badge value. This parameter is only applicable to iOS for which badge counters must be maintained server side. On iOS, when a user read or loads more unread items, you must inform the server of the badge's new value. This badge value will be incremented automatically by pushd each time a new notification is sent to the subscriber.
- `version`: This is the OS subscriber version. This parameter is only needed by Windows Phone OS. By setting this value to 7.5 or greater an `mpns` subscriber ids will enable new MPNS push features.

NOTE: this method should be called each time the app is opened to inform pushd the subscriber is still alive. If you don’t, the subscriber may be automatically unregistered in case of repeated push error.

##### Return Codes

- `204` subscriber info edited successfully
- `400` Format of the subscriber id or a field value is invalid
- `404` The specified subscriber does not exist

#### Unregister a subscriber ID

When the user chooses to disable notifications from within your app, you can delete the subscriber from pushd so pushd won't send further push notifications.

    > DELETE /subscriber/subscriber_ID HTTP/1.1
    >
    ---
    < HTTP/1.1 204 No Content

##### Return Codes

- `204` subscriber unregistered successfully
- `400` Invalid subscriber id format
- `404` The specified subscriber does not exist

#### Get information about a subscriber ID

You may want to read informations stored about a subscriber id.

    > GET /subscriber/subscriber_ID HTTP/1.1
    >
    ---
    < HTTP/1.1 200 Ok
    < Content-Type: application/json
    <
    < {
    <   "created":1332638892,
    <   "updated":1332638892,
    <   "proto":"apns",
    <   "token":"FE66489F304DC75B8D6E8200DFF8A456E8DAEACEC428B427E9518741C92C6660",
    <   "lang":"fr",
    <   "badge":10
    < }

*Carriage returns are added for readability*

##### Return Codes
- `200` subscriber exists, information returned
- `400` Invalid subscriber id format
- `404` The specified subscriber does not exist

#### Subscribe to an Event

For pushd, an event is represented as a simple string. By default a subscriber won't receive push notifications other than broadcasts or direct messages if it’s not subscribed to events. Events are text and/or data sent by your service on pushd. Pushd's role is to convert this event into a push notification for any subscribed subscriber.

You subscribe a previously registered subscriber by POSTing on `/subscriber/subscriber_ID/subscriptions/EVENT_NAME` where `EVENT_NAME` is a unique string code for the event. You may post an option parameter to configure the subscription.

	> POST /subscriber/subscriber_ID/subscriptions/EVENT_NAME HTTP/1.1
	> Content-Type: application/x-www-form-urlencoded
    >
	> ignore_message=1
    >
    ---
	< HTTP/1.1 204 No Content

##### Allowed Parameter

- `ignore_message`: Defaults to 0, if set to 1, the message part of the subscribed event won't be sent with the notification. On iOS, the badge will still be incremented and updated. You can use this to update the badge counter on iOS without disturbing the user if he didn't want to be explicitly notified for this event but still wants to know how many new items are available. On Android you may use this kind of subscription to notify your application about new items available so it can perform background pre-fetching without user notification.

##### Return Codes

- `201` Subscription successfully created
- `204` Subscription successfully updated
- `400` Invalid subscriber id event name format
- `404` The specified subscriber does not exist

#### Unsubscribe from an Event

To unsubscribe from an event, perform a DELETE on the subscription URL.

	> DELETE /subscriber/subscriber_ID/subscriptions/EVENT_NAME HTTP/1.1
    >
    ---
	< HTTP/1.1 204 No Content

##### Return Codes

- `204` Subscription deleted
- `400` Invalid subscriber id or event name format
- `404` The specified subscriber does not exist

#### List subscribers’ Subscriptions

To get the list of events a subscriber is subscribed to, perform a GET on the `/subscriber/subscriber_ID/subscriptions`.

    > GET /subscriber/subscriber_ID/subscriptions HTTP/1.1
    >
    ---
    < HTTP/1.1 200 Ok
    < Content-Type: application/json
    <
    < {
    <   "EVENT_NAME": {"ignore_message": false},
    <   "EVENT_NAME2": ...
    < }

To test for the presence of a single subscription, perform a GET on the subscription URL

    > GET /subscriber/subscriber_ID/subscriptions/EVENT_NAME HTTP/1.1
    >
    ---
    < HTTP/1.1 200 Ok
    < Content-Type: application/json
    <
    < {"ignore_message":false}

### Event Ingestion

To generate notifications, your service must send events to pushd. The service doesn't have to know if a subscriber is subscribed to an event in order to send it, it just send all subscriptable events as they happen and pushd handles the rest.

An event is a JSON object in a specific format sent to pushd either using HTTP POST or UDP datagrams.

#### Event Message Format

An event message is a dictionary of optional key/values:

- `title`, `msg`: The event title/message. If no title nor message are provided, the event will only send data to the app and won't notify the user. The `title` is only relevant for Android and Windows Phone, iOS doesn't support this value: the application name is always used as notification title. *The value can contain placeholders to other keys, see Message Template bellow.*
- `title.<lang>`, `msg.<lang>`: The translated version of the event title/message. The `<lang>` part must match the `lang` property of a target subscriber. If subscribers use full locale (i.e. `fr_CA`), and no matching locale value is provided, pushd will fallback to a language only version of the value if any (i.e. `fr`). If no translation matches, the `title` or `msg` key is used. *The value can contain placeholders to other keys, see Message Template bellow.*
- `data.<key>`: Key/values to be attached to the notification
- `var.<key>`: Stores strings to be reused in `msg` and `<lang>.msg` contents
- `sound`: The name of a sound file to be played. It must match a sound file name contained in you bundle app. (iOS only)

#### Event Message Template

The `msg` and `<lang>.msg` keys may contain references to others keys in the event object. You may refer either to `data.<key>` or `var.<key>`. Use the `${<key name>}` syntax to refer to those keys (ex: `${var.title}`).

Here is an example of an event message using translations and templating (spaces and carriage returns have been added for readability):

    msg=${var.name} sent a new video: ${var.title}
    msg.fr=${var.name} a envoyé une nouvelle video: ${var.title}
    sound=newVideo.mp3
    data.user_id=fkwhpd
    data.video_id=1k3dxk
    var.name=John Doe
    var.title=Super awesome video

#### Event API

##### HTTP

To send an event to pushd over HTTP, POST the JSON object to the `/event/EVENT_NAME` endpoint of the pushd server:

    > POST /event/user.newVideo:fkwhpd HTTP/1.1
    > Content-Type: application/x-www-form-urlencoded
    >
    > msg=${var.name} sent a new video: ${var.title}&
    > msg.fr=${var.name} a envoyé une nouvelle video: ${var.title}&
    > sound=newVideo.mp3&
    > data.user_id=fkwhpd&
    > data.video_id=1k3dxk&
    > var.name=Jone Doe&
    > var.title=Super awesome video
    ---
    < HTTP/1.1 204 Ok

*Carriage returns are added for readability*

The server will answer OK immediately. This doesn't mean the event has already been delivered.

##### UDP

The UDP event posting API consists of a UDP datagram targeted at the UDP port 80 containing the URI of the event followed by the message content as query-string:

    /event/user.newVideo:fkwhpd?msg=%24%7Bvar.name%7D+sent+a+new+video%3A+%24%7Bvar.title%7D&msg.fr=%24%7Bvar…


License
-------

(The MIT License)

Copyright (c) 2011 Olivier Poitrey <rs@dailymotion.com>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the 'Software'), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
