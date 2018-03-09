FROM postgres:10

RUN apt-get update && apt-get install -y wget gnupg
RUN echo "deb [arch=amd64] http://packages.2ndquadrant.com/pglogical/apt/ jessie-2ndquadrant main" > /etc/apt/sources.list.d/2ndquadrant.list \
        && wget --quiet -O - http://packages.2ndquadrant.com/pglogical/apt/AA7A6805.asc | apt-key add - \
        && apt-get update \
        && apt-get install -y postgresql-10-pglogical
