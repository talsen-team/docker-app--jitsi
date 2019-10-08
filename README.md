Hello there,

I'm coming from a few different GitHub issues / community questions ( https://github.com/jitsi/jitsi-meet/issues/2422 ; https://community.jitsi.org/t/issues-with-using-nginx-on-separate-server/15783 ; https://github.com/jitsi/jitsi-meet/issues/223 ).

Link back to [community.jitsi.org](https://community.jitsi.org/t/server-docker-jitsi-from-subroute-behind-nginx-reverse-proxy/20134)

## How does our setup look like:

We have a certain sub-domain ( e.g. sub.example.com ), which is serving an application that should / will integrate Jitsi via an IFrame.
We want to be able to reload IFrames for sub.example.com, so Jitsi must come from the same sub-domain.
The idea is that Jitsi ( running in docker -> https://github.com/jitsi/docker-jitsi-meet ) is not directly served at the sub-domain level, but instead on a sub-route ( e.g. sub.example.com/__jitsi/ ).
The sub-domain sub.example.com is pointing to a publicly visible IP address ( so no advanced config for the JVB will be needed, as far as I understood ).

Docker Jitsi also will be running in HTTP mode behind an extra NGINX reverse proxy.
The nginx reverse proxy configuration looks like the following:

```
server {
    listen                    443 ssl http2;
    server_name               ${DOMAIN};

    ssl_certificate           /etc/letsencrypt/live/${PATH}/fullchain.pem;
    ssl_certificate_key       /etc/letsencrypt/live/${PATH}/privkey.pem;
    ssl_dhparam               /etc/ssl/dhparams.pem;

    ssl_ciphers               "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA";
    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_session_cache         shared:SSL:10m;
    add_header                Strict-Transport-Security "max-age=63072000; includeSubdomains; preload" always;
    add_header                X-Frame-Options SAMEORIGIN;
    add_header                X-Content-Type-Options nosniff;
    ssl_session_tickets       off;
    ssl_stapling              on;
    ssl_stapling_verify       on;

    root                      /etc/letsencrypt/webrootauth;

    location / {
      proxy_pass              http://${APP_UPSTREAM};
      proxy_set_header        Host $host;
      proxy_set_header        X-Forwarded-Host $host:$server_port;
      proxy_set_header        X-Forwarded-Server $host;
      proxy_set_header        X-Forwarded-For $remote_addr;
      proxy_set_header        X-Forwarded-Proto $scheme;
      proxy_cache             anonymous;
      proxy_buffering         off;
      proxy_http_version      1.1;
      proxy_set_header        Upgrade $http_upgrade;
      proxy_set_header        Connection $http_connection;
    }

    location /__jitsi/ {
      proxy_pass              http://${JITSI_UPSTREAM}/;
      proxy_set_header        Host $host;
      proxy_set_header        X-Forwarded-Host $host:$server_port;
      proxy_set_header        X-Forwarded-Server $host;
      proxy_set_header        X-Forwarded-For $remote_addr;
      proxy_set_header        X-Forwarded-Proto $scheme;
      proxy_cache             anonymous;
      proxy_buffering         off;
      proxy_http_version      1.1;
      proxy_set_header        Upgrade $http_upgrade;
      proxy_set_header        Connection $http_connection;
    }
}

server {
    listen                    80;
    server_name               ${DOMAIN};
    return                    301 https://$server_name$request_uri;
}
```
where **APP_UPSTREAM** will be replaced with the local upstream IP for our application ( e.g. 127.0.0.1:<APP_HTTP_PORT> ) and **JITSI_UPSTREAM** will be replaced with the local upstream IP for docker jitsi ( e.g. 127.0.0.1:<JITSI_HTTP_PORT> ).

In the **.jitsi-meet-cfg/web/config.js** changed the following:

```
var config = {
    // Custom function which given the URL path should return a room name.
    getroomnode: function (path) { return path.replace('__jitsi', ''); },

    ...

    // BOSH URL. FIXME: use XEP-0156 to discover it.
    bosh: '/__jitsi/http-bind',
```
all other values stayed as they were generated at the first docker container startup.

Docker Jitsi uses the JVB_PORT 10000/udp and 4443 which are both passed through the server firewall.

## What does now not work as expected:

When I will now run Docker Jitsi and try to connect via Google Chrome ( 77.0.3865.90 ) to the mentioned address sub.example.com/__jitsi/test-room the room is loading correctly and works so far.
When I'm using a private Chrome window and add a second attendee the rooms stay stable as well ( audio, video and screen-sharing work via P2P at least when both endpoints are on the same machine ).
After adding a third attendee the Jitsi steaming switches from P2P to a relayed connection. That's where the audio, video and screen-sharing stop woorking.

The Chrome Console Log looks like the following after all three steps happened:
```
(TIME) index.html loaded:	 446.85999999637716
Logger.js:125 [modules/browser/BrowserCapabilities.js] <new t>:  This appears to be chrome, ver: 77.0
LocalStatsCollector.js:22 The AudioContext was not allowed to start. It must be resumed (or created) after a user gesture on the page. https://goo.gl/7K7WLu
(anonymous) @ LocalStatsCollector.js:22
n @ bootstrap:19
(anonymous) @ SDPUtil.js:672
(anonymous) @ statistics.js:755
n @ bootstrap:19
(anonymous) @ JitsiConferenceEventManager.js:1
(anonymous) @ JitsiConferenceEventManager.js:655
n @ bootstrap:19
(anonymous) @ JitsiConnection.js:154
(anonymous) @ JitsiConference.js:3061
n @ bootstrap:19
(anonymous) @ JitsiConnection.js:1
n @ bootstrap:19
(anonymous) @ index.js:3
(anonymous) @ JitsiMeetJS.js:123
n @ bootstrap:19
(anonymous) @ index.js:3
n @ bootstrap:19
(anonymous) @ bootstrap:83
(anonymous) @ bootstrap:83
(anonymous) @ universalModuleDefinition:9
(anonymous) @ universalModuleDefinition:1
Logger.js:125 [react/index.web.js] <HTMLDocument.<anonymous>>:  (TIME) document ready:	 2374.439999985043
Logger.js:125 [react/features/base/storage/PersistenceRegistry.js] <e.value>:  redux state rehydrated as {features/base/settings: {…}, features/dropbox: {…}, features/recent-list: Array(1), features/welcome: {…}, features/video-layout: {…}, …}
Logger.js:125 [modules/UI/videolayout/VideoLayout.js] <Object.changeUserAvatar>:  Missed avatar update - no small video yet for undefined
o @ Logger.js:125
changeUserAvatar @ VideoLayout.js:905
_.refreshAvatarDisplay @ UI.js:616
De @ middleware.js:317
(anonymous) @ middleware.js:116
dispatch @ redux.js:563
(anonymous) @ middleware.js:197
(anonymous) @ middleware.js:57
dispatch @ redux.js:563
(anonymous) @ actions.js:18
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
(anonymous) @ BaseApp.js:85
Promise.then (async)
value @ BaseApp.js:85
value @ AbstractApp.js:49
Na @ react-dom.production.min.js:228
Ta @ react-dom.production.min.js:220
Pa @ react-dom.production.min.js:219
Oa @ react-dom.production.min.js:216
Zi @ react-dom.production.min.js:214
Ra @ react-dom.production.min.js:233
Ba @ react-dom.production.min.js:233
Va.render @ react-dom.production.min.js:241
(anonymous) @ react-dom.production.min.js:244
Fa @ react-dom.production.min.js:231
Wa @ react-dom.production.min.js:244
render @ react-dom.production.min.js:246
(anonymous) @ index.web.js:24
Show 22 more frames
Logger.js:125 [JitsiMeetJS.js] <Object.init>:  Analytics disabled, disposing.
i @ Logger.js:125
init @ JitsiMeetJS.js:179
(anonymous) @ actions.js:49
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
dispatch @ redux.js:563
(anonymous) @ middleware.js:83
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
dispatch @ redux.js:563
(anonymous) @ actions.js:86
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
dispatch @ redux.js:563
c @ actions.js:87
(anonymous) @ actions.js:64
Promise.then (async)
(anonymous) @ actions.js:63
(anonymous) @ actions.js:139
(anonymous) @ actions.js:35
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
value @ AbstractApp.js:136
(anonymous) @ AbstractApp.js:54
Promise.then (async)
value @ AbstractApp.js:51
Na @ react-dom.production.min.js:228
Ta @ react-dom.production.min.js:220
Pa @ react-dom.production.min.js:219
Oa @ react-dom.production.min.js:216
Zi @ react-dom.production.min.js:214
Ra @ react-dom.production.min.js:233
Ba @ react-dom.production.min.js:233
Va.render @ react-dom.production.min.js:241
(anonymous) @ react-dom.production.min.js:244
Fa @ react-dom.production.min.js:231
Wa @ react-dom.production.min.js:244
render @ react-dom.production.min.js:246
(anonymous) @ index.web.js:24
Show 118 more frames
Logger.js:125 [modules/statistics/AnalyticsAdapter.js] <e.value>:  Disposing of analytics adapter.
i @ Logger.js:125
value @ AnalyticsAdapter.js:108
init @ JitsiMeetJS.js:180
(anonymous) @ actions.js:49
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
dispatch @ redux.js:563
(anonymous) @ middleware.js:83
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
dispatch @ redux.js:563
(anonymous) @ actions.js:86
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
dispatch @ redux.js:563
c @ actions.js:87
(anonymous) @ actions.js:64
Promise.then (async)
(anonymous) @ actions.js:63
(anonymous) @ actions.js:139
(anonymous) @ actions.js:35
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
value @ AbstractApp.js:136
(anonymous) @ AbstractApp.js:54
Promise.then (async)
value @ AbstractApp.js:51
Na @ react-dom.production.min.js:228
Ta @ react-dom.production.min.js:220
Pa @ react-dom.production.min.js:219
Oa @ react-dom.production.min.js:216
Zi @ react-dom.production.min.js:214
Ra @ react-dom.production.min.js:233
Ba @ react-dom.production.min.js:233
Va.render @ react-dom.production.min.js:241
(anonymous) @ react-dom.production.min.js:244
Fa @ react-dom.production.min.js:231
Wa @ react-dom.production.min.js:244
render @ react-dom.production.min.js:246
(anonymous) @ index.web.js:24
Show 119 more frames
Logger.js:125 [react/features/base/media/middleware.js] <>:  Start muted: 
Logger.js:125 [react/features/base/media/middleware.js] <>:  Start audio only set to false
Logger.js:125 [react/features/base/conference/middleware.js] <>:  Audio-only disabled
Logger.js:125 [modules/RTC/RTCUtils.js] <t.value>:  Using the new gUM flow
Logger.js:125 [modules/xmpp/xmpp.js] <t.value>:  P2P STUN servers:  (3) [{…}, {…}, {…}]
Logger.js:125 [modules/xmpp/xmpp.js] <t.value>:  Lip-sync enabled !
Logger.js:125 [modules/xmpp/xmpp.js] <t.value>:  (TIME) Strophe connecting:	 2526.769999996759
Logger.js:125 [modules/RTC/RTCUtils.js] <t.<anonymous>>:  Got media constraints:  {video: {…}, audio: {…}}
Logger.js:125 [modules/RTC/RTCUtils.js] <>:  onUserMediaSuccess
Logger.js:125 [react/features/base/tracks/functions.js] <>:  Failed to create local tracks (2) ["audio", "video"] a {name: "track.no_data_from_source", message: "The track has stopped receiving data from it's source", stack: "Error↵    at new a (https://sub.example.com/__jit…_jitsi/libs/lib-jitsi-meet.min.js?v=3216:6:108824"}
o @ Logger.js:125
(anonymous) @ functions.js:92
Promise.catch (async)
a @ functions.js:91
createInitialLocalTracksAndConnect @ conference.js:628
init @ conference.js:719
(anonymous) @ actions.web.js:28
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
value @ Conference.js:276
value @ Conference.js:157
Na @ react-dom.production.min.js:228
Ta @ react-dom.production.min.js:220
Pa @ react-dom.production.min.js:219
Oa @ react-dom.production.min.js:216
Zi @ react-dom.production.min.js:214
enqueueSetState @ react-dom.production.min.js:134
w.setState @ react.production.min.js:13
n @ I18n.js:135
t @ I18n.js:147
(anonymous) @ EventEmitter.js:46
e.emit @ EventEmitter.js:45
(anonymous) @ i18next.js:162
(anonymous) @ i18next.js:265
(anonymous) @ i18next.js:281
t.load @ BackendConnector.js:183
(anonymous) @ i18next.js:215
t.load @ CacheConnector.js:40
t.loadResources @ i18next.js:214
r @ i18next.js:280
t.changeLanguage @ i18next.js:286
s @ i18next.js:159
setTimeout (async)
t.init @ i18next.js:171
(anonymous) @ i18next.js:68
n @ bootstrap:19
(anonymous) @ actionTypes.js:21
n @ bootstrap:19
(anonymous) @ actions.js:19
n @ bootstrap:19
(anonymous) @ actions.js:265
(anonymous) @ functions.js:97
n @ bootstrap:19
(anonymous) @ AnalyticsEvents.js:630
n @ bootstrap:19
(anonymous) @ index.js:1
(anonymous) @ actions.js:715
n @ bootstrap:19
(anonymous) @ toConsumableArray.js:8
n @ bootstrap:19
(anonymous) @ actions.js:5
(anonymous) @ actions.js:265
n @ bootstrap:19
(anonymous) @ index.js:3
n @ bootstrap:19
(anonymous) @ actions.js:20
n @ bootstrap:19
(anonymous) @ possibleConstructorReturn.js:16
n @ bootstrap:19
(anonymous) @ actions.js:17
n @ bootstrap:19
(anonymous) @ AuthHandler.js:1
(anonymous) @ AuthHandler.js:229
n @ bootstrap:19
(anonymous) @ connection.js:1
(anonymous) @ connection.js:189
n @ bootstrap:19
(anonymous) @ Tooltip.js:289
(anonymous) @ conference.js:2786
n @ bootstrap:19
(anonymous) @ withAnalyticsEvents.js:111
n @ bootstrap:19
(anonymous) @ bootstrap:83
(anonymous) @ bootstrap:83
Show 81 more frames
Logger.js:125 [modules/RTC/RTCUtils.js] <t.value>:  Using the new gUM flow
Logger.js:125 [modules/RTC/RTCUtils.js] <t.<anonymous>>:  Got media constraints:  {video: false, audio: {…}}
Logger.js:125 [modules/RTC/RTCUtils.js] <>:  onUserMediaSuccess
Logger.js:125 [react/features/base/tracks/functions.js] <>:  Failed to create local tracks ["audio"] a {name: "track.no_data_from_source", message: "The track has stopped receiving data from it's source", stack: "Error↵    at new a (https://sub.example.com/__jit…_jitsi/libs/lib-jitsi-meet.min.js?v=3216:6:108824"}
o @ Logger.js:125
(anonymous) @ functions.js:92
Promise.catch (async)
a @ functions.js:91
(anonymous) @ conference.js:637
Promise.catch (async)
createInitialLocalTracksAndConnect @ conference.js:630
init @ conference.js:719
(anonymous) @ actions.web.js:28
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
value @ Conference.js:276
value @ Conference.js:157
Na @ react-dom.production.min.js:228
Ta @ react-dom.production.min.js:220
Pa @ react-dom.production.min.js:219
Oa @ react-dom.production.min.js:216
Zi @ react-dom.production.min.js:214
enqueueSetState @ react-dom.production.min.js:134
w.setState @ react.production.min.js:13
n @ I18n.js:135
t @ I18n.js:147
(anonymous) @ EventEmitter.js:46
e.emit @ EventEmitter.js:45
(anonymous) @ i18next.js:162
(anonymous) @ i18next.js:265
(anonymous) @ i18next.js:281
t.load @ BackendConnector.js:183
(anonymous) @ i18next.js:215
t.load @ CacheConnector.js:40
t.loadResources @ i18next.js:214
r @ i18next.js:280
t.changeLanguage @ i18next.js:286
s @ i18next.js:159
setTimeout (async)
t.init @ i18next.js:171
(anonymous) @ i18next.js:68
n @ bootstrap:19
(anonymous) @ actionTypes.js:21
n @ bootstrap:19
(anonymous) @ actions.js:19
n @ bootstrap:19
(anonymous) @ actions.js:265
(anonymous) @ functions.js:97
n @ bootstrap:19
(anonymous) @ AnalyticsEvents.js:630
n @ bootstrap:19
(anonymous) @ index.js:1
(anonymous) @ actions.js:715
n @ bootstrap:19
(anonymous) @ toConsumableArray.js:8
n @ bootstrap:19
(anonymous) @ actions.js:5
(anonymous) @ actions.js:265
n @ bootstrap:19
(anonymous) @ index.js:3
n @ bootstrap:19
(anonymous) @ actions.js:20
n @ bootstrap:19
(anonymous) @ possibleConstructorReturn.js:16
n @ bootstrap:19
(anonymous) @ actions.js:17
n @ bootstrap:19
(anonymous) @ AuthHandler.js:1
(anonymous) @ AuthHandler.js:229
n @ bootstrap:19
(anonymous) @ connection.js:1
(anonymous) @ connection.js:189
n @ bootstrap:19
(anonymous) @ Tooltip.js:289
(anonymous) @ conference.js:2786
n @ bootstrap:19
(anonymous) @ withAnalyticsEvents.js:111
n @ bootstrap:19
(anonymous) @ bootstrap:83
(anonymous) @ bootstrap:83
Show 82 more frames
Logger.js:125 [modules/RTC/RTCUtils.js] <t.value>:  Using the new gUM flow
Logger.js:125 [modules/RTC/RTCUtils.js] <t.<anonymous>>:  Got media constraints:  {video: {…}, audio: false}
Logger.js:125 [modules/RTC/RTCUtils.js] <>:  onUserMediaSuccess
favicon.ico:1 GET https://sub.example.com/images/favicon.ico?v=1 404
Logger.js:125 [modules/xmpp/xmpp.js] <t.value>:  (TIME) Strophe connected:	 3637.4199999845587
Logger.js:125 [modules/xmpp/xmpp.js] <t.value>:  My Jabber ID: qwxmghfdahsefsqz@meet.jitsi/IF0muobd
Logger.js:125 [conference.js] <>:  initialized with 1 local tracks
Logger.js:125 [modules/xmpp/ChatRoom.js] <new t>:  Joined MUC as allinsectsvanishdeliberately@muc.meet.jitsi/qwxmghfd
Logger.js:125 [modules/e2eping/e2eping.js] <new e>:  Initializing e2e ping; pingInterval=10000, analyticsInterval=60000.
Logger.js:125 [modules/connectivity/ParticipantConnectionStatus.js] <new e>:  RtcMuteTimeout set to: 2000
Logger.js:125 [modules/statistics/AvgRTPStatsReporter.js] <new e>:  Avg RTP stats will be calculated every 15 samples
Logger.js:125 [JitsiConference.js] <new X>:  backToP2PDelay: 5
Logger.js:125 [modules/UI/videolayout/VideoLayout.js] <Object.electLastVisibleVideo>:  Last visible video no longer exists
Logger.js:125 [modules/UI/videolayout/VideoLayout.js] <Object.electLastVisibleVideo>:  Fallback to local video...
Logger.js:125 [modules/UI/videolayout/VideoLayout.js] <Object.electLastVisibleVideo>:  electLastVisibleVideo: qwxmghfd
Logger.js:125 [JitsiConference.js] <X._doReplaceTrack>:  _doReplaceTrack - no JVB JingleSession
Logger.js:125 [JitsiConference.js] <X._doReplaceTrack>:  _doReplaceTrack - no P2P JingleSession
Logger.js:125 [modules/remotecontrol/RemoteControl.js] <t.value>:  Initializing remote control.
Logger.js:125 [modules/xmpp/moderator.js] <d.setFocusUserJid>:  Focus jid set to:  focus@auth.meet.jitsi
Logger.js:125 [modules/xmpp/moderator.js] <d.createConferenceIq>:  Session ID: null machine UID: e5136b270783a21859c2bd7600c35cbd
Logger.js:125 [react/features/base/tracks/actions.js] <>:  Replace video track - unmuted
Logger.js:125 [modules/xmpp/strophe.ping.js] <s.value>:  XMPP pings will be sent every 10000 ms
Logger.js:125 [modules/xmpp/moderator.js] <d.parseConfigOptions>:  Authentication enabled: false
Logger.js:125 [modules/xmpp/moderator.js] <d.parseConfigOptions>:  External authentication enabled: false
Logger.js:125 [modules/xmpp/moderator.js] <d.parseConfigOptions>:  Sip gateway enabled:  true
Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  entered allinsectsvanishdeliberately@muc.meet.jitsi/focus {affiliation: "owner", role: "moderator", jid: "focus@auth.meet.jitsi/focus306981126817385", isFocus: true, isHiddenDomain: false}
Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  Ignore focus: allinsectsvanishdeliberately@muc.meet.jitsi/focus, real JID: focus@auth.meet.jitsi/focus306981126817385
Logger.js:125 [modules/version/ComponentsVersions.js] <>:  Got xmpp version: Prosody(0.11.2,Linux)
Logger.js:125 [modules/version/ComponentsVersions.js] <>:  Got focus version: JiCoFo(1.0.1.0-457,Linux)
Logger.js:125 [conference.js] <r.<anonymous>>:  My role changed, new role: none
Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  (TIME) MUC joined:	 4089.6999999822583
Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  Subject is changed to 
Logger.js:125 [conference.js] <r.<anonymous>>:  My role changed, new role: moderator
Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  Ignore focus: allinsectsvanishdeliberately@muc.meet.jitsi/focus, real JID: focus@auth.meet.jitsi/focus306981126817385
Logger.js:125 [react/features/base/storage/PersistenceRegistry.js] <e.value>:  redux state persisted. 5efacf1626d578a30628c11172372175 -> 1c610eecd31d103d009ac337c6f8ec23
Logger.js:125 [modules/UI/videolayout/LargeVideoManager.js] <>:  hover in %s qwxmghfd
Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  entered allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j {affiliation: "none", role: "participant", jid: "9dlxku5jdwtujq-e@meet.jitsi/Zt8X7sQc", isFocus: false, isHiddenDomain: false, …}
Logger.js:125 [conference.js] <r.<anonymous>>:  USER 9dlxku5j connnected: e {_jid: "allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j", _id: "9dlxku5j", _conference: X, _displayName: undefined, _supportsDTMF: false, …}
Logger.js:125 [JitsiConference.js] <X._maybeStartOrStopP2P>:  Will start P2P with: allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j
Logger.js:125 [JitsiConference.js] <X._startP2PSession>:  Created new P2P JingleSession allinsectsvanishdeliberately@muc.meet.jitsi/qwxmghfd allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j
index.js:44 SdpSimulcast: using 3 layers
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <new w>:  Create new TPC[1,p2p:true]
Logger.js:125 [JitsiConference.js] <X._startP2PSession>:  Starting CallStats for P2P connection...
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w.addTrack>:  add LocalTrack[4,video] to: TPC[1,p2p:true]
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[1,p2p:true] sdp-consistency caching primary ssrc439603136
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._processLocalSSRCsMap>:  Storing new local SSRC for LocalTrack[4,video] in TPC[1,p2p:true] {ssrcs: Array(2), groups: Array(1), msid: "c78b755a-d1bd-4b39-94fd-17f3576a0027 0eaf5bbd-6072-4581-967d-fa669739c6f9"}
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Session-initiate:  <iq to=​"allinsectsvanishdeliberately@muc.meet.jitsi/​9dlxku5j" type=​"set" xmlns=​"jabber:​client" id=​"24410362-0948-4318-8c08-0f3ad28cc1e3:​sendIQ">​…​</iq>​
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  sendIceCandidates (2) [RTCIceCandidate, RTCIceCandidate]
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  sendIceCandidates (3) [RTCIceCandidate, RTCIceCandidate, RTCIceCandidate]
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  sendIceCandidate: last candidate.
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  sendIceCandidates (3) [RTCIceCandidate, RTCIceCandidate, RTCIceCandidate]
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "xvtW"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
3Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  Ignore focus: allinsectsvanishdeliberately@muc.meet.jitsi/focus, real JID: focus@auth.meet.jitsi/focus306981126817385
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <w.peerconnection.oniceconnectionstatechange>:  (TIME) ICE checking P2P? true:	 18193.249999982072
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  on jingle session-initiate from allinsectsvanishdeliberately@muc.meet.jitsi/focus <iq xmlns=​"jabber:​client" from=​"allinsectsvanishdeliberately@muc.meet.jitsi/​focus" id=​"cXd4bWdoZmRhaHNlZnNxekBtZWV0LmppdHNpL0lGMG11b2JkAG0wTjB1LTI1MAB7/​DcW/​nCpS3tKz2ToZEgW" to=​"qwxmghfdahsefsqz@meet.jitsi/​IF0muobd" type=​"set">​…​</iq>​
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  (TIME) received session-initiate:	 18301.04500000016
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  Marking session from allinsectsvanishdeliberately@muc.meet.jitsi/focus as *not* P2P
index.js:44 SdpSimulcast: using 3 layers
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <new w>:  Create new TPC[2,p2p:false]
Logger.js:125 [JitsiConference.js] <X._acceptJvbIncomingCall>:  Starting CallStats for JVB connection...
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w.addTrack>:  add LocalTrack[4,video] to: TPC[2,p2p:false]
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._remoteStreamAdded>:  TPC[2,p2p:false] ignored remote 'stream added' event for non-user streamid: mixedmslabel
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[2,p2p:false] sdp-consistency caching primary ssrc3644189216
index.js:404 SdpSimulcast: current ssrc cache:  []
index.js:405 SdpSimulcast: parsed primary ssrc 3644189216
index.js:414 SdpSimulcast: Have not seen primary ssrc before, generating source data
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._processLocalSSRCsMap>:  Storing new local SSRC for LocalTrack[4,video] in TPC[2,p2p:false] {ssrcs: Array(6), groups: Array(4), msid: "c78b755a-d1bd-4b39-94fd-17f3576a0027 0eaf5bbd-6072-4581-967d-fa669739c6f9"}
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Sending session-accept <iq to=​"allinsectsvanishdeliberately@muc.meet.jitsi/​focus" type=​"set" xmlns=​"jabber:​client" id=​"75cee44a-a8ac-4d3f-9b9f-ee3a004507af:​sendIQ">​…​</iq>​
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <w.peerconnection.oniceconnectionstatechange>:  (TIME) ICE checking P2P? false:	 18333.50499998778
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "e+YY"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  sendIceCandidates [RTCIceCandidate]
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "e+YY"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "e+YY"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  sendIceCandidate: last candidate.
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  sendIceCandidates [RTCIceCandidate]
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "ufrag" = "e+YY"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-id" = "1"
Logger.js:125 [modules/xmpp/SDPUtil.js] <Object.candidateToJingle>:  not translating "network-cost" = "10"
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <>:  Got RESULT for "session-initiate"
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  on jingle session-accept from allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j <iq xmlns=​"jabber:​client" from=​"allinsectsvanishdeliberately@muc.meet.jitsi/​9dlxku5j" id=​"cXd4bWdoZmRhaHNlZnNxekBtZWV0LmppdHNpL0lGMG11b2JkADgyNzQ3NGQ0LTAzMGQtNGYxMC1hMjYzLTA0NWZiZjYzODNkYTpzZW5kSVEAtE0UDfzSIzk8k4PEe7GzZg==" to=​"qwxmghfdahsefsqz@meet.jitsi/​IF0muobd" type=​"set">​…​</iq>​
Logger.js:125 [JitsiConference.js] <X.onCallAccepted>:  P2P setAnswer
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  on jingle transport-info from allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j <iq xmlns=​"jabber:​client" from=​"allinsectsvanishdeliberately@muc.meet.jitsi/​9dlxku5j" id=​"cXd4bWdoZmRhaHNlZnNxekBtZWV0LmppdHNpL0lGMG11b2JkAGUwMGI5Mjc0LWE3ZDItNDlhZS1iZjg3LTQ4MmY1NzY2YWI4ZTpzZW5kSVEAtE0UDfzSIzk8k4PEe7GzZg==" to=​"qwxmghfdahsefsqz@meet.jitsi/​IF0muobd" type=​"set">​…​</iq>​
Logger.js:125 [JitsiConference.js] <X.onTransportInfo>:  P2P addIceCandidates
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <w.peerconnection.oniceconnectionstatechange>:  (TIME) ICE connected P2P? true:	 18765.05499999621
Logger.js:125 [JitsiConference.js] <X._setP2PStatus>:  Peer to peer connection established!
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Queued make video active, audio active task...
Logger.js:125 [JitsiConference.js] <X._suspendMediaTransferForJvbConnection>:  Suspending media transfer over the JVB connection...
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Queued make video inactive, audio inactive task...
Logger.js:125 [JitsiConference.js] <X._onIceConnectionEstablished>:  Starting remote stats with p2p connection
Logger.js:125 [modules/statistics/statistics.js] <Function.b.sendAnalyticsAndLog>:  {"type":"operational","action":"established","source":"p2p","attributes":{"initiator":true}}
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._remoteStreamAdded>:  TPC[1,p2p:true] ignored remote 'stream added' event for non-user streamid: default
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <w.peerconnection.oniceconnectionstatechange>:  (TIME) ICE completed P2P? true:	 18801.99000000721
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[1,p2p:true] sdp-consistency replacing new ssrc439603136 with cached 439603136
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[2,p2p:false] sdp-consistency replacing new ssrc3644189216 with cached 3644189216
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._adjustLocalMediaDirection>:  Adjusted local audio direction to inactive
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._adjustLocalMediaDirection>:  Adjusted local video direction to inactive
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  removal not necessary
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  addition not necessary
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <>:  setAnswer - succeeded
Logger.js:125 [JitsiConference.js] <>:  Suspended media transfer over the JVB connection !
Logger.js:125 [JitsiConference.js] <X._suspendMediaTransferForJvbConnection>:  Suspending media transfer over the JVB connection...
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Queued make video inactive, audio inactive task...
Logger.js:125 [JitsiConference.js] <>:  Suspended media transfer over the JVB connection !
Logger.js:125 [modules/RTC/RTCUtils.js] <t.value>:  Using the new gUM flow
Logger.js:125 [modules/UI/videolayout/LargeVideoManager.js] <>:  hover in %s 9dlxku5j
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w.addTrack>:  add LocalTrack[5,video] to: TPC[2,p2p:false]
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w.addTrack>:  add LocalTrack[5,video] to: TPC[1,p2p:true]
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[1,p2p:true] sdp-consistency replacing new ssrc2688381908 with cached 439603136
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._processLocalSSRCsMap>:  Storing new local SSRC for LocalTrack[5,video] in TPC[1,p2p:true] {ssrcs: Array(2), groups: Array(1), msid: "l9GoiiO6fNmuCpAXVYshmmGAkH5C4DmHjBr5 b9f57a93-9c51-4946-ab75-88f171fcd290"}
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[2,p2p:false] sdp-consistency replacing new ssrc2040690420 with cached 3644189216
index.js:404 SdpSimulcast: current ssrc cache:  (3) [3644189216, 1443044054, 1498902839]
index.js:405 SdpSimulcast: parsed primary ssrc 3644189216
index.js:410 SdpSimulcast: Have seen primary ssrc before, filling in data from cache
index.js:282 SdpSimulcast restoring from cache:  (3) [3644189216, 1443044054, 1498902839]
index.js:284 SdpSimulcast Parsed new sim ssrcs:  [3644189216]
index.js:288 SdpSimulcast built replacement map:  {3644189216: 3644189216}
index.js:293 SdpSimulcast built ssrcs to add:  (2) [1443044054, 1498902839]
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._processLocalSSRCsMap>:  Storing new local SSRC for LocalTrack[5,video] in TPC[2,p2p:false] {ssrcs: Array(6), groups: Array(4), msid: "l9GoiiO6fNmuCpAXVYshmmGAkH5C4DmHjBr5 b9f57a93-9c51-4946-ab75-88f171fcd290"}
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._adjustLocalMediaDirection>:  Adjusted local audio direction to inactive
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._adjustLocalMediaDirection>:  Adjusted local video direction to inactive
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w._remoteStreamRemoved>:  Ignored remote 'stream removed' event for non-user stream default
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  removal not necessary
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  addition not necessary
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <Object.callback>:  Replace track done!
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  removal not necessary
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  addition not necessary
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <Object.callback>:  Replace track done!
Logger.js:125 [react/features/base/tracks/actions.js] <>:  Replace video track - unmuted
Logger.js:125 [modules/UI/videolayout/VideoLayout.js] <Object.onVideoTypeChanged>:  Peer video type changed:  qwxmghfd desktop
Logger.js:125 [conference.js] <>:  Screen sharing started
Logger.js:125 [JitsiConference.js] <e.sendMessage>:  Failed to send a ping request or response.
i @ Logger.js:125
(anonymous) @ JitsiConference.js:271
value @ e2eping.js:92
setInterval (async)
e @ e2eping.js:62
value @ e2eping.js:298
r.emit @ events.js:96
X.onMemberJoined @ JitsiConference.js:1315
r.emit @ events.js:89
value @ ChatRoom.js:465
value @ strophe.emuc.js:103
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5829
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
sendIQ @ strophe.js:3411
value @ strophe.ping.js:79
(anonymous) @ strophe.ping.js:101
setInterval (async)
value @ strophe.ping.js:100
(anonymous) @ xmpp.js:190
Promise.then (async)
value @ xmpp.js:187
_changeConnectStatus @ strophe.js:3683
_sasl_session_cb @ strophe.js:4305
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
_sasl_bind_cb @ strophe.js:4275
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
_sasl_auth1_cb @ strophe.js:4234
o @ strophe.js:4181
(anonymous) @ strophe.js:4185
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:3439
setTimeout (async)
_sendRestart @ strophe.js:3438
_sasl_success_cb @ strophe.js:4192
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
_attemptSASLAuth @ strophe.js:4018
authenticate @ strophe.js:4070
_connect_cb @ strophe.js:3945
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_connect @ strophe.js:5170
connect @ strophe.js:3051
value @ xmpp.js:329
value @ xmpp.js:389
c.connect @ JitsiConnection.js:61
e @ connection.js:38
(anonymous) @ connection.js:149
s @ connection.js:86
c @ connection.js:179
H @ conference.js:151
createInitialLocalTracksAndConnect @ conference.js:680
init @ conference.js:719
(anonymous) @ actions.web.js:28
(anonymous) @ index.js:11
(anonymous) @ middleware.js:41
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:13
(anonymous) @ middleware.js:21
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:22
(anonymous) @ middleware.js:99
(anonymous) @ middleware.js:63
(anonymous) @ middleware.js:43
(anonymous) @ middleware.web.js:36
(anonymous) @ middleware.any.js:55
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:17
(anonymous) @ middleware.js:30
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:39
(anonymous) @ middleware.js:12
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:20
(anonymous) @ middleware.js:25
(anonymous) @ middleware.web.js:23
(anonymous) @ middleware.any.js:94
(anonymous) @ middleware.js:65
(anonymous) @ middleware.js:33
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:29
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:104
(anonymous) @ middleware.js:62
(anonymous) @ middleware.js:24
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:44
(anonymous) @ middleware.js:27
(anonymous) @ middleware.js:23
(anonymous) @ middleware.js:25
(anonymous) @ middleware.js:45
(anonymous) @ middleware.js:42
(anonymous) @ middleware.js:18
(anonymous) @ middleware.js:130
(anonymous) @ middleware.js:126
value @ Conference.js:276
value @ Conference.js:157
Na @ react-dom.production.min.js:228
Ta @ react-dom.production.min.js:220
Pa @ react-dom.production.min.js:219
Oa @ react-dom.production.min.js:216
Zi @ react-dom.production.min.js:214
enqueueSetState @ react-dom.production.min.js:134
w.setState @ react.production.min.js:13
n @ I18n.js:135
t @ I18n.js:147
(anonymous) @ EventEmitter.js:46
e.emit @ EventEmitter.js:45
(anonymous) @ i18next.js:162
(anonymous) @ i18next.js:265
(anonymous) @ i18next.js:281
t.load @ BackendConnector.js:183
(anonymous) @ i18next.js:215
t.load @ CacheConnector.js:40
t.loadResources @ i18next.js:214
r @ i18next.js:280
t.changeLanguage @ i18next.js:286
s @ i18next.js:159
setTimeout (async)
t.init @ i18next.js:171
(anonymous) @ i18next.js:68
n @ bootstrap:19
(anonymous) @ actionTypes.js:21
n @ bootstrap:19
(anonymous) @ actions.js:19
n @ bootstrap:19
(anonymous) @ actions.js:265
(anonymous) @ functions.js:97
n @ bootstrap:19
(anonymous) @ AnalyticsEvents.js:630
n @ bootstrap:19
(anonymous) @ index.js:1
(anonymous) @ actions.js:715
n @ bootstrap:19
(anonymous) @ toConsumableArray.js:8
n @ bootstrap:19
(anonymous) @ actions.js:5
(anonymous) @ actions.js:265
n @ bootstrap:19
(anonymous) @ index.js:3
n @ bootstrap:19
(anonymous) @ actions.js:20
n @ bootstrap:19
(anonymous) @ possibleConstructorReturn.js:16
n @ bootstrap:19
(anonymous) @ actions.js:17
n @ bootstrap:19
(anonymous) @ AuthHandler.js:1
(anonymous) @ AuthHandler.js:229
n @ bootstrap:19
(anonymous) @ connection.js:1
(anonymous) @ connection.js:189
n @ bootstrap:19
(anonymous) @ Tooltip.js:289
(anonymous) @ conference.js:2786
n @ bootstrap:19
(anonymous) @ withAnalyticsEvents.js:111
n @ bootstrap:19
(anonymous) @ bootstrap:83
(anonymous) @ bootstrap:83
Show 121 more frames
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <w.peerconnection.oniceconnectionstatechange>:  (TIME) ICE failed P2P? false:	 33413.354999996955
Logger.js:125 [modules/connectivity/IceFailedNotification.js] <>:  ICE connection restored - not sending ICE failed
Logger.js:125 [modules/xmpp/ChatRoom.js] <t.value>:  entered allinsectsvanishdeliberately@muc.meet.jitsi/wszzkuma {affiliation: "none", role: "participant", jid: "wszzkuma4coo07xz@meet.jitsi/JIbn2ieM", isFocus: false, isHiddenDomain: false, …}
Logger.js:125 [conference.js] <r.<anonymous>>:  USER wszzkuma connnected: e {_jid: "allinsectsvanishdeliberately@muc.meet.jitsi/wszzkuma", _id: "wszzkuma", _conference: X, _displayName: undefined, _supportsDTMF: false, …}
Logger.js:125 [JitsiConference.js] <X._maybeStartOrStopP2P>:  Will stop P2P with: allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j
Logger.js:125 [modules/statistics/statistics.js] <Function.b.sendAnalyticsAndLog>:  {"type":"operational","action":"switch.to.jvb","source":"p2p","attributes":{}}
Logger.js:125 [JitsiConference.js] <X._resumeMediaTransferForJvbConnection>:  Resuming media transfer over the JVB connection...
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Queued make video active, audio active task...
Logger.js:125 [JitsiConference.js] <X._stopP2PSession>:  Stopping remote stats for P2P connection
Logger.js:125 [JitsiConference.js] <X._stopP2PSession>:  Stopping CallStats for P2P connection
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Sending session-terminate <iq to=​"allinsectsvanishdeliberately@muc.meet.jitsi/​9dlxku5j" type=​"set" xmlns=​"jabber:​client" id=​"b8f50a5c-bbf7-4701-be20-bb025b6b2956:​sendIQ">​…​</iq>​
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  Session terminated JingleSessionPC[p2p=true,initiator=true,sid=da425a4e8e61] undefined undefined
Logger.js:125 [modules/RTC/TraceablePeerConnection.js] <w.close>:  Closing TPC[1,p2p:true]...
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  The session has ended - cancelling action: oniceconnectionstatechange
Logger.js:125 [JitsiConference.js] <X._setP2PStatus>:  Peer to peer connection closed!
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[2,p2p:false] sdp-consistency replacing new ssrc3644189216 with cached 3644189216
Logger.js:125 [JitsiConference.js] <>:  Resumed media transfer over the JVB connection!
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  on jingle session-terminate from allinsectsvanishdeliberately@muc.meet.jitsi/9dlxku5j <iq xmlns=​"jabber:​client" from=​"allinsectsvanishdeliberately@muc.meet.jitsi/​9dlxku5j" id=​"cXd4bWdoZmRhaHNlZnNxekBtZWV0LmppdHNpL0lGMG11b2JkADc1YWQyZWQ0LWFkN2ItNDBlMy05ZGM2LTBhZmNlZTc4YTljYjpzZW5kSVEAtE0UDfzSIzk8k4PEe7GzZg==" to=​"qwxmghfdahsefsqz@meet.jitsi/​IF0muobd" type=​"set">​…​</iq>​
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  invalid session id <iq xmlns=​"jabber:​client" from=​"allinsectsvanishdeliberately@muc.meet.jitsi/​9dlxku5j" id=​"cXd4bWdoZmRhaHNlZnNxekBtZWV0LmppdHNpL0lGMG11b2JkADc1YWQyZWQ0LWFkN2ItNDBlMy05ZGM2LTBhZmNlZTc4YTljYjpzZW5kSVEAtE0UDfzSIzk8k4PEe7GzZg==" to=​"qwxmghfdahsefsqz@meet.jitsi/​IF0muobd" type=​"set">​…​</iq>​
i @ Logger.js:125
value @ strophe.jingle.js:88
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
sendIQ @ strophe.js:3411
value @ JingleSessionPC.js:1301
X._stopP2PSession @ JitsiConference.js:2928
X._maybeStartOrStopP2P @ JitsiConference.js:2889
X.onMemberJoined @ JitsiConference.js:1326
r.emit @ events.js:89
value @ ChatRoom.js:465
value @ strophe.emuc.js:103
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5829
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:4473
setTimeout (async)
_onIdle @ strophe.js:4472
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
sendIQ @ strophe.js:3411
value @ strophe.ping.js:79
(anonymous) @ strophe.ping.js:101
setInterval (async)
value @ strophe.ping.js:100
(anonymous) @ xmpp.js:190
Promise.then (async)
value @ xmpp.js:187
_changeConnectStatus @ strophe.js:3683
_sasl_session_cb @ strophe.js:4305
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
_sasl_bind_cb @ strophe.js:4275
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
_sasl_auth1_cb @ strophe.js:4234
o @ strophe.js:4181
(anonymous) @ strophe.js:4185
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:3439
setTimeout (async)
_sendRestart @ strophe.js:3438
_sasl_success_cb @ strophe.js:4192
run @ strophe.js:2556
(anonymous) @ strophe.js:3822
forEachChild @ strophe.js:1522
_dataRecv @ strophe.js:3810
_onRequestStateChange @ strophe.js:5559
XMLHttpRequest.send (async)
l @ strophe.js:5662
_processRequest @ strophe.js:5677
_throttledRequestHandler @ strophe.js:5823
_onIdle @ strophe.js:5455
_onIdle @ strophe.js:4467
(anonymous) @ strophe.js:5789
setTimeout (async)
_send @ strophe.js:5788
send @ strophe.js:3270
_attemptSASLAuth @ strophe.js:4018
authenticate @ strophe.js:4070
_connect_cb @ strophe.js:3945
_onRequestStateChange @ strophe.js:5559
Logger.js:125 [modules/xmpp/strophe.jingle.js] <s.value>:  on jingle source-add from allinsectsvanishdeliberately@muc.meet.jitsi/focus <iq xmlns=​"jabber:​client" from=​"allinsectsvanishdeliberately@muc.meet.jitsi/​focus" id=​"cXd4bWdoZmRhaHNlZnNxekBtZWV0LmppdHNpL0lGMG11b2JkAG0wTjB1LTI5OQB7/​DcW/​nCpS3tKz2ToZEgW" to=​"qwxmghfdahsefsqz@meet.jitsi/​IF0muobd" type=​"set">​…​</iq>​
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <>:  Processing addRemoteStream
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <>:  ICE connection state:  failed
index.js:146 Halt: There are no SSRC groups in the remote description.
Logger.js:125 [modules/xmpp/SdpConsistency.js] <e.value>:  TPC[2,p2p:false] sdp-consistency replacing new ssrc3644189216 with cached 3644189216
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <>:  addRemoteStream - OK, SDPs:  o {media: Array(3), raw: "v=0
↵o=- 4807607192767580361 5 IN IP4 127.0.0.1
↵s…trickle
↵a=sctpmap:5000 webrtc-datachannel 1024
↵", session: "v=0
↵o=- 4807607192767580361 5 IN IP4 127.0.0.1
↵s…GAkH5C4DmHjBr5
↵a=group:BUNDLE audio video data
↵"} o {media: Array(3), raw: "v=0
↵o=- 4807607192767580361 6 IN IP4 127.0.0.1
↵s…trickle
↵a=sctpmap:5000 webrtc-datachannel 1024
↵", session: "v=0
↵o=- 4807607192767580361 6 IN IP4 127.0.0.1
↵s…GAkH5C4DmHjBr5
↵a=group:BUNDLE audio video data
↵"}
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  removal not necessary
Logger.js:125 [modules/xmpp/JingleSessionPC.js] <t.value>:  addition not necessary
```

What can now be done to fix this?
I'm not quite sure where the error arises from.

## Additional info:

Regading the Docker Jitsi, we are using a rather old image from around February 2019 ( https://hub.docker.com/r/talsenteam/docker-jitsi-jicofo/tags ; https://hub.docker.com/r/talsenteam/docker-jitsi-jvb/tags ; https://hub.docker.com/r/talsenteam/docker-jitsi-prosody/tags ; https://hub.docker.com/r/talsenteam/docker-jitsi-web/tags ) -> tag v2019-02.

Furthermore when Docker Jitsi is served directly under sub.example.com with the same configuration ( except changes made in **.jitsi-meet-cfg/web/config.js** ) it works fine, even for three or more users.
