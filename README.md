cartodb installation instruction on ubuntu 12.x 

    aptitude install software-properties-common (for 12.10)
    apt-get install python-software-properties (for 12.04)
    add-apt-repository ppa:mapnik/nightly-2.1
    add-apt-repository ppa:ubuntugis/ppa

refresh packages:

    apt-get update && apt-get upgrade 

install dependencies for cartodb installation and few other helper tools

    apt-get install g++ make checkinstall build-essential htop git unp zip gdal-bin \
    libgdal1-dev libjson0 python-simplejson libjson0-dev proj-bin proj-data libproj-dev \
    postgresql-9.1 postgresql-client-9.1 postgresql-contrib-9.1 postgresql-server-dev-9.1 \
    postgresql-plpython-9.1 curl redis-server python-pip python-dev python-gdal libmapnik-dev \
    python-mapnik2 mapnik-utils libxslt-dev libgeos-c1 libgeos-dev pgbouncer varnish git npm

postgis installation:

    wget http://download.osgeo.org/postgis/source/postgis-2.0.3.tar.gz
     tar -xvf postgis-2.0.3.tar.gz 
     cd postgis-2.0.3/
     ./configure --with-raster --with-topology
     make && make install

CartoDB depends on a geospatial database template named template_postgis. change to postgres user (sudo su postgres)

    cd /usr/share/postgresql/9.1/contrib/postgis-2.0/
    createdb -E UTF8 template_postgis
    createlang -d template_postgis plpgsql
    psql -d postgres -c \
    "UPDATE pg_database SET datistemplate='true' WHERE datname='template_postgis'"
    psql -d template_postgis -f $POSTGIS_SQL_PATH/postgis.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/spatial_ref_sys.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/legacy.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/rtpostgis.sql
    psql -d template_postgis -f $POSTGIS_SQL_PATH/topology.sql
    psql -d template_postgis -c "GRANT ALL ON geometry_columns TO PUBLIC;"
    psql -d template_postgis -c "GRANT ALL ON spatial_ref_sys TO PUBLIC;"

edit /etc/postgresql/9.1/main/pg_hba.conf and "trust" local connections

edit pgbouncer config /etc/pgbouncer/pgbouncer.ini and change settings as needed

    [databases]
    postgres = host=127.0.0.1 port=5432
    * = host=127.0.0.1 port=5432
    [pgbouncer]
    listen_addr = 127.0.0.1
    listen_port = 6432
    auth_file = /etc/pgbouncer/userlist.txt
    admin_users = postgres
    stats_users = postgres,root
    
Download and install node 0.8.9 from github

    git clone git://github.com/joyent/node.git node0.8.9
    cd node0.8.9
    git checkout v0.8.9
    ./configure --prefix=/usr/local/node0.8.9
    make
    sudo make install
    cd ..

Include node to PATH variable

    export PATH="/usr/local/node/bin":"${PATH}"
    
Link node to node0.8.9

    sudo ln -s /usr/local/node0.8.9 /usr/local/node

Adding/changing /etc/sudoers.d/secure_path:

    Defaults  secure_path=/usr/local/node/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/X11R6/bin
    
If you had to create the secure_path file, you will need to update the permissions

    sudo su
    sudo chmod 0440 /etc/sudoers.d/secure_path
    exit
    
Install node-inspector from NPM repository

    npm install node-inspector -g

Get the latest cartodb

    git clone --recursive https://github.com/CartoDB/cartodb20.git

The CartoDB SQL API component powers the SQL queries over HTTP. To install it:

    git clone git://github.com/CartoDB/CartoDB-SQL-API.git
    cd CartoDB-SQL-API
    git checkout master
    npm install

Link missing sigc++ config
    
    cd /usr/include/sigc++-2.0
    sudo ln -s /usr/lib/x86_64-linux-gnu/sigc++-2.0/include/sigc++config.h .


The Windshaft-cartodb component powers the CartoDB Maps API. To install it:

    git clone git://github.com/CartoDB/Windshaft-cartodb.git
    cd Windshaft-cartodb
    git checkout master
    npm install

Install RVM and bundle cartodb: # cd to cartodb20/

    \curl -L https://get.rvm.io | bash -s stable --ruby=1.9.2
    rvm rvmrc warning ignore /home/cartodb/cartodb20/.rvmrc
    #rvm install 1.9.2 
    rvm use 1.9.2@cartodb --create && bundle install

add /usr/lib/postgresql/9.1/bin to the user path running the 

test if everything is working ok

    bundle exec foreman start -p 3000

comment out redis service from Procfile since it is started by init scripts on boot and install upstart

    rvmsudo foreman export upstart /etc/init -a cartodb -u cartodb
    rvmsudo passenger-install-nginx-module (install nginx)

edit /etc/init/cartodb.conf and start stops

    start on runlevel [2345]
    stop on runlevel [016]

varnish config /etc/varnish/default.vcl

       backend default {
        .host = "127.0.0.1";
        .port = "3000";
        }
        sub vcl_recv {     
        return (lookup);
        }
        
        DAEMON_OPTS="-a :80 \
                 -T localhost:6082 \
                 -f /etc/varnish/default.vcl \
                 -S /etc/varnish/secret \
                 -s malloc,1G"


Installation Based on the original instructions from https://github.com/CartoDB/cartodb
