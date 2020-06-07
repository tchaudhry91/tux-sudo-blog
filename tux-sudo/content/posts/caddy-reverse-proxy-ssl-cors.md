---
title: "Easy SSL Termination - CORS Edition"
date: 2020-05-05T09:43:52Z
categories: ["Ops", "Development", "CORS", "Caddy", "SSL"]
---

I've always been a big fan of [Caddy](https://caddyserver.com/), and it just got a 2.0 release! In the meantime, I have developed another throwaway service called [Archy](https://github.com/tchaudhry91/archy), and naturally, it deserves the best Ops can offer.

So, here's the scenario. I need a simple HTTPS service exposed to the world. I have a VPS with a public IP. Of course, I'll celebrate Caddy's 2nd birth and write a simple Caddyfile.

```bash
archy.tux-sudo.com

reverse_proxy /* 127.0.0.1:15999
```

Done. Cool. See you later.

I can easily use `https://archy.tux-sudo.com` for my API using my command line. Everyone (only me) is happy. A coconut fell on my head and I decided to venture into the marvellous world of Javascript. I'll build myself a UI. Why not? Really though, why? Nevertheless, the age old developer horror story of having to resolve CORS struck. Now, it's not my first time, I know what CORS is, how it works etc etc. But my service was released, I didn't want to go back and make changes in there (see, already thrown away). Now, I need to solve the following problems:

- My service needs to add Access-Control Headers
- My service needs to respond 200 to OPTIONS on a particular path.

Ugh. No, my service doesn't do this and I'm happy to keep it that way. BUT Caddy can help. Have a look at my updated Caddyfile:

```bash
archy.tux-sudo.com

@entries_options {
        method OPTIONS
        path /entries
}

header {
        Access-Control-Allow-Headers "*"
        Access-Control-Allow-Origin "*"
        Access-Control-Allow-Methods "GET, POST, OPTIONS"
}

respond @entries_options 200

reverse_proxy /* 127.0.0.1:15999 {
}
```

Use the appropriate headers of course and not "\*", but you get the point. This achieves both. I've added headers to be put on my final response and it's also handling the OPTIONS call on `/entries` using my `@entries_options` matcher properly. My service has no idea. Now, I'm sure Nginx or similar might be able to do this as well. But the level of power and clarity in this config is unparalleled imho. Wishing Caddy many many happy years ahead.
