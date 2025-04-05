# immich-ml-proxy-nginx

Route Immich ML requests to different ML servers depending on the type of the query in the request body.

Motivation: https://github.com/immich-app/immich/discussions/17045

## Options

- `router1`: pure Nginx.
- `router2`: OpenResty with the `lua-resty-upload` module for streaming processing of the request body.
- `router3`: OpenResty with the `lua-nginx-module` module for buffered processing with `get_body_data`.

See [compose.yaml](./compose.yaml) for definitions.

## Details

At the moment, Immich server specifies the query type in the request body.

Example smart search POST request from Immich server to Immich ML (captured using `docker run --rm -ti --net container:"immich-ml" nicolaka/netshoodshoot ngrep -dany -Wbyline '' 'port 3003'`):

```
POST /predict HTTP/1.1
host: ml:3003
connection: keep-alive
content-type: multipart/form-data; boundary=----formdata-undici-043847285935
accept: */*
accept-language: *
sec-fetch-mode: cors
accept-encoding: gzip, deflate
content-length: 265

------formdata-undici-043847285935
Content-Disposition: form-data; name="entries"

{"clip":{"textual":{"modelName":"ViT-B-32__openai"}}}

------formdata-undici-043847285935
Content-Disposition: form-data; name="text"

cat

------formdata-undici-043847285935--
```

Therefore unless it is changed in the code to be path/URI-based, we have to perform the POST request body inspection to determine the type of the query and choose a backend/target accordingly.

So far I considered Tyk, Kong, Caddy, Nginx. There is no simple config/solution with request routing based on the body inspection. (Otherwise, let me know). So I implemented a few routers based on Nginx.

## Caveats

Performance-wise, memory is the main concern, as we read the entire request body buffered into RAM. Alternatively, it can be spooled to disk, though it will be slower.

Also max body size is set to an arbitrary value of 32MB.

For dynamic backend selection during proxying we need to read the request body early.

## Testing

Minimal curl to create a multipart/form-data request for testing:

```sh
# router1
docker compose exec client curl -F 'entries={"clip":{"textual":{"modelName":"ViT-B-32__openai"}}}' -F 'text=cat' http://router1/predict

# router2
docker compose exec client curl -F 'entries={"clip":{"textual":{"modelName":"ViT-B-32__openai"}}}' -F 'text=cat' http://router2/predict
```

## Links

- https://github.com/openresty/lua-resty-upload

- https://github.com/openresty/lua-nginx-module?tab=readme-ov-file#ngxreqget_body_data

- https://stackoverflow.com/questions/21866477/nginx-use-environment-variables
- https://stackoverflow.com/questions/70849609/how-can-i-get-an-environment-variable-or-a-default-string

- https://github.com/adrianbrad/nginx-request-body-reverse-proxy
- https://dev.to/danielkun/nginx-everything-about-proxypass-2ona#better-logging-format-for-proxypass

- https://www.f5.com/company/blog/nginx/deploying-nginx-plus-as-an-api-gateway-part-2-protecting-backend-services
