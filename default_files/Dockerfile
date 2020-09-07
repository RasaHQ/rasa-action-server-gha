ARG RASA_SDK_VERSION
# Extend the official Rasa SDK image
FROM rasa/rasa-sdk:${RASA_SDK_VERSION}

# Use subdirectory as working directory
WORKDIR /app

# Change back to root user to install dependencies
USER root

# To install packages from PyPI
COPY  ./tmp/requirements.txt /tmp/requirements.txt
RUN pip install --no-cache-dir -r /tmp/requirements.txt

# Copy actions folder to working directory
COPY ./tmp/actions /app/actions

# Switch back to non-root to run code
USER 1001
