FROM dockerfile/ubuntu

RUN apt-get update && apt-get install -y wget git curl zip && rm -rf /var/lib/apt/lists/*
ENV JENKINS_VERSION 1.595

#Setting Jenkins
RUN mkdir /usr/share/jenkins/
RUN useradd -d /home/jenkins -m -s /bin/bash jenkins
ENV JENKINS_HOME /var/jenkins_home
RUN usermod -m -d "$JENKINS_HOME" jenkins && chown -R jenkins "$JENKINS_HOME"
VOLUME /var/jenkins_home

#Installing Maven
RUN apt-get install -y wget
RUN wget --no-verbose -O /usr/share/jenkins/jenkins.war http://mirrors.jenkins-ci.org/war/latest/jenkins.war 
RUN wget --no-verbose -O /tmp/apache-maven-3.2.2.tar.gz http://archive.apache.org/dist/maven/maven-3/3.2.2/binaries/apache-maven-3.2.2-bin.tar.gz
RUN echo "87e5cc81bc4ab9b83986b3e77e6b3095 /tmp/apache-maven-3.2.2.tar.gz" | md5sum -c

# Install Java.
RUN \
	echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | debconf-set-selections && \
	add-apt-repository -y ppa:webupd8team/java && \
	apt-get update && \
	apt-get install -y oracle-java8-installer && \
	rm -rf /var/lib/apt/lists/* && \
	rm -rf /var/cache/oracle-jdk8-installer

# install git
RUN apt-get install -y git

RUN tar xzf /tmp/apache-maven-3.2.2.tar.gz -C /opt/
RUN ln -s /opt/apache-maven-3.2.2 /opt/maven
RUN ln -s /opt/maven/bin/mvn /usr/local/bin
RUN rm -f /tmp/apache-maven-3.2.2.tar.gz
ENV MAVEN_HOME /opt/maven

# remove download archive files
RUN apt-get clean

# Define commonly used JAVA_HOME variable
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle

#Copying the job description and the plugins (mostly, for github)
COPY AMT_REST $JENKINS_HOME/jobs/AMT_REST
ADD plugins $JENKINS_HOME/plugins

#Not used in the project but still useful
ADD jenkins-cli.jar /jenkins-cli.jar

#Adding the config for jenkins (maven and jdk location, credentials, etc)
ADD configs/credentials.xml $JENKINS_HOME/credentials.xml
ADD configs/config.xml $JENKINS_HOME/config.xml
ADD configs/hudson.tasks.Maven.xml $JENKINS_HOME/hudson.tasks.Maven.xml

# for main web interface:
EXPOSE 8080
# will be used by attached slave agents:
EXPOSE 50000
#USER jenkins

ADD jenkins.sh /jenkins.sh
RUN chmod 755 /jenkins.sh
ENTRYPOINT ["/jenkins.sh"]