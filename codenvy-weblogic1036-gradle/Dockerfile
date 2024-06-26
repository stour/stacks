FROM centos:latest

# Environment variables required for Java and Weblogic
# ----------------------------------------------------
ENV JAVA_HOME /opt/jdk1.8.0_65

ENV WLS_PKG wls1036_generic.jar
ENV BEA_HOME /home/user/weblogic
ENV MW_HOME /home/user/weblogic
ENV SILENT_XML silent.xml
ENV USER_MEM_ARGS -Xms256m -Xmx512m -XX:MaxPermSize=256m
ENV LD_LIBRARY_PATH $LD_LIBRARY_PATH:/home/user/weblogic/wlserver/server/native/linux/x86_64

ENV MAVEN_VERSION=3.3.9
ENV M2_HOME=/home/user/apache-maven-$MAVEN_VERSION

ENV GRADLE_VERSION=2.13
ENV GRADLE_HOME=/home/user/gradle-$GRADLE_VERSION

ENV PATH=$GRADLE_HOME/bin:$JAVA_HOME/bin:$M2_HOME/bin:$PATH

ENV JDK_VERSION_WS_AGENT=8u65-b17
ENV JDK_ARCHIVE_WS_AGENT=jdk-8u65-linux-x64.tar.gz

ENV TERM xterm
ENV LANG C.UTF-8

# Enable Weblogic debugging
# -------------------------
ENV debugFlag=true

# Ports opening for Codenvy workspace
# -----------------------------------
EXPOSE 4403 8080 8000 22

# Ports opening for Weblogic
# --------------------------
EXPOSE 7001 8453

# Create user and install utilities for Codenvy workspace
# -------------------------------------------------------
RUN yum -y update && \
    yum -y install sudo openssh-server procps wget unzip git curl subversion nmap tar.x86_64 && \
    mkdir /var/run/sshd && \
    sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd && \
    echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers && \
    useradd -u 1000 -G users,wheel -b /home -s /bin/bash -m user && \
    echo -e "codenvy2016\ncodenvy2016" | passwd user && \
    sed -i 's/requiretty/!requiretty/g' /etc/sudoers && \
    wget \
    --no-cookies \
    --no-check-certificate \
    --header "Cookie: oraclelicense=accept-securebackup-cookie" \
    -qO - \
   "http://download.oracle.com/otn-pub/java/jdk/$JDK_VERSION_WS_AGENT/$JDK_ARCHIVE_WS_AGENT" | tar -zx -C /opt/ && \
    echo -e "#! /bin/bash\nset -e\n" > /home/user/entrypoint.sh && \
    echo -e "sudo ssh-keygen -A" >> /home/user/entrypoint.sh && \
    echo -e "sudo /usr/sbin/sshd -D &\nexec \"\$@\"\n" >> /home/user/entrypoint.sh && \
    chmod a+x /home/user/entrypoint.sh

# Go to /home/user as user 'user' to proceed with installation
# ------------------------------------------------------------
WORKDIR /home/user
USER user

LABEL che:server:8080:ref=tomcat8 che:server:8080:protocol=http che:server:8000:ref=tomcat8-debug che:server:8000:protocol=http

# Install Maven
# -------------
RUN mkdir /home/user/apache-maven-$MAVEN_VERSION && \
    wget -qO - "http://mirror.olnevhost.net/pub/apache/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz" \
    | tar -zx --strip-components=1 -C /home/user/apache-maven-$MAVEN_VERSION

# Install Gradle
# --------------
RUN wget -q "https://services.gradle.org/distributions/gradle-$GRADLE_VERSION-bin.zip" && \
    unzip -oq gradle-$GRADLE_VERSION-bin.zip && \
    rm gradle-$GRADLE_VERSION-bin.zip

# Copy Wevlogic package and configuration file
# --------------------------------------------
COPY $SILENT_XML /home/user/
COPY $WLS_PKG /home/user/

# Install Weblogic
# ----------------
RUN java -jar $WLS_PKG -mode=silent -silent_xml=$SILENT_XML && \
    rm $WLS_PKG $SILENT_XML
RUN ln -s /home/user/weblogic/wlserver_10.3 /home/user/weblogic/wlserver

ENV PATH $PATH:/home/user/weblogic/wlserver/common/bin

# Set Weblogic environment
# ------------------------
RUN cd $MW_HOME/wlserver/server/bin && ./setWLSEnv.sh

# Create weblogic domain
# ----------------------
COPY create-wls-domain.py /home/user
RUN sudo chmod +x /home/user/create-wls-domain.py && \
   /home/user/weblogic/wlserver/common/bin/wlst.sh -skipWLSModuleScanning /home/user/create-wls-domain.py && \
   mkdir -p /home/user/weblogic/user_projects/domains/base_domain/servers/AdminServer/security && \
   echo "username=weblogic" > /home/user/weblogic/user_projects/domains/base_domain/servers/AdminServer/security/boot.properties && \
   echo "password=codenvy2016" >> /home/user/weblogic/user_projects/domains/base_domain/servers/AdminServer/security/boot.properties && \
   echo "/home/user/weblogic/user_projects/domains/base_domain/bin/setDomainEnv.sh" >> ~/.bashrc && \
   echo "export PATH=$PATH:/home/user/weblogic/user_projects/domains/base_domain/bin" >> ~/.bashrc && \
   sudo sed -i 's/^securerandom.source=.*$/securerandom.source=file:\/dev\/.\/urandom/' $JAVA_HOME/jre/lib/security/java.security

# Copy utility needed for deployment of Gradle app to Weblogic
# See https://github.com/Liverpool-Victoria/gradle-weblogic-plugin#usage
COPY wlfullclient-1036.jar $MW_HOME
RUN sudo chmod +x $MW_HOME/wlfullclient-1036.jar

# Define default command to start bash
# -------------------------------------
ENTRYPOINT ["/home/user/entrypoint.sh"]

CMD tail -f /dev/null
