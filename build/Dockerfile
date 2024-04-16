# Using official Nginx image as the base image
FROM nginx:1.19.10

COPY nginx.conf /etc/nginx/nginx.conf

# Copy our web application files into the Nginx document root
COPY . /var/www/html 

WORKDIR /var/www/html -

# Expose port 80 to the outside world
EXPOSE 80  

CMD ["nginx", "-g", "daemon off;"]
