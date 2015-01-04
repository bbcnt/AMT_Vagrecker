FROM dockerfile/ubuntu

RUN groupadd mysql
RUN useradd mysql -g mysql -u 4242

RUN apt-get update \
 && apt-get install -y mysql-server \
 && rm -rf /var/lib/mysql/mysql \
 && rm -rf /var/lib/apt/lists/* # 20140918

ADD start /start
ADD example_config.sql /tmp/example_config.sql
RUN chmod 755 /start

EXPOSE 3306

VOLUME ["/var/lib/mysql"]
VOLUME ["/run/mysqld"]

CMD ["/start"]
