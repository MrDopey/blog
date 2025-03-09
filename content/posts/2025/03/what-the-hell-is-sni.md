--- 
date: 2025-03-09T10:52:25Z
title: 'What The Hell Is Sni'
description: ""
slug: ''
authors: []
tags: [ 'server-name-indication', 'nginx', 'cross-origin-resource-sharing', 'claude', 'llm-coding' ]
categories: []
externalLink: ""
series: []
---


## The problem

I have an nginx proxy running, I am targeting a url. The actual endpoint I can't mention, but for all intents and purposes, I will call it MyAPI here.

Snippet of config.

```nginx
location /my-api {
    rewrite    /my-api/(.+) /api/$1 break;
    proxy_pass ${MY_API_URL};
}
```

I do not have control of MyAPI, but I do have an account and can authenticate and interact with it.

I have crafted a postman request, that is working flawlessly with said API, and I replicate that exactly in my frontend application. However due to CORS, I cannot call the API directly, so I create a reverse proxy to send the request instead.

Simple, yes?
WRONG

Error text from Nginx

```text
2025/03/09 11:43:14 [error] 45#45: *2 upstream timed out (110: Operation timed out) while SSL handshaking to upstream, client: 127.0.0.1, server: , request: "GET /my-api HTTP/1.1", upstream: "https://ip-address-of-my-api:443/my-api", host: "localhost:2000"
```

This error also shows up when I change the url in postman to target my proxy, so it's not a matter of my frontend logic failing.

## Where I went wrong

### Red herring

I'm not a network engineer, but when I read that error, I think there's something wrong with the SSL handshake, so I hop into my docker container that is running my nginx server and probe the dns using `dig` and other limited networking tools I know on hand. It was weird, there's something funky with the record because I could sometimes get an answer, but I don't understand what I'm reading.

```text
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 33207
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
```

Looks like it doesn't exist? Which is a weird statement because `postman` and `curl` commands are working. Something else is at play.

### Claude to the rescue?

At work we were given a shiny new tool called Claude. Why not probe this magical black box for answers?

The following is the list of things Claude suggested I tried over many prompts

Note: I illustrate this because I am ignorant of this topic, and I am using Claude as an SME / brainstorming.

#### Set the X-Real-IP

Insisted that I use

```nginx
proxy_set_header X-Real-IP $remote_addr
```

It did not work

#### Set the resolver

Suggested using a resolver <https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver>

Which delays the resolution of the DNS, but doesn't actually help with resolving my error.

#### docker networking

Suggested creating a docker network and getting that to resolve my IP. I don't even.
It doesn't work.

#### Set the backend

```nginx
set $backend "http://backend.example.com"
```

It didn't work.

#### Kitchen sink of suggestions

When I asked Claude to focus on the dam problem, it listed the following

Suggestion

- Use unix sockets
- Use network level solutions like `iptables`
- Application level changes to the backend of my proxy to accept my IP
- Reverse DNS 
- Custom TCP proxy

Then suggested some alternative application that would solve my problem

- HAProxy
- Use Traefik
- Ues Envoy
- Apache with mod_proxy
- Caddy
- sslh
- socat

#### Being explicit what I wanted Claude to ask

Prompt
> Can nginx perform the handshake based on the domain name rather then the ip

Abbreviated answer

> The short answer is no, Nginx cannot perform the TCP handshake based on the domain name rather than the IP address. This is due to fundamental networking principles.
> ...
> However, there are some approaches that might help address your specific need
>
> - SNI proxy, you could use a specialized SNI proxy in front of Nginx
> ...

Which in hindsight is very weird, since it's outright saying it cannot be done, but also suggesting that I use SNI which nginx does support.

Note: It is conceivable that what Claude stated is correct, but I don't understand networking enough to distinguish the difference, in sofar that I think it was ~lying~ hallucinating on me.

## The scent of an answer

At this point Claude was re-suggesting things it's already suggested and was no longer being helpful.
I decided do some research on SNI, as that seemed relevant.

## What is SNI?

Oddly enough, this circled back to an article I read a few months ago, which evidently I didn't commit to memory, <https://www.cloudflare.com/en-gb/learning/ssl/what-is-sni/>.

Relevant bit here

> When multiple websites are hosted on one server and share a single IP address, and each website has its own SSL certificate, the server may not know which SSL certificate to show when a client device tries to securely connect to one of the websites

Incidentally I've been working with API gateways at the time, and it made sense in that regard.

## What do I configure nginx for SNI?

I did what any sensible person would do, search 'sni' in the nginx docs <https://nginx.org/en/docs/http/ngx_http_core_module.html>. 0 / 0 results :thinking:

Hmmm, this might not be as straight forward as I thought it might be.

I've already used Claude for my problem and it was kind of helpful, even though it felt like the long way to an answer, so worth another shot?

It suggested that I use resolvers again, which DOESN'T WORK!

Then is suggested

```nginx
proxy_ssl_server_name on
```

Which IS actually in the [documentation](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) if you search 'SNI', and IF I was searching the correct page :facepalm:.

So that solved my problem, and I bet I will never forget about what SNIs are.
