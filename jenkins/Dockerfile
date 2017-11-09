FROM jenkins/jenkins

USER root
RUN apt-get update && apt-get install -y vim \
 && apt-get install -y ssh openssh-client

WORKDIR /app

COPY /scripts/credentials/ssh_key/* /var/jenkins_home/.ssh/
COPY /scripts/credentials/ssh_key/id_rsa_github /var/jenkins_home/.ssh/id_rsa
COPY /scripts/credentials/ssh_key/id_rsa_github.pub /var/jenkins_home/.ssh/id_rsa.pub
RUN chmod -R 700 /var/jenkins_home/.ssh \
    && chown -R jenkins:jenkins /var/jenkins_home/.ssh

RUN apt-get install -y postgresql
RUN mkdir /mnt/postgres \
    && chown jenkins:jenkins /mnt/postgres

RUN apt-get install -y mysql-client
RUN mkdir /mnt/mysql \
    && chown jenkins:jenkins /mnt/mysql

ENV TZ 'America/Los_Angeles'

COPY scripts/db_bootstrap/2017-11-05-190006_backup.sql /app/bootstrap.sql
COPY scripts/db_bootstrap/bootstrap.sh /app/bootstrap.sh

USER jenkins



