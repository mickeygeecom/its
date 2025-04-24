# Nginx Full Reverse Proxy Configuration

Denne konfiguration anvendes på reverse proxy-serverne til at håndtere TCP og UDP trafik mellem loadbalanceren og FiveM origin-serveren. Den inkluderer performanceoptimeringer til lav latency og høj gennemstrømning.

```nginx
user www-data;
worker_processes  auto;
error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections 65535;
    multi_accept on;
    use epoll;

    # Enhanced UDP handling
    worker_aio_requests 512;      # Increased from 256
    epoll_events 2048;            # Increased from 1024
    accept_mutex on;              # Better connection distribution
    accept_mutex_delay 100ms;     # Slight delay for mutex
}

stream {
    upstream fivem_backend {
        server ip.fivem.socialrp.dk:30121;
    }

    # TCP server block
    server {
        listen 30120;
        proxy_pass fivem_backend;
        
        # TCP optimizations
        proxy_buffer_size 128k;
        proxy_connect_timeout 10s;
        proxy_timeout 30s;
    }

    # UDP server block
    server {
        listen 30120 udp reuseport;
        proxy_pass fivem_backend;
        
        # Optimized buffer settings
        proxy_buffer_size 512k;  # Increased from 8k to handle bursts
        proxy_timeout 3s;        # Reduced from 5s for faster recovery
    }
}

worker_rlimit_core 500M;
worker_shutdown_timeout 10s;
worker_rlimit_nofile 200000;      # Increased file descriptor limit
```
