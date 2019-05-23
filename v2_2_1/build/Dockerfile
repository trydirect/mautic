FROM ubuntu:14.04

RUN apt-get update && apt-get install -y apache2 software-properties-common

RUN groupadd -r -g 2000 mautic; useradd -r -u 2000 -g 2000 -m -c "app account" -d /home/mautic -s /bin/bash mautic

RUN rm -rf /var/www/html && mkdir -p /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www/html \
    && chown -R mautic. /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www/html

RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf
ENV APACHE_LOCK_DIR=/var/lock/apache2
ENV APACHE_PID_FILE=/var/run/apache2/apache2.pid
ENV APACHE_RUN_DIR=/var/run/apache2/
ENV APACHE_RUN_USER=mautic
ENV APACHE_RUN_GROUP=mautic
ENV APACHE_LOG_DIR=/var/log/apache2
ENV APACHE_SERVER_NAME=demo.loc

RUN a2enmod rewrite ssl

RUN add-apt-repository -y ppa:ondrej/php && apt-get update && apt-get install -y --force-yes php5 libapache2-mod-php5.6

# Enable required php5 modules
RUN apt-get update && apt-get install -y --force-yes php5.6-mcrypt php5.6-mysql php5.6-xsl php5.6-gd php5.6-intl php5.6-curl \
    && php5enmod mcrypt mysql xsl gd intl curl imap imap-ssl kerberos mbstring mysqli pdo pdo_mysql

RUN apt-get update && apt-get install -y --force-yes libc-client-dev libicu-dev libkrb5-dev libmcrypt-dev libssl-dev unzip zip git php5.6-cli cron vim curl wget
RUN rm -rf /var/lib/apt/lists/*
RUN git clone https://github.com/docker-library/php
RUN chmod 777 -R php/5.6
WORKDIR php/5.6
RUN /php/5.6/docker-php-ext-configure imap --with-imap --with-imap-ssl --with-kerberos
RUN /php/5.6/docker-php-ext-install imap intl mbstring mcrypt mysqli pdo pdo_mysql

# Add the Apache virtual host file
ADD config/apache_default_vhost /etc/apache2/sites-available/demo.loc.conf
ADD config/apache_default_vhost_ssl /etc/apache2/sites-available/ssl.conf
COPY config/ports.conf /etc/apache2/ports.conf
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
ENV MAUTIC_VERSION 2.2.1
ENV MAUTIC_SHA1 f0f89343f9ce67b6b4cafb44fd7b15f325ed726f

# Download package and extract to web volume
WORKDIR /usr/src
RUN git clone -b optimum-patches https://github.com/optimum-web/mautic.git
WORKDIR /usr/src/mautic
RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer
RUN composer install
RUN mkdir /usr/src/mautic/app/cache/ip_data
RUN chown -R mautic. /usr/src/mautic

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
RUN touch /var/log/cron.log
RUN touch /var/log/cron.log && \
    chown mautic. /var/log/cron.log && \
    chown -R mautic. /var/run/ && \
    chmod u+s /usr/sbin/cron

RUN /bin/bash -c crontab -l > /etc/cron.d/mautic-cronjobs
RUN echo '* 2 10 * * mautic /bin/sh /update_geolite.sh >/dev/null 2>&1' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * mautic php /var/www/html/app/console mautic:segments:update --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * mautic php /var/www/html/app/console mautic:campaigns:rebuild --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * mautic php /var/www/html/app/console mautic:campaigns:trigger --force --batch-limit=20 --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * mautic php /var/www/html/app/console mautic:emails:send --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '*/5 * * * * mautic php /var/www/html/app/console mautic:email:fetch --env=prod' >> /etc/cron.d/mautic-cronjobs
RUN echo '* * * * * mautic echo "Mautic cronjob check every minute" >> /var/log/cron.log 2>&1' >> /etc/cron.d/mautic-cronjobs
RUN crontab /etc/cron.d/mautic-cronjobs
WORKDIR /var/www/html

# Copy init scripts and custom .htaccess
COPY docker-entrypoint.sh /entrypoint.sh
COPY makeconfig.php /makeconfig.php
COPY makedb.php /makedb.php

RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER mautic
ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 80
EXPOSE 443
CMD ["apache2-foreground"]
