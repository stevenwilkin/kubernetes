FROM nginx

COPY . .

RUN cp -f nginx.conf /etc/nginx/ \
	&& cp -rf conf.d /etc/nginx
