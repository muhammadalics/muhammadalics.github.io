---
date: '2026-10-01T23:10:03-06:00'
draft: false
title: 'Canary Deployments, Weighted Round Robin and Stickiness'
tags: ["nginx", "server", "weighted round robin", "canary"]
---

Lets test a canary deployment using nginx weighted round robin strategy. Canary deployment is a pattern where a new version of a service is rolled out to a subset of users to make sure that the new version works as intended.

Lets say we have an exisiting deployment and we would like to route about 20% of the requests to a canary while the existing deployment gets 80% of the requests. We can specify the weights in the nginx reverse proxy config. See highlighted lines below.

```bash {hl_lines=["2-3", "11"]}
upstream demo {
    server 127.0.0.1:8001 weight=4;  # main
    server 127.0.0.1:8002 weight=1;  # canary
}

server {
    listen 80 default_server;

    location / {
        proxy_pass http://demo;
	    add_header X-Deployment-Target $upstream_variant;
    }
}

```

There are two things to notice in this config: 
- the weight distribution and 
- the line `add_header X-Deployment-Target $upstream_variant;` 

Out of every 5 requests, four (80%) will get routed to the server listening on port 8001 and only 1 (20%) will be routed to the canary deployment. 

The `add_header` directive adds the `X-Deployment-Target` header in the response that specifies what server actually served the caller's request. This is useful for monitoring the performance of the canary deployment.

Lets send 100 curl requests and grep for the port number in the response. The result is piped into sort and then uniq to count the number of responses from each server.

```bash
for i in {1..100}; do curl -sI http://localhost | grep -o '800[12]'; done | sort | uniq -c
```
{{< video src="/videos/canary.webm" >}}


Notice that roughly 80% of the requests are getting routed to the original server and 20% to the canary.

It is important to call out here that sticky sessions are important. When requests from a client are being directed to one server, we should make sure that the subsequent requests from the same client keep getting directed to the same server. This is acheived by adding the hash directive in nginx config. See the line highlighted below.

```bash {hl_lines=["4"]}
upstream demo {
    server 127.0.0.1:8001 weight=4;  # App 1
    server 127.0.0.1:8002 weight=1;  # App 2
    hash $cookie_route_session consistent;

}

server {
    listen 80 default_server;

    location / {
        proxy_pass http://demo;
	    add_header X-Deployment-Target $upstream_addr;
	
    }
}
```
I set the `route_session` with the curl command and send 100 request to simulate all requests coming from the same client. All of those get routed to just one server. This is because all 100 requests have the same cookie.
```bash
for i in {1..100}; do curl --cookie "route_session=user982734" -sI http://localhost | grep -o '800[12]'; done | sort | uniq -c
```
{{< video src="/videos/sticky-00.webm" >}}

Lets see what happens if cookie in each of those 100 requests is unique. Lets generate a random cookie with `openssl rand -hex 8`.

```bash
for i in {1..100}; do curl --cookie "route_session=$(openssl rand -hex 8)" -sI http://localhost | grep -o '800[12]'; done | sort | uniq -c
```

{{< video src="/videos/sticky-01.webm" >}}

As you can see the requests get split (80/20) based on the weights we set in the config. This is because each cookie is unique representing a unique client.