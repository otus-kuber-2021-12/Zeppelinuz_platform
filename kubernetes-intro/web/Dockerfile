FROM nginx:latest
RUN useradd -u 1001 nginxuser
WORKDIR /app
RUN chown -R nginxuser:nginxuser /app
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 8000
STOPSIGNAL SIGQUIT
