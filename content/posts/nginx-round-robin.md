---
date: '2026-09-29T21:35:44-06:00'
draft: false
title: 'Round Robin with nginx'
tags: ["nginx", "server", "round robin"]
categories: ["system design"]
videos:
    - videos/round-robin.webm
---

This is a simple demo of a round robin reverse proxy. Round robin is a fundamental concept in system design. I will show how this concept can be demonstrated locally.

I am going to create three http servers locally on Ubuntu. I will then put nginx reverse proxy in front of the servers and we will see how the reverse proxy routes the traffic.

Lets fire up a terminal multiplexer. We will have four sessions here. I will run three servers in three separate panes

```bash
python3 -m http.server 8001
```

```bash
python3 -m http.server 8002
```

```bash
python3 -m http.server 8003
```
{{< figure src="/images/round-robin-00.png" alt="tmux" >}}


I am now going to set up the nginx config at `/etc/nginx/conf.d/demo.conf`
```bash
upstream demo {
    server 127.0.0.1:8001;  # App 1
    server 127.0.0.1:8002;  # App 2
    server 127.0.0.1:8003;  # App 3
}

server {
    listen 80;

    location / {
        proxy_pass http://demo;
    }
}

```

For this demo, I am assuming that nginx is already installed and nginx picked up this config without issues. Lets start nginx.
```
systemctl start service nginx
```

Lets hit the nginx endpoint.

Notice that we want to hit the localhost with a delay of one second after each request to see how round robin works.
```bash
for i in {1..100}; do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code}\n" http://localhost
  sleep 1
done

```


{{< video src="/videos/round-robin.webm" >}}

Notice in the above video how the requests intially got routed to a different server in a cyclic manner each time. At around 0:20 mark we terminated one of the servers (the one on port 8002) to simulate a downed server. When this happened, the reverse proxy kept routing to the other two servers.

We then started the server again around 0:32.

Notice how the reverse proxy resumed sending the requests to the server listening on port 8002.

