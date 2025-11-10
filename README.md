https://iam-127-0-0-1.nip.io/auth-admin

# Airlock IAM Podman / Podman Desktop Example

## Why should I use it instead of the classial approach via VM?

### Ressource consumption (after creation in IDLE)
* MariaDB: vRAM 108MB, Volume 175MB
* IAM: vRAM 1,06GBm Volume 4MB

Please add your [quay user](https://docs.airlock.com/iam/latest/index/1589475710428.html#Pulling_the_IAM_image_from_an_image_repository) which is required to downlaod the IAM container from the repo.

```bash
podman login -u='quay user' -p='encrypted quay password' quay.io
```

## Create a Pod for easier management:

```bash
podman pod create \
  --name iam \
  -p 8443:8443 \
  #-p 3306:3306 # optional port for MariaDB access outside of the POD
```

## Create the MariaDB and IAM container.

### Create folders if they are not already available

* You have to correct the permission of the folder for MariaDB as well as IAM. In order to figure out the Userid, run the following commands.

```bash
podman run --rm --entrypoint id docker.io/library/mariadb:11.8.3-ubi
podman run --rm --entrypoint id quay.io/airlock/iam:8.5.0
```

Create the folder and set the correct permissions.

```bash
mkdir -p ~/ContainerDataFolder/mariadb ~/ContainerDataFolder/iam
podman unshare chown -R 999:999 ~/ContainerDataFolder/mariadb
podman unshare chown -R 1000:0 ~/ContainerDataFolder/iam

```

### Create the MariaDB Container

```bash
podman create \
  --name mariadb \
  --pod iam \
  -e MARIADB_ROOT_PASSWORD="ND+w\Y2CWvW}>gn-s)H_" \
  -v ~/ContainerDataFolder/mariadb:/var/lib/mysql:Z \
  docker.io/library/mariadb:11.8.3-ubi
```

### Create the IAM Container

```bash
podman create \
  --name airlock-iam \
  --pod iam \
  --memory 4g \
  -e IAM_MODULES=adminapp \
  -e TZ=Europe/Berlin \
  -e IAM_JAVA_OPTS='-XX:MaxRAMPercentage=50' \
  -v ~/ContainerDataFolder/iam:/home/airlock/iam:Z \
  quay.io/airlock/iam:8.5.0 \
  run -i auth

```

### Run the init container to create the instance

```bash
podman run \
  --rm \
  -v ~/ContainerDataFolder/iam:/home/airlock/iam:Z \
  quay.io/airlock/iam:8.5.0 \
  init --instance auth --analytics LICENSE_DATA
```

### Start the IAM Pod.

```bash
podman pod start iam
```

## Preapare Mariadb as well as the IAM container.

Get the latest version of the MariaDB [connector](https://mariadb.com/docs/connectors/mariadb-connector-j/about-mariadb-connector-j)

```bash
LATEST=$(curl -fsSL https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/maven-metadata.xml \
  | grep -Po '(?<=<release>)[^<]+')
curl -fLO "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/${LATEST}/mariadb-java-client-${LATEST}.jar"
curl -fsSL "https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/${LATEST}/mariadb-java-client-${LATEST}.jar.sha256" \
  | awk -v f="mariadb-java-client-${LATEST}.jar" '{print $1"  "f}' \
  | sha256sum -c -
```

Copy the MarioDB connector to the Airlock IAM instance

```bash
podman cp mariadb-java-client-*.jar airlock-iam:instances/common/libs/
```

Adjust the Database and create the needed Table and User.

```bash
podman exec -e MYSQL_PWD="ND+w\Y2CWvW}>gn-s)H_" -it mariadb \
  mariadb -uroot -e "\
    CREATE DATABASE IF NOT EXISTS airlockiam CHARACTER SET utf8mb4;
    CREATE USER IF NOT EXISTS 'airlockiam'@'%' IDENTIFIED BY '123456';
    GRANT ALL PRIVILEGES ON airlockiam.* TO 'airlockiam'@'%';
    FLUSH PRIVILEGES;"
```

Restart of the DB is required to active the previous made changes.

```bash
podman restart mariadb
```

Applying of the Meduse Table Schema

```bash
podman exec -e MYSQL_PWD="123456" -i mariadb \
  mariadb -u airlockiam airlockiam < ressources/create-medusa-schema.sql
```

Creation of the IAM default admin

```bash
podman exec -e MYSQL_PWD="123456" -i mariadb \
  mariadb -u airlockiam airlockiam < ressources/insert-admin.sql
```

Verification of the table schema.

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb \
  mariadb -u airlockiam -D airlockiam -e "SHOW TABLES;"
```

Verification of the admin user.

```bash
podman exec -e MYSQL_PWD=123456 -it mariadb \
  mysql -u airlockiam -D airlockiam \
  -e "SELECT username, givenname, surname, roles, pwd_chg_enf FROM medusa_admin WHERE username='admin';"
```

## Login and start with the basic configuration

You can enter the IAM via https://127.0.0.1:8443/auth-admin
> As the DB is not connected, you can use the user admin with any password you like, as it will not get checked.