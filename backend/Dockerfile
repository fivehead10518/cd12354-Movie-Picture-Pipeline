FROM public.ecr.aws/docker/library/python:3.10-alpine
ENV FLASK_RUN_HOST=0.0.0.0

WORKDIR /
RUN adduser -u 1000 -D app
RUN apk update && \
  apk add \
  pcre \
  pcre-dev \
  build-base \
  gcc \
  linux-headers \
  openssl \
  libffi-dev \
  && pip install --upgrade pip \
  && pip install pipenv

RUN mkdir -p /var/www/app && mkdir /app

WORKDIR /app

COPY Pipfile* /app
RUN cd /app && pipenv install --system --deploy

# WICHTIG: Erlaubt GCC 15, uwsgi trotz inkompatibler Pointer-Typen zu kompilieren
ENV CFLAGS="-Wno-error=incompatible-pointer-types"
ENV LDFLAGS="-Wno-error=incompatible-pointer-types"

RUN pipenv install --system --deploy

COPY . /app

EXPOSE 5000

CMD uwsgi --http :5000 --master --enable-threads -s /var/www/app/app.sock --manage-script-name --mount /=app:app