FROM alpine:latest

RUN apk add --update nginx
RUN mkdir -p /run/nginx && touch /run/nginx/nginx.pid
RUN echo '<h1>Hello from MyNginx Container!</h1>' > /var/lib/nginx/html/index.html
RUN sed -i -e 's/return 404;/index index.html;/' -e 's/internal;/return 404;/' /etc/nginx/conf.d/default.conf
CMD ["nginx", "-g", "daemon off;"]
