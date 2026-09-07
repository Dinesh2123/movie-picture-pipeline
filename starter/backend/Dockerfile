FROM python:3.10-alpine
ENV FLASK_RUN_HOST=0.0.0.0

WORKDIR /
RUN adduser -u 1000 -D app
RUN apk update && \
  apk add --no-cache \
  pcre \
  pcre-dev \
  build-base \
  gcc \
  musl-dev \
  linux-headers \
  openssl-dev \
  libffi-dev \
  && pip install --no-cache-dir pipenv

RUN mkdir -p /var/www/app && mkdir /app

COPY Pipfile* /app/
RUN cd /app && (pipenv install --system --deploy || pipenv install --system)

COPY . /app

CMD uwsgi --http :5000 --master --enable-threads -s /var/www/app/app.sock --manage-script-name --mount /=app:app