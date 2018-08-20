# Introduction to ML-on-Code Workshop

These are materials for a workshop on "Introduction to ML-on-Code" - a guided tour on source{d} open source technology stack for Machine Learning on Code.

Slides [on GDrive](https://docs.google.com/presentation/d/12NdxDQLrtwMu2J-k0HB86I7H-eRDI3N9XDUMvea2Ioc/edit?usp=sharing).


OSS tools covered:
- Public Github Archive: http://pga.sourced.tech/
- Siva: https://github.com/src-d/go-siva#command-line-interface
- source{d} Engine: https://github.com/src-d/engine/
- Project Babelfish: https://doc.bblf.sh/

## Content

  * [Prerequisites](#prerequisites)
  * [Dependencies](#dependencies)
  * [Workflow](#workflow)
     * [1. Play with PublicGithubArchive CLI](#1-play-with-publicgithubarchive-cli)
     * [2. Get used to Siva format](#2-get-used-to-siva-format)
     * [3. Engine (basic queries)](#3-engine-basic-queries)
     * [4. Project Babelfish](#4-project-babelfish)
     * [5. Engine (advanced, UAST)](#5-engine-advanced-uast)
    

## Prerequisites
 - Docker 
 - Go

## Dependencies

Golang for CLI tools: 
```
go get github.com/src-d/datasets/PublicGitArchive/pga
go get -u gopkg.in/src-d/go-siva.v1/...
# add "$GOPATH/bin" to "$PATH"
echo "export PATH=$PATH:$(go env GOPATH)/bin" >> ~/.bash_profile
source ~/.bash_profile
```

Import Docker images (works offline):
```
docker load -i images/engine-jupyter-bblfsh.tgz
docker load -i images/bblfshd-with-drivers.tgz

docker images
```

Run Bblfsh containers:
```
docker run -d --name bblfshd --privileged -p 9432:9432 bblfsh/bblfshd-with-drivers

docker exec -it bblfshd bblfshctl driver list

# if above did not work for some reason, use
docker run -d --name bblfshd --privileged -p 9432:9432 bblfsh/bblfshd
docker exec -it bblfshd bblfshctl driver install --recommended
```

Run Engine container \w Jupyter:
```
docker run --name engine-jupyter -it -p 8080:8080 -v $(pwd)/repositories:/repositories -v $(pwd)/notebooks:/home --link bblfshd:bblfshd srcd/engine-jupyter-bblfsh
```

## Workflow

Workshop is structured as a sequence of steps, each introducing a layer of source{d} technology stack, from bottom up.

<img width="720" alt="Workshop flow" src="https://user-images.githubusercontent.com/5582506/38016881-03b62980-32a3-11e8-9926-2f3d56faf1b3.png">

### 1. Play with PublicGithubArchive CLI

Public Github Playground is a reference dataset of full history of ~180k most popular (>50 stars) projects from Github.

 710 GB of code in 3 TB of packfiles.

```sh
cp -r .pga/latest.csv.gz ~/
pga help

# number of repos from Github
pga list -u github.com/github/ -f json | wc -l

# number of repos from Github in Golang
pga list -u github.com/github/ --lang go -f json | wc -l

# pretty-print src-d repos
pga list -u github.com/src-d/ -f json | jq -r . | less

# URLs and languages for src-d repos \w more then 50 files
pga list -u github.com/src-d/ -f json | jq -r 'select(.fileCount > 50) | .url + " " + .langs[]' | less
```


Materials:
  - http://pga.sourced.tech/
  - https://github.com/src-d/datasets/tree/master/PublicGitArchive/pga
  - https://github.com/src-d/datasets/blob/master/PublicGitArchive/doc/dataset_analysis.md#description-of-the-current-dataset



### 2. Get used to Siva format

[**S**eekable **I**ndexed **B**lock **A**rchiver](https://github.com/src-d/go-siva) file format.

Keeps all files + updates of a single Git repository in 1 file in FS.

```sh
find ./repositories/

# list files in archive
siva list ./repositories/siva/latest/65/65c397a8673c0f4b98e3867e5fd6efdaa7d9ccd2.siva

# extract single file
siva unpack -m=config ./repositories/siva/latest/65/65c397a8673c0f4b98e3867e5fd6efdaa7d9ccd2.siva .
less config

# extract all files (bare Git repository)
siva unpack ./repositories/siva/latest/65/65c397a8673c0f4b98e3867e5fd6efdaa7d9ccd2.siva go-kallax/.git

# list all Git objects
cd go-kallax
git verify-pack -v .git/objects/pack/pack-4a202ad08739b7236f57a3a283f45c27087a99f6.idx

# get a single object
git cat-file -p 72e6129819d6a580512f131f0c8d34cf16ffe4e5
git cat-file -p 63d6012da17573aec5d61d8ba4bae4bf8eab257e
```

Materials:
  - https://github.com/src-d/go-siva#command-line-interface
  - https://blog.sourced.tech/post/siva/
  - https://git-scm.com/book/en/v2/Git-Internals-Packfiles


### 3. Engine (basic queries)

[source{d} engine](https://github.com/src-d/engine/) is a library that allows to query Git repositories in parallele from a cluster of machines using Apache Spark.

To start Apache Spark session:
```sh
spark-shell --packages "tech.sourced:engine:0.5.5"
```

Example of the query:
```scala
from sourced.engine import Engine

Engine(spark, 'siva',
         '/path/to/siva-files')
  .repositories
  .references
  .head_ref
  .files
  .classify_languages()
  .filter("lang = 'java'")
  .select('path',
          'repository_id')
  .write
  .parquet("hdfs://...")
```

Open in browser your [Jupyter Notebook - Engine (basic)](http://localhost:8080/notebooks/Intro%20ML-on-Code%20-%20Python.ipynb#) from a running Docker container.


Materials:
  - https://github.com/src-d/engine#playing-around-with-engine-on-jupyter
  - https://github.com/src-d/engine/blob/master/_examples/pyspark/pyspark-shell-basic.md



### 4. Project Babelfish

<img src="https://avatars2.githubusercontent.com/u/25795418?v=3&s=200f" align="right" width="100px" height="100px" alt="Babelfish logo" />

Project Babelfish provides a universal code parser - contenerized parser infrastructure, to extract uAST representation from the source code text.

Visit http://dashboard.bblf.sh/ to try experiment with uAST representation.

```xpath
(: function names :)
//*[@roleFunction and @roleDeclaration and @roleName and not(@roleArgument)]
    
(: python Docstrings :)
//*[@roleFunction and @roleDeclaration and @roleBody]/*/*[@roleLiteral]
    
(: identifiers :)
//*[@roleIdentifier and not(@roleIncomplete)]
```

Materials:
  - https://blog.sourced.tech/post/announcing_babelfish/
  - https://doc.bblf.sh/
  - https://doc.bblf.sh/using-babelfish/getting-started.html
  - https://doc.bblf.sh/using-babelfish/uast-querying.html
  - https://doc.bblf.sh/uast/roles.html#roles-list


### 5. Engine (advanced, UAST)

Through Engine, it is possible to parse files to uASTs using Bblfsh and then query those with XPath.

Open in browser your [Jupyter Notebook - Engine (advanced)](http://localhost:8080/notebooks/Intro%20ML-on-Code%20-%20Python.ipynb#) from your running Docker container.

Materials:
  - https://github.com/src-d/engine-tour#exploring-public-git-archive-with-sourced-engine
  - https://github.com/src-d/engine/blob/master/_examples/pyspark/pyspark-shell-xpath-query.md
  - https://github.com/src-d/engine/blob/master/examples/notebooks/Example.ipynb


### 6. (TBD) ML: train a model

Use the data, saved from a previous step to train source code identifier embedding model with Tensorflow.

Materials:
  - https://blog.sourced.tech/post/id2vec/
