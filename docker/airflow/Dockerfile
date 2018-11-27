FROM puckel/docker-airflow:1.10.1

MAINTAINER 'Freckle IoT'

COPY bootstrap /

USER root

# Need to patch the entry point to allow specifying the database number for Redis
RUN apt-get update \
    && apt-get -y install procps redis-tools \
    && pip install awscli psycopg2-binary Flask-Cache Flask-OAuthlib apache-airflow[s3,slack,mysql,postgres,redis,password,databricks] boto3 \
    && sed -i -e 's/\(AIRFLOW__CELERY__BROKER_URL="redis:\/\/\$REDIS_PREFIX\$REDIS_HOST:\$REDIS_PORT\/\)\(1\)"/\1\${REDIS_DB_NUM:-\2}"/' /entrypoint.sh

USER airflow

# Run the bootstrap script before the original entrypoint
ENTRYPOINT ["/bootstrap", "/entrypoint.sh"]

# set default arg for entrypoint
CMD ["webserver"]