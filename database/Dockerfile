FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

COPY CreateScheme.sql /docker-entrypoint-initdb.d/01-create-scheme.sql
COPY InsertData.sql /docker-entrypoint-initdb.d/02-insert-data.sql

VOLUME db-data:/var/lib/postgresql/data

