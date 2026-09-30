---
date: '2026-09-29T21:35:44-06:00'
draft: false
title: 'nginx round robin'
tags: ["nginx", "server"]
categories: ["system design"]
videos:
    - videos/round-robin.webm
---
This is my first post.

This is a demo of round robin.

For this demo, I am going to create three http servers locally on Ubuntu. I will then put nginx, reverse proxy in front of the servers and we will see how nginx routes the traffic.

Let me fire up my terminal multiplexer.

We have four sessions here. I will run three servers in three separate panes

```
python3 -m http.server 8001
```

```
python3 -m http.server 8002
```

```
python3 -m http.server 8003
```

I am going to set up the nginx config at `/etc/nginx/conf.d/demo.conf`
```
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

Lets start nginx.
```
systemctl start service nginx
```

Lets hit the nginx endpoint.

Notice that we want to hit the localhost with a delay of one second to see how round robin works.
```
for i in {1..100}; do
  curl -s -o /dev/null -w "Request $i: HTTP %{http_code}\n" http://localhost
  sleep 1
done

```

The request got routed to the server listening on port ABCXYZ

Lets stop one of the servers.

You can see how the requests are getting routed to the other two servers.

Lets bring back the downed server on the same port.

See how the requests resumed routing in round robin again.



{{< video src="/videos/round-robin.webm" >}}
