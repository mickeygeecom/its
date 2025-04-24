# Nginx Loadbalancer Configuration

Dette er den komplette `nginx.conf`-fil brugt som loadbalancer foran vores reverse proxies og FiveM-serveren. Den håndterer både HTTP/S og TCP/UDP via stream-blokken.

```nginx
user www-data;
worker_processes auto;
worker_rlimit_nofile 200000;
error_log /var/log/nginx/error.log;
pid /var/run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 100000;
    multi_accept on;
    use epoll;
}

http {
    resolver 1.1.1.1 valid=300s;

    limit_req_zone $binary_remote_addr zone=one:10m rate=5r/s;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    server_tokens off;

    server_names_hash_bucket_size 64;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    log_format json escape=json '{ '
                                '"time": "$time_iso8601", '
                                '"addr": "$remote_addr", '
                                '"forwarded_for": "$http_x_forwarded_for", '
                                '"url": "$scheme://$http_host$request_uri", '
                                '"method": "$request_method", '
                                '"protocol": "$server_protocol", '
                                '"status": $status, '
                                '"user_agent": "$http_user_agent", '
                                '"referer": "$http_referer", '
                                '"length": $request_length, '
                                '"request_time": $request_time, '
                            '}';

    access_log /var/log/nginx/access.log json;

    gzip on;

    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 131.0.72.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    set_real_ip_from 2400:cb00::/32;
    set_real_ip_from 2606:4700::/32;
    set_real_ip_from 2803:f800::/32;
    set_real_ip_from 2405:b500::/32;
    set_real_ip_from 2405:8100::/32;
    set_real_ip_from 2a06:98c0::/29;
    set_real_ip_from 2c0f:f248::/32;

    real_ip_header CF-Connecting-IP;

    proxy_cache_path /srv/cache levels=1:2 keys_zone=assets:48m max_size=20g inactive=2h;
    log_format asset '$remote_addr - [$time_local] "$http_host" "$request" $status $body_bytes_sent $upstream_cache_status';

    server {
        listen 443 default_server;
        listen [::]:443 default_server;
        ssl_reject_handshake on;
        server_name _;
        return 444;
    }

    include /etc/nginx/backend.conf;

    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        http2 on;

        server_name fivem.socialrp.dk;

        ssl_certificate /etc/nginx/ssl/certificate.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;

        error_page 497 301 =444 @fallback;
        error_page 502 301 =444 @fallback;
        recursive_error_pages on;

        if ($server_protocol != "HTTP/2.0") {
            return 444;
        }

        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-CF-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_pass_request_headers on;
            proxy_http_version 1.1;
            proxy_pass http://backend_fivem$request_uri;
        }

        location /client {
            limit_req zone=one burst=10 nodelay;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-CF-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_pass_request_headers on;
            proxy_http_version 1.1;
            proxy_pass http://backend_fivem/client;
            limit_except POST {
                deny all;
            }
        }

        location /files/ {
            access_log /var/log/nginx/cache.access.log asset;
            proxy_pass http://backend_fivem$request_uri;
            add_header X-Cache-Status $upstream_cache_status;
            proxy_cache_lock on;
            proxy_cache assets;
            proxy_cache_valid 1y;
            proxy_cache_key $request_uri$is_args$args;
            proxy_cache_revalidate on;
            proxy_cache_min_uses 1;
        }

        location @fallback {
            return 444;
        }
    }
}

stream {
    include /etc/nginx/backend.conf;

    server {
        listen 30120;
        proxy_pass backend_fivem;
    }

    server {
        listen 30120 udp reuseport;
        proxy_pass backend_fivem;
        proxy_buffer_size 512k;
        proxy_timeout 3s;
    }
}
```

