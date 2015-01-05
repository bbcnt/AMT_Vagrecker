FROM oraclelinux:7.0
# Environment variables required for this build (do NOT change)
# -------------------------------------------------------------
ENV JAVA_RPM jdk-8u25-linux-x64.rpm
ENV GLASSFISH_PKG http://dlc-cdn.sun.com/glassfish/4.1/release/glassfish-4.1.zip
ENV PKG_FILE_NAME glassfish-4.1.zip

ENV DOMAIN_NAME domainAMTREST
ENV DB_NAME amtdatabase
ENV DB_TECHNICAL_USER root
ENV DB_TECHNICAL_USER_PASSWORD 123456

ENV JDBC_JNDI_NAME JDBC/domainAMTREST
ENV JDBC_CONNECTION_POOL_NAME domainAMTREST_POOL

# Install and configure Oracle JDK 8u25
# -------------------------------------
#ADD $JAVA_RPM /root/

RUN curl -v -j -k -L -H "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u25-b17/jdk-8u25-linux-x64.rpm > jdk-8u25-linux-x64.rpm
RUN mv $JAVA_RPM /root/$JAVA_RPM
RUN rpm -i /root/$JAVA_RPM && rm /root/$JAVA_RPM
ENV JAVA_HOME /usr/java/default
ENV CONFIG_JVM_ARGS -Djava.security.egd=file:/dev/./urandom
# Setup required packages (unzip), filesystem, and oracle user
# ------------------------------------------------------------
# Enable this if behind proxy
# RUN sed -i -e '/^\[main\]/aproxy=http://proxy.com:80' /etc/yum.conf
RUN yum install -y unzip && yum clean all
RUN useradd -b /opt -m -s /bin/bash glassfish && echo glassfish:glassfish | chpasswd
RUN cd /opt/glassfish && curl -O $GLASSFISH_PKG && unzip $PKG_FILE_NAME && rm $PKG_FILE_NAME
RUN chown -R glassfish:glassfish /opt/glassfish*

ADD mysql-connector-java-5.1.33-bin.jar /mysql-connector-java-5.1.33-bin.jar

# Default glassfish ports
EXPOSE 4848 8080 8181
# Set glassfish user in its home/bin by default
USER glassfish
WORKDIR /opt/glassfish/glassfish4/bin
# User: admin / Pass: glassfish
RUN echo "admin;{SSHA256}80e0NeB6XBWXsIPa7pT54D9JZ5DR5hGQV1kN1OAsgJePNXY6Pl0EIw==;asadmin" > /opt/glassfish/glassfish4/glassfish/domains/domain1/config/admin-keyfile
RUN echo "AS_ADMIN_PASSWORD=glassfish" > pwdfile
# Default to admin/glassfish as user/pass

RUN \
./asadmin start-domain && \
./asadmin --user admin --passwordfile pwdfile enable-secure-admin && \
./asadmin --user admin --passwordfile pwdfile create-jdbc-connection-pool --restype=javax.sql.XADataSource --datasourceclassname=com.mysql.jdbc.jdbc2.optional.MysqlXADataSource --property user=$DB_TECHNICAL_USER:password=$DB_TECHNICAL_USER_PASSWORD:serverName=mysql:portNumber=3306:databaseName=$DB_NAME $JDBC_CONNECTION_POOL_NAME && \
./asadmin --user admin --passwordfile pwdfile create-jdbc-resource --connectionpoolid $JDBC_CONNECTION_POOL_NAME $JDBC_JNDI_NAME && \
./asadmin stop-domain 

RUN cp /mysql-connector-java-5.1.33-bin.jar /opt/glassfish/glassfish4/glassfish/domains/domain1/lib/mysql-connector-java-5.1.33-bin.jar

RUN echo "export PATH=$PATH:/opt/glassfish/glassfish4/bin" >> /opt/glassfish/.bashrc
# Default command to run on container boot
CMD ["/opt/glassfish/glassfish4/bin/asadmin", "start-domain", "--verbose=true"]