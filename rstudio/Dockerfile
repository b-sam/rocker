## Start with the official rocker image providing 'base R'
FROM r-base:latest
## This handle reaches Carl and Dirk
MAINTAINER "Carl Boettiger and Dirk Eddelbuettel" rocker-maintainers@eddelbuettel.com

## Add RStudio binaries to PATH
ENV PATH /usr/lib/rstudio-server/bin/:$PATH


RUN sh -c "echo 'deb http://ftp.de.debian.org/debian jessie main' >> /etc/apt/sources.list"

## Download and install RStudio server & dependencies
## Attempts to get detect latest version, otherwise falls back to version given in $VER
## Symlink pandoc, pandoc-citeproc so they are available system-wide
RUN rm -rf /var/lib/apt/lists/ \
  && apt-get update \
  && apt-get install -t stable libssl1.0.0 \
  && apt-get install -y -t unstable --no-install-recommends \
    ca-certificates \
    file \
    git \
    libapparmor1 \
    libedit2 \
    libcurl4-openssl-dev \
    libssl-dev \
    psmisc \
    python-setuptools \
    sudo \
  && VER=$(wget --no-check-certificate -qO- https://s3.amazonaws.com/rstudio-server/current.ver) \
  && wget -q http://download2.rstudio.org/rstudio-server-${VER}-amd64.deb \
  && dpkg -i rstudio-server-${VER}-amd64.deb \
  && rm rstudio-server-*-amd64.deb \
  && ln -s /usr/lib/rstudio-server/bin/pandoc/pandoc /usr/local/bin \
  && ln -s /usr/lib/rstudio-server/bin/pandoc/pandoc-citeproc /usr/local/bin \
  && wget https://github.com/jgm/pandoc-templates/archive/1.15.0.6.tar.gz \
  && mkdir -p /opt/pandoc/templates && tar zxf 1.15.0.6.tar.gz \
  && cp -r pandoc-templates*/* /opt/pandoc/templates && rm -rf pandoc-templates* \
  && mkdir /root/.pandoc && ln -s /opt/pandoc/templates /root/.pandoc/templates \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/

## Ensure that if both httr and httpuv are installed downstream, oauth 2.0 flows still work correctly.
RUN echo '\n\
\n# Configure httr to perform out-of-band authentication if HTTR_LOCALHOST \
\n# is not set since a redirect to localhost may not work depending upon \
\n# where this Docker container is running. \
\nif(is.na(Sys.getenv("HTTR_LOCALHOST", unset=NA))) { \
\n  options(httr_oob_default = TRUE) \
\n}' >> /etc/R/Rprofile.site

RUN echo "PATH=$PATH" >> /etc/R/Renviron.site

## A default user system configuration. For historical reasons,
## we want user to be 'rstudio', but it is 'docker' in r-base
RUN usermod -l rstudio docker \
  && usermod -m -d /home/rstudio rstudio \
  && groupmod -n rstudio docker \
  && echo '"\e[5~": history-search-backward' >> /etc/inputrc \
  && echo '"\e[6~": history-search-backward' >> /etc/inputrc \
  && echo "rstudio:rstudio" | chpasswd

## Use s6
RUN wget -P /tmp/ https://github.com/just-containers/s6-overlay/releases/download/v1.11.0.1/s6-overlay-amd64.tar.gz \
  && tar xzf /tmp/s6-overlay-amd64.tar.gz -C /

# Downloads URLs
ENV MS_ODBC_URL https://download.microsoft.com/download/B/C/D/BCDD264C-7517-4B7D-8159-C99FC5535680/RedHat6/msodbcsql-11.0.2270.0.tar.gz
ENV FIX_SCRIPT Microsoft--SQL-Server--ODBC-Driver-1.0-for-Linux-Fixed-Install-Scripts
ENV FIX_SCRIPT_URL https://github.com/Andrewpk/${FIX_SCRIPT}/archive/master.zip

# Set the locale
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

RUN apt-get update && apt-get -y install aptitude wget unzip make gcc lsb-release libkrb5-3 libgssapi-krb5-2

# Download ODBC install files & scripts
RUN cd /tmp && wget -O msodbcsql.tar.gz ${MS_ODBC_URL} && wget -O odbc-fixed.zip ${FIX_SCRIPT_URL}

RUN cd /tmp && wget -O unixODBC-2.3.4.tar.gz  http://www.unixodbc.org/unixODBC-2.3.4.tar.gz

# Unzip downloaded files
RUN cd /tmp && tar -xzf ./msodbcsql.tar.gz && unzip -o ./odbc-fixed.zip && cp ./${FIX_SCRIPT}-master/* ./msodbcsql-11.0.2270.0

# Run install scripts
RUN cd /tmp/msodbcsql-11.0.2270.0 && yes YES | ./build_dm.sh --download-url=file:///tmp/unixODBC-2.3.4.tar.gz  --accept-warning --libdir=/usr/lib/x86_64-linux-gnu && ./install.sh install --accept-license --force

# Clean installation files
#RUN apt-get remove -y aptitude wget unzip make gcc && apt-get -y autoremove && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*



COPY userconf.sh /etc/cont-init.d/conf
COPY run.sh /etc/services.d/rstudio/run
COPY odbc.ini /etc/odbc.ini
RUN chmod 777 /etc/odbc.ini

# Add symbolic links for missing libraries
RUN ln -s /usr/lib/x86_64-linux-gnu/libssl.so.1.0.0 /usr/lib/x86_64-linux-gnu/libssl.so.10
RUN ln -s /usr/lib/x86_64-linux-gnu/libcrypto.so.1.0.0 /usr/lib/x86_64-linux-gnu/libcrypto.so.10
RUN ln -s /usr/lib/x86_64-linux-gnu/libodbcinst.so.2 /usr/lib/x86_64-linux-gnu/libodbcinst.so.1

RUN R -e "install.packages(c('RODBC','ggplot2','shiny', 'rmarkdown', 'tm', 'wordcloud', 'memoise'), repos='http://cran.rstudio.com/')"

COPY add-students.sh /usr/local/bin/add-students
EXPOSE 8787

## Expose a default volume for Kitematic
VOLUME /home/rstudio

CMD ["/init"]
