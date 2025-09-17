# Deploying the Zinc server demo

Sept 2, 2025

The page at [https://zn.stfx.eu](https://zn.stfx.eu) has an extra paragraph with a summary
of how the Zinc Components' server demo is set up.
I want to explain some of the aspects that are involved.

The first step is to take a stock Pharo image and load the necessary code.
In this case, the latest version of the 
[upstream Zinc repository](https://github.com/svenvc/zinc) is loaded
along with some support packages like [NeoConsole](https://github.com/svenvc/NeoConsole).
Although there are different ways to go about it, 
you basically use Metacello to load baselines of the projects you need.

The next step is to configure and start our production image with our demo.
I prefer to have a production image in which nothing is running or started automatically.
Instead I use a startup run script that is executed by the production image.
Here is the one used for the demo:

```smalltalk
| staticFileServerDelegate |

(DailyNonInteractiveTranscript onFileNamed: '/home/stfx/pharo13/zn-server-{1}.log') install.

Transcript
  cr;
  show: 'Starting '; show: (NeoConsoleTelnetServer startOn: 42031); cr;
  show: 'Starting '; show: (NeoConsoleMetricDelegate startOn: 42032); cr;
  flush.

(ZnServer defaultOn: 8180)
  logToTranscript;
  logLevel: 1;
  start.

(staticFileServerDelegate := ZnStaticFileServerDelegate new)
  prefixFromString: 'zn'; 
  directory: '/home/stfx/zn' asFileReference.

ZnServer default delegate prefixMap 
  at: 'zn' 
  put: [ :request | staticFileServerDelegate handleRequest: request ].

ZnServer default delegate extraWelcomeText: 'This is a live cloud instance of Pharo Smalltalk running a Zinc HTTP Components server with the default delegate. More specifically, the cloud hardware is a 1 GB, 1 vCPU Digital Ocean basic droplet, the OS is GNU Linux Ubuntu Server 24.04 LTS, with nginx in front of the Pharo Smalltalk image to do TLS/SSL termination and HTTP/1.1 to HTTP/2 conversion.'.
```

The first expression redirects all Transcript output to a log file that is created per day.
The NeoConsole servers make the image accessible while it is running, by means of a REPL and metrics.
This means we can go peek inside the running image, all without a graphical UI, just using a command line.

By asking the server to logToTranscript with a certain logLevel we get pretty standard output like so:

```console
2025-09-02 16:23:27 109 078757 GET / 200 1437B 14ms
2025-09-02 16:23:28 110 188471 GET /zn/small.html 200 113B 0ms
2025-09-02 16:23:29 111 125899 GET /zn/small.html 200 113B 0ms
2025-09-02 16:23:29 112 713623 GET /zn/small.html 200 113B 0ms
2025-09-02 16:23:52 113 845899 HEAD /zn/small.html 200 21ms
2025-09-02 16:23:54 114 618028 GET /zn/Hot-Air-Balloon.gif 200 25119B 0ms
```

The default server is used almost completely in its out of the box configuration.
A specific port is used (8180) and a static file server is installed under the `/zn` prefix.
Some extra welcome text is added as well.

Nowadays, a production HTTP server needs to implement HTTPS and ideally HTTP/2.
We can do that by running [nginx](https://nginx.org) in front of our backend server.
Nginx will proxy our backend server, translating from an external request on port 443 to 
an internal request on port 8180 - a port not accessible by the outside world.

This takes care of two important requirements: TLS/HTTPS termination using well known tools
like [Let's Encrypt](https://letsencrypt.org) and HTTP/1.1 to HTTP/2 conversion.
Furthermore, nginx can protect us from malformed requests and other attacks.

Additionally, it would be possible to do load balancing and fail over across 
multiple Pharo instances, but that is another, interesting story.

The third step is to extend the nginx configuration.
Here is the file for the zn.stfx.eu demo server:

```nginx
server {
  server_name zn.stfx.eu;

  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Remote-Address $remote_addr;
    proxy_set_header Connection "";
    proxy_http_version 1.1;
    proxy_pass http://localhost:8180;
    add_header X-Server '$upstream_http_server';
  }

  listen 443 ssl http2; # managed by Certbot
  ssl_certificate /etc/letsencrypt/live/zn.stfx.eu/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/zn.stfx.eu/privkey.pem; # managed by Certbot
  include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
  if ($host = zn.stfx.eu) {
    return 301 https://$host$request_uri;
  } # managed by Certbot

  listen 80;
  server_name zn.stfx.eu;
  return 404; # managed by Certbot
}
```

Skipping the `location` section, this is all pretty standard.
Old HTTP traffic on port 80 is redirected to HTTPS on port 443.
Let's Encrypt's Certbot did and does all the hard work with the certificates.

One crucial addition is the `http2` option added to the `listen 443 ssl http2;` directive.
This enables the protocol translation.

The `location` section sets up a proxy. It says that all traffic should be sent to the server
running locally on port 8180, i.e. our Zinc server demo running in Pharo.
It manipulates a couple of headers going in and coming out of the backend,
mostly for cosmetic reasons.

These make little functional difference except for 2 of them:

```nginx
proxy_set_header Connection "";
proxy_http_version 1.1;
```

They make sure that connections between the front end nginx and backend zinc server are kept alive,
which should improve performance.
State management of these connections is quite complex, depending on traffic,
but let's assume that nginx is battle tested in this respect.

The result of all this is that a standard tool like curl shows us that all is OK
with respect to security and HTTP/2, as can be seen from this interaction:

```console
$ curl -v https://zn.stfx.eu/small                   

* Host zn.stfx.eu:443 was resolved.
* IPv6: (none)
* IPv4: 146.185.177.20
*   Trying 146.185.177.20:443...
* Connected to zn.stfx.eu (146.185.177.20) port 443
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-AES256-GCM-SHA384 / [blank] / UNDEF
* ALPN: server accepted h2
* Server certificate:
*  subject: CN=zn.stfx.eu
*  start date: Aug 25 17:10:24 2025 GMT
*  expire date: Nov 23 17:10:23 2025 GMT
*  subjectAltName: host "zn.stfx.eu" matched cert's "zn.stfx.eu"
*  issuer: C=US; O=Let's Encrypt; CN=E8
*  SSL certificate verify ok.
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://zn.stfx.eu/small
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: zn.stfx.eu]
* [HTTP/2] [1] [:path: /small]
* [HTTP/2] [1] [user-agent: curl/8.7.1]
* [HTTP/2] [1] [accept: */*]
> GET /small HTTP/2
> Host: zn.stfx.eu
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/2 200 
< server: nginx/1.24.0 (Ubuntu)
< date: Tue, 02 Sep 2025 15:06:01 GMT
< content-type: text/html;charset=utf-8
< content-length: 521
< x-server: Zinc HTTP Components 2.0 (Pharo/13.1)
< 
* Connection #0 to host zn.stfx.eu left intact
<!DOCTYPE html><html><head><title>Small</title><style type="text/css">body { color: black; background: white; width: 900px; font-family: Verdana, Arial, Helvetica, sans-serif; font-size: 16px } p { width: 800px; padding: 0 20px 10px 20px } ul, ol { width: 800px; padding: 0 5px 5px 30px } #logo { color: orange; font-family: Helvetica, sans-serif; font-weight: bold; font-size: 128px; text-decoration: none }</style></head><body><a id="logo" href="/">Zn</a><h1>Small</h1><p>This is a small HTML document</p></body></html>
```

From the server and x-server headers in the response, we can confirm our frontend/backend setup.

