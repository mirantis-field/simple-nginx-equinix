FROM nginx:{{ .Parameters.nginxVersion }}

COPY html /usr/share/nginx/html

HEALTHCHECK --interval=5s --timeout=5s CMD wget -q -O - -U "healthcheck" http://localhost:80/ || exit 1