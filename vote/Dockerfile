# define base stage that uses official python runtime base image
FROM python:3.11-slim AS base

# add curl for healtcheck
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# set application directory
WORKDIR /usr/local/app

# install requirements.txt
COPY requirements.txt ./requirements.txt

RUN pip install --no-cache-dir -r requirements.txt

# define a stage specifically for development, where it'll watch for
# filesystem changes
FROM base AS dev
RUN pip install watchdog
ENV FLASK_ENV=development
CMD ["python", "app.py"]

# Define final stage that will bundle application for production
FROM base AS final

# copy our code from current folder to working directory inside the container
COPY . .

# make port 80 available for links and/or publish
EXPOSE 80

# defind our command to run  when launching container
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80", "--log-file", "-","--access-log", "-", "--workers", "4", "--keep-alive", "0"]
