FROM nginx:1.25-alpine

# Remove default Nginx static files
RUN rm -rf /usr/share/nginx/html/*

# Copy the frontend HTML
COPY index.html /usr/share/nginx/html/index.html

# Custom Nginx config to serve on port 80
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
