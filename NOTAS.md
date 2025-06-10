# Notas Minimum Viable Dataspace | MVD

- [Notas Minimum Viable Dataspace | MVD](#notas-minimum-viable-dataspace--mvd)
  - [Despliegue](#despliegue)
    - [Pre-requisitos](#pre-requisitos)
    - [Clonamos el repositorio](#clonamos-el-repositorio)
    - [Construir las imágenes](#construir-las-imágenes)
    - [Desplegar los servicios en el clúster](#desplegar-los-servicios-en-el-clúster)
    - [Seedear](#seedear)

## Despliegue

Estare haciendo todo el despliegue desde WSL y la imágen de Debian 12. En Linux debería funcionar igual.

> [!NOTE]
> La version de Docker que estaremos instalando sera la Docker CE, la cual la estaremos instalando mediante WSL, en mi caso, en una maquina Debian 12.
> Es importante ya que en la empresa no podemos usar Docker Desktop, y la version de Docker CE es la que nos permite ejecutar contenedores sin problemas en WSL.

### Pre-requisitos

Requisitos para el despliegue:

```bash
# Docker
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker

# Terraform
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# JDK 17
sudo apt install openjdk-17-jdk

# Git
sudo apt install git

# OpenSSL
sudo apt install openssl

# Newman
sudo apt install npm
npm install -g newman
```

### Clonamos el repositorio

Clonamos el repositorio de Minimum Viable Dataspace:

```bash
git clone https://github.com/eclipse-edc/MinimumViableDataspace.git && cd MinimumViableDataspace && git checkout remotes/origin/release/0.12.0

```

> [!NOTE]
> En este momento, la rama `release/0.12.0` es la estable.

### Construir las imágenes

Para construir las imágenes de los servicios:

```bash
./gradlew build
./gradlew -Ppersistence=true dockerize
```

Para pushear las imágenes a Nexus:

```bash
# Añadir el repositorio de Nexus
openssl s_client \
  -showcerts \
  -connect srvdocker.lksoutsourcing.local:8083 \
  -servername srvdocker.lksoutsourcing.local \
  </dev/null 2>/dev/null \
  | sed -n '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/ {p; /-----END CERTIFICATE-----/ q}' \
  > srvdocker.crt

sudo mkdir -p /etc/docker/certs.d/srvdocker.lksoutsourcing.local:8083
sudo cp srvdocker.crt /etc/docker/certs.d/srvdocker.lksoutsourcing.local:8083/ca.crt
sudo systemctl restart docker
rm srvdocker.crt

# Iniciar sesión en Nexus
echo "23LKSout23" | docker login -u admin --password-stdin srvdocker.lksoutsourcing.local:8083

# Taggear y pushear las imágenes
docker tag controlplane:latest srvdocker.lksoutsourcing.local:8083/mvd/controlplane:latest
docker tag dataplane:latest srvdocker.lksoutsourcing.local:8083/mvd/dataplane:latest
docker tag catalog-server:latest srvdocker.lksoutsourcing.local:8083/mvd/catalog-server:latest
docker tag identity-hub:latest srvdocker.lksoutsourcing.local:8083/mvd/identity-hub:latest
docker tag issuerservice:latest srvdocker.lksoutsourcing.local:8083/mvd/issuerservice:latest
docker push srvdocker.lksoutsourcing.local:8083/mvd/controlplane:latest
docker push srvdocker.lksoutsourcing.local:8083/mvd/dataplane:latest
docker push srvdocker.lksoutsourcing.local:8083/mvd/catalog-server:latest
docker push srvdocker.lksoutsourcing.local:8083/mvd/identity-hub:latest
docker push srvdocker.lksoutsourcing.local:8083/mvd/issuerservice:latest
```

> [!NOTE]
> Deberías ver estas 6 imágenes al ejecutar el comando `docker images`:
> `controlplane`, `dataplane`, `catalog-server`, `identity-hub`, `issuerservice`

### Desplegar los servicios en el clúster

Modificamos despliege mediante estos comandos para que apunten a las imágenes que hemos construido y pusheado a Nexus:

```bash
sed -i 's|image_pull_policy = "Never"|image_pull_policy = "IfNotPresent"|g' deployment/modules/identity-hub/main.tf
sed -i 's|image             = "identity-hub:latest"|image             = "srvdocker.lksoutsourcing.local:8083/mvd/identity-hub:latest"|g' deployment/modules/identity-hub/main.tf
sed -i 's|image_pull_policy = "Never"|image_pull_policy = "IfNotPresent"|g' deployment/modules/connector/controlplane.tf
sed -i 's|image             = "controlplane:latest"|image             = "srvdocker.lksoutsourcing.local:8083/mvd/controlplane:latest"|g' deployment/modules/connector/controlplane.tf
sed -i 's|image_pull_policy = "Never"|image_pull_policy = "IfNotPresent"|g' deployment/modules/connector/dataplane.tf
sed -i 's|image             = "dataplane:latest"|image             = "srvdocker.lksoutsourcing.local:8083/mvd/dataplane:latest"|g' deployment/modules/connector/dataplane.tf
sed -i 's|image_pull_policy = "Never"|image_pull_policy = "IfNotPresent"|g' deployment/modules/catalog-server/catalog-server.tf
sed -i 's|image             = "catalog-server:latest"|image             = "srvdocker.lksoutsourcing.local:8083/mvd/catalog-server:latest"|g' deployment/modules/catalog-server/catalog-server.tf
sed -i 's|image_pull_policy = "Never"|image_pull_policy = "IfNotPresent"|g' deployment/modules/issuer/main.tf
sed -i 's|image             = "issuerservice:latest"|image             = "srvdocker.lksoutsourcing.local:8083/mvd/issuerservice:latest"|g' deployment/modules/issuer/main.tf
```

Ejecutamos terraform para desplegar los servicios:

```bash
cd deployment
terraform init
terraform apply
cd ..
```

> [!NOTE]
> Por lo que he visto, el despliegue falla más que una escopeta de feria. Prueba haciendo `kubectl delete ns mvd` y vuelve a ejecutar el `terraform apply`.

### Seedear

Para seedear he modificado el script `seed-k8s.sh` generando uno nuevo llamado `seed-k8s-remote.sh` que contiene todo lo necesarios para seedear el MVD en un clúster remoto.

```bash
sudo chmod +x seed-k8s-remote.sh
./seed-k8s-remote.sh 192.168.5.251
```

> [!NOTE]
> En nuestro cluster, 192.168.5.251 es la IP del Ingress Controller, en caso de que esto haya cambiado, deberás de modificar la IP con la que llamas al script.
