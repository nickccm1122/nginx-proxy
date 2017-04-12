# nginx-proxy-prerender
This is a repo forked from [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy) but modified to accept 1 extra environment variable and the updated nginx.tmpl. If the PRERENDER_TOKEN is not set, this nginx-proxy-prerender should behave the same as the original nginx-proxy docker container.

For detail of the this docker repo, please refer to original [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy).

## Build the docker image
```zsh
# At the root of the repo
docker build nginx-proxy-prerender .
```

## Push the docker image to docker hub
```zsh
docker push nginx-proxy-prerender
```

## Main differences
The change is based on the [official prerender.io nginx.conf for nginx gist](https://gist.github.com/thoop/8165802). 
And referenced a modified version of the gist for meteor, [Link](https://gist.github.com/blaind/12e53d5d9aa77c9fd841)

/w no PRERENDER_TOKEN passed in

```zsh
location / {
        {{ if eq $proto "uwsgi" }}
        include uwsgi_params;
        uwsgi_pass {{ trim $proto }}://{{ trim $upstream_name }};
        {{ else }}
        proxy_pass {{ trim $proto }}://{{ trim $upstream_name }};
        {{ end }}
        {{ if (exists (printf "/etc/nginx/htpasswd/%s" $host)) }}
        auth_basic  "Restricted {{ $host }}";
        auth_basic_user_file    {{ (printf "/etc/nginx/htpasswd/%s" $host) }};
        {{ end }}
                {{ if (exists (printf "/etc/nginx/vhost.d/%s_location" $host)) }}
                include {{ printf "/etc/nginx/vhost.d/%s_location" $host}};
                {{ else if (exists "/etc/nginx/vhost.d/default_location") }}
                include /etc/nginx/vhost.d/default_location;
                {{ end }}
    }
```

/w PRERENDER_TOKEN passed in

```zsh
location / {
        try_files $uri @prerender;
    }
    location  @prerender {
        set $prerender 0;
        if ($http_user_agent ~* "baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest|slackbot|vkShare|W3C_Validator") {
                set $prerender 1;
        }
        if ($args ~ "_escaped_fragment_") {
                set $prerender 1;
        }
        if ($http_user_agent ~ "Prerender") {
                set $prerender 0;
        }
        if ($uri ~ "\.(js|css|xml|less|png|jpg|jpeg|gif|pdf|doc|txt|ico|rss|zip|mp3|rar|exe|wmv|doc|avi|ppt|mpg|mpeg|tif|wav|mov|psd|ai|xls|mp4|m4a|swf|dat|dmg|iso|flv|m4v|torrent|ttf|woff)") {
                set $prerender 0;
        }

        #resolve using Google's DNS server to force DNS resolution and prevent caching of IPs
        resolver 8.8.8.8;

        if ($prerender = 1) {
                #setting prerender as a variable forces DNS resolution since nginx caches IPs and doesnt play well with load balancing
                set $prerender "service.prerender.io";
                rewrite .* /$scheme://$host$request_uri? break;
                proxy_pass http://$prerender;
        }

        if ($prerender = 0) {
                set $proxy_host $host;
        }

        proxy_set_header X-Prerender-Token {{ (printf $prerender_token) }};
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $proxy_host;

        if ($prerender = 0) {
            {{ if eq $proto "uwsgi" }}
            include uwsgi_params;
            uwsgi_pass {{ trim $proto }}://{{ trim $upstream_name }};
            {{ else }}
            proxy_pass {{ trim $proto }}://{{ trim $upstream_name }};
            {{ end }}
        }

        {{ if (exists (printf "/etc/nginx/vhost.d/%s_location" $host)) }}
        include {{ printf "/etc/nginx/vhost.d/%s_location" $host}};
        {{ else if (exists "/etc/nginx/vhost.d/default_location") }}
        include /etc/nginx/vhost.d/default_location;
        {{ end }}
    }
```
