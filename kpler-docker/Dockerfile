# Start with a base image that includes Maven and Java 21
FROM maven:3.9.6-eclipse-temurin-21

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Install additional necessary packages
RUN apt-get update && apt-get install -y \
    libxrender1 \
    libxtst6 \
    libxi6 \
    libgl1-mesa-glx \
    libgtk2.0-0 \
    libdbus-glib-1-2 \
    libhdf5-dev \
    libjhdf5-java \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*


# Set environment variables for Java to find libjhdf5.so
ENV LD_LIBRARY_PATH=/usr/lib/jni:$LD_LIBRARY_PATH
ENV JAVA_LIBRARY_PATH=/usr/lib/jni

# Create directory for SNAP source code
WORKDIR /opt/snap-src

# 1) clone snap-engine only
RUN git clone https://github.com/senbox-org/snap-engine.git

# 2) copy **this repo’s** microwave-toolbox source in place of the public one
COPY . /opt/snap-src/microwave-toolbox


# Build the SNAP Engine
WORKDIR /opt/snap-src/snap-engine
RUN mvn -q -B clean install -DskipTests

# Build the Microwave Toolbox
WORKDIR /opt/snap-src/microwave-toolbox
RUN mvn -q -B clean install -DskipTests


# Set environment variables for SNAP
ENV SNAP_HOME=/opt/snap-src/snap-engine
ENV SNAP_RUNTIME_CLASSPATH=/opt/snap-src/snap-engine/snap-runtime/target/snap-runtime.jar

# Find all JAR files and add them to the classpath
RUN find /opt/snap-src/snap-engine -name "*.jar" | grep -v "^\." | grep -v "/src/" |  \
    grep -v "/target/classes/" | grep -v "/target/test-classes/" > /opt/snap-src/snap-classpath.txt

# Set up environment for running GPT
RUN echo '#!/bin/bash' > /usr/local/bin/gpt && \
    echo 'CLASSPATH=""' >> /usr/local/bin/gpt && \
    echo 'for JAR in $(cat /opt/snap-src/snap-classpath.txt); do' >> /usr/local/bin/gpt && \
    echo '  CLASSPATH="$CLASSPATH:$JAR"' >> /usr/local/bin/gpt && \
    echo 'done' >> /usr/local/bin/gpt && \
    echo 'for JAR in $(find /opt/snap-src/microwave-toolbox -name "*.jar" | grep -v "^\." | grep -v "/src/" | grep -v "/target/classes/" | grep -v "/target/test-classes/"); do'  \
    >> /usr/local/bin/gpt && \
    echo '  CLASSPATH="$CLASSPATH:$JAR"' >> /usr/local/bin/gpt && \
    echo 'done' >> /usr/local/bin/gpt && \
    echo 'java -Xmx10G -XX:+UseG1GC -XX:+UseStringDeduplication -XX:MaxGCPauseMillis=200 -cp $CLASSPATH org.esa.snap.core.gpf.main.GPT "$@"'  \
    >> /usr/local/bin/gpt && \
    chmod +x /usr/local/bin/gpt

# Add information to the profile
RUN echo 'export SNAP_HOME=/opt/snap-src/snap-engine' >> /root/.bashrc && \
    echo 'export PATH=$SNAP_HOME/bin:$PATH' >> /root/.bashrc

# Define working directory for when container is running
WORKDIR /data

CMD ["/bin/bash"]
