*Read this in other languages: [中国](README-cn.md).*
# Building a Recommender using Data Science Experience, Apache Spark and HDP Search(Solr) 

Recommendation engines are one of the most well known, widely used and highest value use cases for applying machine learning. Despite this, while there are many resources available for the basics of training a recommendation model, there are relatively few that explain how to actually deploy these models to create a large-scale recommender system.

This Code Pattern demonstrates the key elements of creating such a system, using Apache Spark and HDP Search(Solr)

This repo contains a Jupyter notebook illustrating how to use Spark for training a collaborative filtering recommendation model from ratings data stored in Solr, saving the model factors to Solr, and then using Solr to serve real-time recommendations using the model. The data you will use comes from [MovieLens](https://grouplens.org/datasets/movielens/) and is a common benchmark dataset in the recommendations community. The data consists of a set of ratings given by users of the MovieLens movie rating system, to various movies. It also contains metadata (title and genres) for each movie.

When you have completed this Code Pattern, you will understand how to:

* Ingest and index user event data into Solr using the Solr Spark connector
* Load event data into Spark DataFrames and use Spark's machine learning library (MLlib) to train a collaborative filtering recommender model
* Export the trained model into Solr 
* Using a custom Solr plugin, compute _personalized user_ and _similar item_ recommendations and combine recommendations with search and content filtering

![Architecture diagram](doc/source/images/architecture.png)

## Flow
1. Load the movie dataset into Spark.
2. Use Spark DataFrame operations to clean up the dataset and load it into Solr.
3. Using Spark MLlib, train a collaborative filtering recommendation model.
4. Save the resulting model into Solr.
5. Using Solr queries and a custom vector scoring plugin, generate some example recommendations. [The Movie Database](https://www.themoviedb.org/) API is used to display movie poster images for the recommended movies.

## Included components
* [Apache Spark](http://spark.apache.org/): An open-source, fast and general-purpose cluster computing system
* [Horton works platform](https://hortonworks.com) : Apache Hadoop and Big Data Platform for a Data Driven Enterprise
* [HDP Search](https://doc.lucidworks.com/lucidworks-hdpsearch/2.6/Guide-Install.html): Open-source search and analytics engine
* [Jupyter Notebooks](http://jupyter.org/): An open-source web application that allows you to create and share documents that contain live code, equations, visualizations and explanatory text. T

## Featured technologies
* [Data Science Local](https://content-dsxlocal.mybluemix.net/docs/content/local/welcome.html): Systems and scientific methods to analyze
structured and unstructured data in order to extract knowledge and insights.
* [Artificial Intelligence](https://medium.com/ibm-data-science-experience): Artificial intelligence can be applied to disparate solution spaces to deliver disruptive technologies.
* [Python](https://www.python.org/): Python is a programming language that lets you work more quickly and integrate your systems more effectively.

# Steps
Follow these steps to create the required services and run the notebook locally.

1. [Clone the repo](#1-clone-the-repo)
2. [Set up HDP platform](#2-set-up-hdp-platform)
3. [Set up HDP Search](#2-set-up-hdp-search)
4. [Download the Solr Spark connector](#3-download-the-solr-spark-connector)
5. [Download and move data to HDFS](#5-download-and-move-the-data)
6. [Launch the notebook](#6-launch-the-notebook)
7. [Run the notebook](#7-run-the-notebook)

### 1. Clone the repo

Clone the `hdp-search-spark-recommender` repository locally. In a terminal, run the following command:

```
$ git clone https://github.com/IBM/hdp-search-spark-recommender.git
```

### 2. Setup HDP platform
(Fill in)

### 3. Set up HDP Search

Install the HDP Search component by following the instructions [instructions](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.4/bk_solr-search-installation/content/hdp-search30-install-mpack.html). The code pattern currently depends on solr version 6.6.2. The code pattern also assumes that the Solr is started in the cloud mode. After installing the HDP Search(Solr) component, verify that the server is running by following one of the two ways.

   1. Access the solr admin URL http://solr_host:8983/solr to see the dashboard.
   2. cd <solr_installation_dir>/solr/bin; ./solr status 

Next, you will need to install the [Solr vector scoring plugin](https://github.com/saaay71/solr-vector-scoring). Please follow the installation instructions on the page to install the plugin. 

Next, restart solr from Ambari.


### 3. Download the Solr Spark connector

This code pattern reads the movie data set and computes the model vectors using spark. The spark dataframes are then written to
Solr by using the Solr Spark connector. The solr connector for spark can be downloaded from [spark-solr-connector](https://github.com/lucidworks/spark-solr). This code pattern has been tested with connector version 3.3.3.The spark configurations need to be changed to add the connector jars in the spark driver and executor classpath. This can be done via by following the steps below.

   1. Log in to Ambari 
   2. Select spark2 component
   3. Choose configuration tab and add the following property keys under "Custom spark2-defaults"
      * spark.driver.extraClassPath -> Path to the spark solr connector jar.
        Example :
        spark.driver.extraClassPath /home/user1/spark-solr-3.3.3-shaded.jar
      * spark.executor.extraClassPath -> Path to the spark solr connector jar
      * spark.jars -> Path to the spark solr connector jar

**Note:**
In the example above, the path is a local system path. In case of a multi node HDP cluster, please make sure that this jar is
available under same path in all the nodes. Another option is to put the jar in HDFS and specify the HDFS location.


You will also need to install [Numpy](http://www.numpy.org) in order to use Spark's machine learning library, [MLlib](http://spark.apache.org/mllib). If you don't have Numpy installed, run:
```
$ pip install numpy
```

### 5. Download the data

You will be using the [Movielens dataset](https://grouplens.org/datasets/movielens/) of ratings given by a set of users to movies, together with movie metadata. There are a few versions of the dataset. You should download the ["latest small" version](http://files.grouplens.org/datasets/movielens/ml-latest-small.zip).

Run the following commands from the base directory of the Code Pattern repository:

```
$ cd data
$ wget http://files.grouplens.org/datasets/movielens/ml-latest-small.zip
$ unzip ml-latest-small.zip
```

This code pattern is targeted to run against a multi node HDP cluster. Therefore the dataset needs to be moved to HDFS. Move the data
to hdfs by issuing the ```hadoop fs -put``` command. 

### 6. Launch the notebook

> The notebook should work with Python 2.7 or 3.x (and has been tested on 2.7.11 and 3.6.1)

To run the notebook you will need to start DSX Local. Below are the steps that need to be performed before running the notebook
the very first time.
   1. Import the notebook into DSX Local from the github location.
   2. Fill in the values such as solr install location, solr install host name , ssh userid/password, tmdb access key etc.
   3. Run the notebook. 
```

### 7. Run the notebook

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

![](doc/source/images/notebook-run-cells.png)

# Sample output

The example output in the `data/examples` folder shows the output of the notebook after running it in full. View it it [here]().

> *Note:* To see the code and markdown cells without output, you can view [the raw notebook in the Github viewer](notebooks/elasticsearch-spark-recommender.ipynb).

# Troubleshooting
<fill-in>
# Learn more

* **Data Analytics Code Patterns**: Enjoyed this Code Pattern? Check out our other [Data Analytics Code Patterns](https://developer.ibm.com/code/technologies/data-science/)
* **AI and Data Code Pattern Playlist**: Bookmark our [playlist](https://www.youtube.com/playlist?list=PLzUbsvIyrNfknNewObx5N7uGZ5FKH0Fde) with all of our Code Pattern videos
* **Watson Studio**: Master the art of data science with IBM's [Watson Studio](https://datascience.ibm.com/)
* **Spark on IBM Cloud**: Need a Spark cluster? Create up to 30 Spark executors on IBM Cloud with our [Spark service](https://console.bluemix.net/catalog/services/apache-spark)

# License
[Apache 2.0](LICENSE)
