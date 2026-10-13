# Proxy image inherited from the previous team.
#
# Bonus: switched from a full ubuntu:24.04 + apt-get install to the
# official nginx alpine image. nginx is already installed and built
# for this image, so there's no apt-get at all, and none of the extra
# debugging tools (curl, htop, vim, git) that were never needed at
# runtime in the first place.
FROM nginx:1.27-alpine

COPY nginx.conf /etc/nginx/nginx.conf

# Bonus: run as a non-root user. nginx:alpine already ships a low
# privilege "nginx" user and group. Listening on 8080 (instead of 80)
# means nginx never needs root to bind the port, so the whole process
# -- master included, not just the worker processes -- can run as
# this user.
#
# nginx's default temp/cache directories are only writable by root in
# the base image, so they need to be created and handed over to the
# nginx user before we switch to it.
RUN mkdir -p /var/cache/nginx/client_temp \
             /var/cache/nginx/proxy_temp \
             /var/cache/nginx/fastcgi_temp \
             /var/cache/nginx/uwsgi_temp \
             /var/cache/nginx/scgi_temp && \
    chown -R nginx:nginx /var/cache/nginx

USER nginx

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
