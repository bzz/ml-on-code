Images, created for workshop to be used offline.

# Create bblfshd + drivers image

```
docker run -d --name bblfshd --privileged -p 9432:9432 bblfsh/bblfshd
docker exec -it bblfshd bblfshctl driver install --recommended
docker exec -it bblfshd bblfshctl driver list

docker commit 4df5e27b75b0 bblfsh/bblfshd-with-drivers
docker save -o images/bblfshd-with-drivers.tgz bblfsh/bblfshd-with-drivers
```


# Create Engine image

```sh
docker run --name engine-jupyter -it -p 8080:8080 -v $(pwd)/repositories:/repositories -v $(pwd)/notebooks:/home --link bblfshd:bblfshd srcd/engine-jupyter

docker exect -it engine-jupyter /bin/bash

# https://github.com/bblfsh/client-python#dependencies
apt-get update
apt install curl
apt install libxml2-dev
apt install build-essential
pip install bblfsh
apt-get remove --purge build-essential
apt-get autoremove

docker commit 3075a5a3eca7 srcd/engine-jupyter-bblfsh
docker save -o images/engine-jupyter-bblfsh.tgz srcd/engine-jupyter-bblfsh
```
