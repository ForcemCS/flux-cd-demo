FROM  ubuntu

WORKDIR /opt/facebooc

COPY . .

RUN  apt-get update &&  \
     apt-get install -yq build-essential make git libsqlite3-dev sqlite3 && \ 
     make all 

EXPOSE 16000

CMD bin/facebooc

