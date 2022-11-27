FROM nginx:stable-alpine
COPY ./nginx.conf /etc/nginx/conf.d/default.conf
COPY ./nginx_server.conf /etc/nginx/nginx.conf
RUN mkdir -p /var/cache/nginx
CMD ["nginx", "-g", "daemon off;"]
