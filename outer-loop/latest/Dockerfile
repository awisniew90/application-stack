# Step 1: Build the user's application
FROM awisniew90/application-stack:0.3.1 as compile

# Make a well known place for shared library jars separate from the rest of the <server> contents (to help with caching)
RUN mkdir /work/configlibdir \
   && mkdir /work/config \
   &&  mkdir /work/shared

# Copy the rest of the application source
COPY --chown=1001:0 ./src /projects/src
COPY --chown=1001:0 ./pom.xml /projects/pom.xml

# Build (and run unit tests) 
#  also liberty:create copies config from src->target
#  also remove quick-start-security.xml since it's convenient for local dev mode but should not be in the production image.
RUN cd /projects && \
    echo "QUICK START SECURITY IS NOT SECURE FOR PRODUCTION ENVIRONMENTS. IT IS BEING REMOVED" && \
    rm -f src/main/liberty/config/configDropins/defaults/quick-start-security.xml && \
    mvn -e liberty:create package

# Process any resources or shared libraries - if they are present in the dependencies block for this project (there may be none potentially)
# Test to see if each is present and move to a well known location for later processing in the next stage. This allows the COPY commands below
# to work even if these directories do not exist. 
# 
RUN cd /projects/target/liberty/wlp/usr/servers && \
    if [ -d ./*/lib ]; then mv ./*/lib /work/configlibdir; fi && \
    if [ ! -d /work/configlibdir/lib ]; then mkdir /work/configlibdir/lib; fi && \
    mv -f */* /work/config/ && \
    if [ -d ../shared ]; then mv ../shared/* /work/shared/; fi

# Step 2: Package Open Liberty image
FROM openliberty/open-liberty:20.0.0.10-kernel-java11-openj9-ubi

#2a) copy any resources 
COPY --from=compile --chown=1001:0 /work/shared /opt/ol/wlp/usr/shared/

# 2b) next copy shared library
#      but can't assume config/lib exists - copy from previous stage to a tmp holding place and test
COPY --from=compile --chown=1001:0 /work/configlibdir/ /config

# 2c) Server config, bootstrap.properties, and everything else
COPY --from=compile --chown=1001:0 /work/config/ /config/

# 2d) Changes to the application binary
COPY --from=compile --chown=1001:0 /projects/target/*.[ew]ar /config/apps
RUN configure.sh && \
    chmod 664 /opt/ol/wlp/usr/servers/*/configDropins/defaults/keystore.xml
