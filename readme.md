# Agent Log Viewer

The service is a secure way to expose files within a specified directory via https. It is useful for exposing logs generated by an infrastructure provider's program to its developers.

## Run On Localhost

A minimal dockerized python code for exposing files within a specified directory via http. To setup, run

```bash
cp .env.example .env
```

and fill in the .env file with required fields. Then build locally via

```bash
docker build . -t log-viewer
```

and run

```bash
docker compose up -d
```

## Expose Via NGINX

Append the below configuration to your existing nginx configuration:

```text
location /<ROOT_API_PATH> {
    proxy_pass http://127.0.0.1:<API_PORT>;
    rewrite ^/<ROOT_API_PATH>/(.*)$ /$1 break;
}
```

### IP whitelisting

To allow access only from selected IP, prepend

```text
allow <IP>;
deny all;
```

to the previous nginx location rule.

### Password protect

To password-protect endpoint access for user <USERNAME> via basic auth, run

```bash
sudo sh -c "echo -n '<USERNAME>:' >> /etc/nginx/.htpasswd"
sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd" # choose password
```

then prepend

```text
auth_basic "Restricted Content";
auth_basic_user_file /etc/nginx/.htpasswd;
```

to the previous nginx location rule (after IP whitelisting).

### Example

Example nginx configuration is

```text
{
    listen 443 ssl http2;
    server_name mydomain.com;

    ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem;

    location /flare/view-logs {
        allow 155.130.131.55;
        deny all;
        auth_basic           "Restricted Content";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://127.0.0.1:57005;
        rewrite ^/flare/view-logs/(.*)$ /$1 break;
    }
}
```

## Access

access by navigating to hostname.ext/<ROOT_API_PATH>/ (note the ending frontslash).

## Security

The service guarantees that only files within the specified `.env` directory `LOG_DIR_PATH` are exposed, via multiple layers of protection:

1. **Application**: Application is a 30-line python script, without external dependencies, that can be quickly reviewed by the user. It explicitly ensures that no files outside the given directory are exposed, and serves locally on 127.0.0.1.
1. **Container**: The service is dockerized with only the specified directory mounted as a volume. Effectively, even if there is a bug at the application layer, it would have great difficulty penetrating outside the docker host's filesystem.

Additionally, the service is further secured by the deployer's nginx configuration:

1. **IP Whitelisting**: Only requests from the specified IP are allowed,
1. **Basic Auth**: Only requests with the configured correct username and password are allowed.

Even though the served files should not be sensitive in nature, the nginx configuration needs to provide TLS encryption, because e.g. the basic auth password is sent in plaintext.