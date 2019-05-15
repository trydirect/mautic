FROM ubuntu:16.04

RUN apt-get update && apt-get install -y apache2 software-properties-common

RUN rm -rf /var/www/html && mkdir -p /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www/html \
    && chown -R www-data:www-data /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www/html

RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf
ENV APACHE_LOCK_DIR=/var/lock/apache2
ENV APACHE_PID_FILE=/var/run/apache2/apache2.pid
ENV APACHE_RUN_DIR=/var/run/apache2/
ENV APACHE_RUN_USER=www-data
ENV APACHE_RUN_GROUP=www-data
ENV APACHE_LOG_DIR=/var/log/apache2
ENV APACHE_SERVER_NAME=demo.loc

RUN a2enmod rewrite ssl

RUN apt-get install -y --force-yes php libapache2-mod-php php-mcrypt php-mysql php7.0-mysql php-xsl php-gd php-intl php-curl libc-client-dev \
php-mbstring php7.0-mbstring libapache2-mod-php7.0 php-bcmath php-zip libicu-dev libkrb5-dev libmcrypt-dev libssl-dev unzip zip git php-cli \
cron vim curl wget git && phpenmod mcrypt mysql xsl gd intl curl imap imap-ssl kerberos mbstring mysqli pdo pdo_mysql

RUN rm -rf /var/lib/apt/lists/*

# Add the Apache virtual host file
ADD config/apache_default_vhost /etc/apache2/sites-available/demo.loc.conf
ADD config/apache_default_vhost_ssl /etc/apache2/sites-available/ssl.conf
RUN rm -f /etc/apache2/sites-enabled/000-default.conf

RUN a2ensite demo.loc.conf
RUN a2ensite ssl.conf

# Copy SSL cert over
COPY ssl /etc/apache2/ssl/

VOLUME /var/www/html

COPY apache2-foreground /usr/local/bin/
RUN chmod +x /usr/local/bin/apache2-foreground
WORKDIR /var/www/html

# Define Mautic version and expected SHA1 signature
ENV MAUTIC_VERSION=2.15.0
ENV MAUTIC_SHA1=eec5676cc242b32142b6efcf0c44edc8b16fdd63

# Download package and extract to web volume
WORKDIR /usr/src
RUN git clone https://github.com/mautic/mautic.git

WORKDIR /usr/src/mautic
RUN git fetch --all --tags
RUN git checkout tags/2.15.0 -b 2.15.0

RUN sed -i -e "s/;cgi.fix_pathinfo\s*=\s*1/cgi.fix_pathinfo = 0/g; s/expose_php\s*=\s*On/expose_php = Off/g" \
/etc/php/7.0/apache2/php.ini
RUN sed -i -e "s/expose_php\s*=\s*On/expose_php = Off/g" /etc/php7.0/fpm/php.ini
RUN sed -i -e "s/upload_max_filesize\s*=\s*2M/upload_max_filesize = 100M/g; s/post_max_size\s*=\s*8M/post_max_size = 100M/g" \
/etc/php/7.0/apache2/php.ini

RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer
RUN composer install
RUN mkdir /usr/src/mautic/app/cache/ip_data
RUN chown -R www-data:www-data /usr/src/mautic

# Make script for update Geolite2
RUN touch /update_geolite.sh
RUN echo "cd /var/www/html/app/cache/ip_data" >> /update_geolite.sh
RUN echo "curl -o GeoLite2-City.mmdb.gz -SL http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.mmdb.gz" >> /update_geolite.sh
RUN echo "gunzip -f /var/www/html/app/cache/ip_data/*.gz" >> /update_geolite.sh
RUN echo "chmod 755 /var/www/html/app/cache/ip_data/GeoLite2-City.mmdb" >> /update_geolite.sh
RUN chmod a+x /update_geolite.sh

# Install mautic cronjobs
RUN apt-get update
RUN touch /etc/cron.d/mautic-cronjobs
RUN /bin/bash -c crontab -l > /etc/cron.d/mautic-cronjobs
RUN touch /var/log/cron.log
RUN chown www-data:www-data /var/log/cron.log

RUN echo '* 2 10 * *  www-data /bin/sh /update_geolite.sh >/dev/null 2>&1' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * www-data php /var/www/html/app/console mautic:segments:update --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * www-data php /var/www/html/app/console mautic:campaigns:rebuild --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * www-data php /var/www/html/app/console mautic:emails:send --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '* * * * * www-data echo "Mautic cronjob check" >> /var/log/cron.log 2>&1' >> /etc/cron.d/mautic-cronjobs

RUN crontab /etc/cron.d/mautic-cronjobs
WORKDIR /var/www/html

# Copy init scripts and custom .htaccess
COPY docker-entrypoint.sh /entrypoint.sh
COPY makeconfig.php /makeconfig.php
COPY makedb.php /makedb.php

RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN service cron restart
ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 80
EXPOSE 443
CMD ["apache2-foreground"]
