# Building a Recommender using Data Science Experience Local and HortonWorks Data Platform

Recommendation engines are one of the most well known, widely used and highest value use cases for applying machine learning. Despite this, while there are many resources available for the basics of training a recommendation model, there are relatively few that explain how to actually deploy these models to create a large-scale recommender system.

This Code Pattern demonstrates the key elements of creating such a system by using Apache Spark and HDP Search (Solr). Note that this code pattern is a port of Nick Pentreath's [Recommender built with Elasticsearch and Apache Spark](https://github.com/IBM/elasticsearch-spark-recommender), but with a focus on using HortonWorks Data Platform (HDP) and IBM's Data Science Experience Local (DSX Local).

> **What is HDP and HDP Search?** HortonWorks Data Platform (HDP) is a massively scalable platform for storing, processing and analyzing large volumes of data. HDP consists of the essential set of Apache Hadoop projects including MapReduce, Hadoop Distributed File System (HDFS), HCatalog, Pig, Hive, HBase, Zookeeper and Ambari. HDP Search provides applications and tools for indexing content from your HDP cluster to Solr (an open source enterprise search platform).

  ![](images/hdp_arch.png)

   *Hortonworks Data Platform by [Hortonworks](https://hortonworks.com/products/data-platforms/hdp/)*

> **What is DSX Local?** DSX Local is an on premises solution for data scientists and data engineers. It offers a suite of data science tools that integrate with RStudio, Spark, Jupyter, and Zeppelin notebook technologies. And yes, it can be configured to use HDP, too.

This repo contains a Jupyter notebook illustrating how to use Spark for training a collaborative filtering recommendation model from ratings data stored in Solr, saving the model factors to Solr, and then using Solr to serve real-time recommendations using the model. The data you will use comes from [MovieLens](https://grouplens.org/datasets/movielens/) and is a common benchmark dataset in the recommendations community. The data consists of a set of ratings given by users of the MovieLens movie rating system, to various movies. It also contains metadata (title and genres) for each movie.

When you have completed this Code Pattern, you will understand how to:

* Ingest and index user event data into Solr using the Solr Spark connector
* Load event data into Spark DataFrames and use Spark's machine learning library (MLlib) to train a collaborative filtering recommender model
* Export the trained model into Solr 
* Using a custom Solr plugin, compute _personalized user_ and _similar item_ recommendations and combine recommendations with search and content filtering

## Flow

![](images/architecture.png)

1. Load the movie dataset into Spark.
2. Use Spark DataFrame operations to clean up the dataset and load it into Solr.
3. Using Spark MLlib, train a collaborative filtering recommendation model.
4. Save the resulting model into Solr.
5. Using Solr queries and a custom vector scoring plugin, generate some example recommendations. [The Movie Database](https://www.themoviedb.org/) API is used to display movie poster images for the recommended movies.

## Included components

* [IBM Data Science Experience Local](https://content-dsxlocal.mybluemix.net/docs/content/local/overview.html): An out-of-the-box on premises solution for data scientists and data engineers. It offers a suite of data science tools that integrate with RStudio, Spark, Jupyter, and Zeppelin notebook technologies.
* [Apache Spark](http://spark.apache.org/): An open-source, fast and general-purpose cluster computing system.
* [Hortonworks Data Platform (HDP)](https://hortonworks.com/products/data-platforms/hdp/): HDP is a massively scalable platform for storing, processing and analyzing large volumes of data. HDP consists of the essential set of Apache Hadoop projects including MapReduce, Hadoop Distributed File System (HDFS), HCatalog, Pig, Hive, HBase, Zookeeper and Ambari.
* [HDP Search](https://doc.lucidworks.com/lucidworks-hdpsearch/2.6/index.html): HDP Search provides applications and tools for indexing content from your HDP cluster to Solr.
* [Apache Livy](https://livy.incubator.apache.org/): Apache Livy is a service that enables easy interaction with a Spark cluster over a REST interface.
* [Jupyter Notebooks](http://jupyter.org/): An open-source web application that allows you to create and share documents that contain live code, equations, visualizations and explanatory text.

## Featured technologies

* [Artificial Intelligence](https://medium.com/ibm-data-science-experience): Artificial intelligence can be applied to disparate solution spaces to deliver disruptive technologies.
* [Python](https://www.python.org/): Python is a programming language that lets you work more quickly and integrate your systems more effectively.

# Steps
Follow these steps to create the required services and run the notebook locally.

1. [Setup HDP platform](#1-set-up-hdp-platform)
2. [Setup HDP Search](#2-set-up-hdp-search)
3. [Setup DSX Local](#3-set-up-dsx-local)
4. [Setup a Livy gateway](#4-set-up-a-livy-gateway)
5. [Download the Solr Spark connector](#5-download-the-solr-spark-connector)
6. [Clone the repo](#6-clone-the-repo)
7. [Download and move the data to HDFS](#7-download-and-move-the-data-to-hdfs)
8. [Setup python plugins](#8-setup-python-plugins)
9. [Launch the notebook](#9-launch-the-notebook)
10. [Run the notebook](#10-run-the-notebook)

### 1. Setup HDP platform

Due to the various deployment methods of HDP we will cover the method we used when developing the code pattern, which was to use a Dockerized HDP Sandbox. This can be accomplished by performing the following steps:

1. Navigate to [https://hortonworks.com/downloads/#sandbox](https://hortonworks.com/downloads/#sandbox) and choose the `Docker Linux/Mac` option. Note, you may be required to create a free account. This will download a `start-sandbox-hdp-standalone_2-6-4.sh.zip` file to your OS.

    ![](images/hdp-dockerized.png)

1. At this point start the Docker daemon and simply unzip the file to execute the script using:

    ```
    unzip start-sandbox-hdp-standalone_2-6-4.sh.zip
    sh start-sandbox-hdp-standalone_2-6-4.sh
    ```

A Dockerized version of HDP v2.6.4 should now be up and running when the script is complete. The full documentation can be seen in the [HDP install guide](https://hortonworks.com/tutorial/sandbox-deployment-and-install-guide/section/3/).

### 2. Set up HDP Search

To install HDP Search, follow the [HDP Search instructions](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.4/bk_solr-search-installation/content/hdp-search30-install-mpack.html). This code pattern was tested with Solar v6.6.2 and assumes that Solr is started in cloud mode. Minimally, this can be accomplished by performing the following steps:

1. Download the Ambari management pack to the Ambari Server host:

    ```
    wget http://public-repo-1.hortonworks.com/HDP-SOLR/hdp-solr-ambari-mp/solr-service-mpack-3.0.0.tar.gz
    ```

1. Install the management pack on the Ambari Server host:

    ```
    ambari-server install-mpack --mpack=/tmp/solr-service-mpack-3.0.0.tar.gz
    ```

1. Restart the Ambari server.

    ```
    ambari-server restart
    ```

1. After installing HDP Search (Solr), verify that the server is running by performing one of the two methods.

   1. Access the Solr admin dashboard:
      ```
      http://solr_host:8983/solr
      ```

      OR

   1. Run the `solr status` command:
      ```
      cd <solr_installation_dir>/solr/bin
      ./solr status
      ```

1. The last step is to install the [Solr vector scoring plugin](https://github.com/saaay71/solr-vector-scoring). Please follow the installation instructions on the page to install the plugin.

1. Finally, restart the Ambari server one last time.

    ```
    ambari-server restart
    ```

### 3. Setup DSX Local

This code pattern was tested using [DSX Desktop](https://www.ibm.com/products/data-science-experience), a lightweight version of DSX Local intended for standalone use and optimized for local development. For production deployments it is recommended to use DSX Local with a three node configuration. For information on how to do that, see the [DSX Install Docs](https://content-dsxlocal.mybluemix.net/docs/content/local/welcome.html).

Additionally, DSX Local provides documentation on how to set up HDP to work with DSX! Check out [Set up an HDP cluster for DSX](https://content-dsxlocal.mybluemix.net/docs/content/local/hdp.html) for more details.

### 4. Setup a Livy gateway

*Coming soon*

### 5. Download the Solr Spark connector

This code pattern reads the movie data set and computes the model vectors using Spark. The Spark dataframes representing the movie data set and model vectors are then written to Solr by using the Solr Spark connector. The Solr connector for Spark can be downloaded from [Lucidworks' Spark-Solr repo](https://github.com/lucidworks/spark-solr).

> This code pattern was tested with version 3.3.3 of the connector. The Spark configurations need to be changed to add the connector jars in the Spark driver and executor classpath. This can be done via by following the steps below.

   1. Log into Ambari
   2. Select spark2 component
   3. Choose configuration tab and add the following property keys under "Custom spark2-defaults"
      * `spark.driver.extraClassPath` -> Path to the spark solr connector jar.
        Example: `spark.driver.extraClassPath` -> `/home/user1/spark-solr-3.3.3-shaded.jar`
      * `spark.executor.extraClassPath` -> Path to the spark solr connector jar
      * `spark.jars` -> Path to the spark solr connector jar

> Note: In the example above, the path is a local system path. In case of a multi node HDP cluster, please make sure that this jar is available under same path in all the nodes. Another option is to put the jar in HDFS and specify the HDFS location.

### 6. Clone the repo

Now that our HDP and DSX Local installation is complete, we can proceed with making the two communicate with each other. We start by cloning the `hdp-search-spark-recommender` repository locally. In a terminal, run the following command:

```
git clone https://github.com/IBM/hdp-search-spark-recommender.git
```

### 7. Download and move the data to HDFS

In this code pattern we will be using the [Movielens dataset](https://grouplens.org/datasets/movielens/), it contains ratings given by a set of users for movies, as well as movie metadata. For the purpose of this code pattern, we can download the ["latest small" version](http://files.grouplens.org/datasets/movielens/ml-latest-small.zip) of the data set.

Run the following commands from the base directory of the code pattern repository to download the data:

```
cd data
wget http://files.grouplens.org/datasets/movielens/ml-latest-small.zip
unzip ml-latest-small.zip
```

This code pattern is targeted to run against a multi node HDP cluster. Therefore the dataset needs to be moved to HDFS. Move the data
to HDFS by issuing the following command:

```
hadoop fs -put
```

### 8. Setup python plugins

This code pattern relies upon a few python plugins for everything to work together. Some plugins are required to be installed in the node where DSX Local is installed and the others need to be installed in all the HDP compute nodes. The following table describes the details where these plugins are to be installed.

| Library | Install Location |
| ------------- | ------------- |
| tmdbsimple | DSX Local Node |
| IPython  | Need to be installed on DSX Local node |
| paramiko | All nodes of HDP cluster | 
| numpy | All nodes of HDP cluster |
| simplejson | DSX Local and all nodes of HDP cluster |
| urllib2 | DSX Local and all nodes of HDP cluster |
| solrcloudpy | DSX Local and all nodes of HDP cluster |

The plugins can be installed using the pip command:

```
pip install numpy
```

> Note: Some of the plugins may already be installed/present in your environment. In that case please skip that plugin and move to the next plugin in the table.

### 9. Launch the notebook

> The notebook provided in this repo should work with Python 2.7.x or 3.x (and has been tested on 2.7.11 and 3.6.1)

To run the notebook you will need to start DSX Local. Below are the steps that need to be performed before running the notebook
the very first time.

1. Create a new project.

    ![](images/dsx-local-create-project-button.png)
    ![](images/dsx-local-create-project.png)

1. Create the notebook into DSX Local from the Github location.

    ```
    https://raw.githubusercontent.com/IBM/hdp-search-spark-recommender/master/notebooks/solr-hdp-spark-recommender.ipynb
    ```

    ![](images/dsx-local-create-notebook-button.png)
    ![](images/dsx-local-create-notebook.png)

1. Update the following variables in the notebook:

    * `SOLR_INSTALL_DIR`
    * `SOLR_HOST`
    * `SOLR_PORT`
    * `ZKHOST`
    * `SSHUSER`
    * `SSHPASSWORD`
    * `tmdb.API_KEY`

1. Run the notebook!

### 10. Run the notebook

When a notebook is executed, what is actually happening is that each code cell in
the notebook is executed, in order, from top to bottom.

Each code cell is selectable and is preceded by a tag in the left margin. The tag
format is `In [x]:`. Depending on the state of the notebook, the `x` can be:

* A blank, this indicates that the cell has never been executed.
* A number, this number represents the relative order this code step was executed.
* A `*`, this indicates that the cell is currently executing.

There are several ways to execute the code cells in your notebook:

* One cell at a time.
  * Select the cell, and then press the `Play` button in the toolbar. You can also hit `Shift+Enter` to execute the cell and move to the next cell.
* Batch mode, in sequential order.
  * From the `Cell` menu bar, there are several options available. For example, you
    can `Run All` cells in your notebook, or you can `Run All Below`, that will
    start executing from the first cell under the currently selected cell, and then
    continue executing all cells that follow.

![](images/dsx-local-run-all.png)

# Sample output

After classifying the data by movies, ratings and genres, we train our recommender model. 

![](images/ratings-table.png)

![](images/genre-table.png)

After loading our model into Solr, we can predict which movies people will like based on ratings of similar movies in the same genre.

![](images/movie-list.png)

The example output in the [data/examples](data/examples) folder shows the output of the notebook after running it in full. View it [here](data/examples/solr-hdp-spark-recommender.ipynb).

# Troubleshooting

* Should DSX Desktop fail to respond at any time perform the following steps:
  * Close IBM DSX desktop
  * Kill all the processes related to DSX Local from terminal:

    ```
    docker ps |grep dsx
    docker rm -f <container id>
    ```

  * Remove DSX Desktop metadata

    ```
    cd ~/Library/Application \Support/
    rm -rf ibm-dsx-desktop
    ```

  * Restart DSX Desktop
  * Load the jupyter notebook again

# Links

* [Teaming on Data: IBM and Hortonworks Broaden Relationship](https://hortonworks.com/blog/teaming-data-ibm-hortonworks-broaden-relationship/)
* [Certification of IBM Data Science Experience (DSX) on HDP is a Win-Win for Customers](https://hortonworks.com/blog/certification-ibm-data-science-experience-dsx-hdp-win-win-customers/)
* [An Exciting Data Science Experience on HDP](https://hortonworks.com/blog/exciting-data-science-experience-hdp/)

# Learn more

* **Data Analytics Code Patterns**: Enjoyed this Code Pattern? Check out our other [Data Analytics Code Patterns](https://developer.ibm.com/code/technologies/data-science/)
* **AI and Data Code Pattern Playlist**: Bookmark our [playlist](https://www.youtube.com/playlist?list=PLzUbsvIyrNfknNewObx5N7uGZ5FKH0Fde) with all of our Code Pattern videos
* **Watson Studio**: Master the art of data science with IBM's [Watson Studio](https://datascience.ibm.com/)
* **Spark on IBM Cloud**: Need a Spark cluster? Create up to 30 Spark executors on IBM Cloud with our [Spark service](https://console.bluemix.net/catalog/services/apache-spark)

# License
[Apache 2.0](LICENSE)
