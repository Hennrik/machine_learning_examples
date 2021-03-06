#Machine Learning with Apache Spark

##Index

* [Overview of ML with Spark](#Spark-Machine-Learning-Overview)
* [Implement ML with Spark](#Spark-Machine-Learning)
* [Model Evaluation](#ML-Tricks-With-Spark)
* [Combine with High Order Functions](#Scala-Functional)

<h2 id='Spark-Machine-Learning-Overview'>Spark Machine Learning Overview</h2>

MLlib is Spark’s machine learning (ML) library. Its goal is to make practical machine learning scalable and easy. It consists of common learning algorithms and utilities, including classification, regression, clustering, collaborative filtering, dimensionality reduction, as well as lower-level optimization primitives and higher-level pipeline APIs [1].

Recently, spark 1.6 was released, **dataframe** has become the most important data structure instead of **RDD** in the past, and, a number of new machine learning algorithms also available in this new version of apache spark.

Spark-1.6 with pre-built hadoop can be found in TU Delft HTTP, download [here](http://ftp.tudelft.nl/apache/spark/spark-1.6.0/spark-1.6.0-bin-hadoop2.6.tgz).

In this section, we will cover these algorithms: 

- linear models
  + linear regression
  + logistic regression
  + linear svm (max-margin classifier)
- naive bayes
- clustering
  + k-means
- dimensionality reduction
  + principal component analysis (PCA)

<h2 id='Spark-Machine-Learning'>Implement Machine Learning with Spark</h2>

###Linear Regression

Because spark is build for large scale machine learning system, so in mllib it only provided *Stochastic Gradient Descent* linear regression, which is named *linearRegressionWithSGD*.

Load data from csv file and remove the first row:

```scala
val csvPath = "/Users/wbcha/Downloads/MachineLearning-master/Example Data/OLS_Regression_Example_3.csv"
val rawData = sc.textFile(csvPath)
val dataWithoutHead = rawData.mapPartitionsWithIndex{(idx, iter) => if (idx == 0) iter.drop(1) else iter}
```

Split data with comma and transformation to US metrics:

```scala
val dataNewMetrics = dataWithoutHead.map(s => (s.split(",")(0), s.split(",")(1).toDouble * 2.54, s.split(",")(2).toDouble * 0.45359237))
```

Replace Male and Female with 1 and 2:

```scala
val dataFinal = dataNewMetrics.map{s=>
    var sexual = 2
    if(s._1 equals "\"Male\""){
    sexual = 1
    }
    (sexual, s._2, s._3)
}
```

Generate train data into **LabeledPoint** format

```scala
val trainData = dataFinal.map{ s =>
    //Label Point, construct a matrix, first arg is label to be predicted,
    //second argument is a vector, argument type must be double
    LabeledPoint(s._3, Vectors.dense(s._1, s._2))
}.cache()
```

Train model using function **LinearRegressionWithSGD**, in **MLlib**, we need to tune parameters(such as learning rate, iteration times) ourselves, we tried different learning rate from 1 decreased to 0.0003, finnaly got a relative low train error 10.73:

```scala
val stepSize = 0.0003
val numIterations = 10000
val model = LinearRegressionWithSGD.train(trainData, numIterations, stepSize)
// Evaluate model on training examples and compute training error
val valuesAndPreds = trainData.map { s =>
  val prediction = model.predict(s.features)
  (s.label, prediction)
}
val MSE = valuesAndPreds.map{case(v, p) => math.pow((v - p), 2)}.mean()
val ModelError = math.sqrt(MSE)
println("Model Error = " + ModelError)
```

###Logistic Regression

As apache MLlib is under construction, they do not provide kernel svm, knn. So we'll implement some existed algorithms with apache spark, first one is Logistic Regression use [IRIS](https://en.wikipedia.org/wiki/Iris_flower_data_set) dataset from [UCI](https://archive.ics.uci.edu/ml/datasets/Iris) machine learning repostory. 

First of all, import data and remove the head line:

```scala
val csvPath = "/Users/wbcha/Desktop/Q1 Course/FP/MachineLearningSamples/extradata/iris.csv"
val rawData = sc.textFile(csvPath)
val dataWithoutHead = rawData.mapPartitionsWithIndex { (idx, iter) => if (idx == 0) iter.drop(1) else iter }
```

Split data by *comma* and generate new RDD.

```scala
val splitData = dataWithoutHead.map(s => (s.split(",")(0), s.split(",")(1), s.split(",")(2), s.split(",")(3), s.split(",")(4)))
```

Because Mllib only alow double format label, so we need to replace flower species into digits, in this case, Iris-setosa -> class 1, Iris-virginica -> class 2, Iris-versicolor -> class 3.

```scala
val dataSpecies = splitData.map{s =>
  var species = 0
  if(s._5 equals "Iris-setosa"){
    species = 0
  }else if(s._5 equals "Iris-virginica"){
    species = 1
  }else{
    species = 2
  }
  (s._1.toDouble, s._2.toDouble, s._3.toDouble, s._4.toDouble, species.toDouble)
}
```

Then we split data into train set and test set in order to validate our model:

```scala
val splits = dataSpecies.randomSplit(Array(0.6, 0.4), seed = 11L)
val training = splits(0)
val test = splits(1)
```

And transform them into *LabeledPoint* format:

```scala
val passedTrainData = training.map{s =>
  LabeledPoint(s._5, Vectors.dense(s._1, s._2, s._3, s._4))
}
val passedTestData = test.map{s =>
  LabeledPoint(s._5, Vectors.dense(s._1, s._2, s._3, s._4))
}
```

Next, we trained a logistic regression model, as we know that flowers can be grouped into 3 species, set *setNumClass* equals to 3.

```scala
val model = new LogisticRegressionWithLBFGS().setNumClasses(3).run(passedTrainData)
```

Predict label based on test data:

```scala
val predictionAndLabels = passedTestData.map { case LabeledPoint(label, features) =>
  val prediction = model.predict(features)
  (prediction, label)
}
```

And finally, evaluate precision rate:

```scala
val metrics = new MulticlassMetrics(predictionAndLabels)
val precision = metrics.precision
println("Precision = " + precision)
```

The precision rate equals to 92.98%. This is just an implementation of Logistic Regression with Spark, if we use cross validation set and do some feature engineering work, the precision rate is likely to be larger.

###Linear SVM

As we said before, Spark MLlib do not provide kernel SVM yet, so we continually use sexual-height-weight(OLS_Regression_Example_3.csv) dataset, try to use Linear SVM(Large-Margin Classifier) to seperate sexual according to different weight and height.

The reason why we use this dataset is that Linear-SVM on MLlib only support **Binary Classification** at the moment.

First of all, load data and remove header line

```scala
val csvPath = "/Users/wbcha/Downloads/MachineLearning-master/Example Data/OLS_Regression_Example_3.csv"
val rawData = sc.textFile(csvPath)
//remove the first line (csv head)
val dataWithoutHead = rawData.mapPartitionsWithIndex { (idx, iter) => if (idx == 0) iter.drop(1) else iter }
```

Generate new RDD by split data with comma, then replace Male with 0, replace Female with 1:

```scala
val splitData = dataWithoutHead.map(s => (s.split(",")(0), s.split(",")(1), s.split(",")(2)))
val dataSpecies = splitData.map{s =>
  var sexual = 0
  if(s._1 equals "\"Male\""){
    sexual = 0
  }else{
    sexual = 1
  }
  (s._2.toDouble, s._3.toDouble, sexual.toDouble)
}
```

Split data into train set and test set, transform then into **LabeledPoint** format:

```scala
//split data to train and test, training (60%) and test (40%).
val splits = dataSpecies.randomSplit(Array(0.6, 0.4), seed = 11L)
val training = splits(0)
val test = splits(1)
//prepare train and test data with labeled point format
val passedTrainData = training.map { s =>
  LabeledPoint(s._3, Vectors.dense(s._1, s._2))
}
val passedTestData = test.map{s =>
  LabeledPoint(s._3, Vectors.dense(s._1, s._2))
}
```

Train a linear-SVM model:

```scala
val numIterations = 200
val model = SVMWithSGD.train(passedTrainData, numIterations)
```

We use 200 as iteration times because we tried 100, 200 and 250 three numbers, and get accuracy rate: 71.96%, 91.68% and 91.60% respectively. So we selected the iteration times with the best performance.

Follow is the code for evaluate our model:

```scala
val metrics = new MulticlassMetrics(predictionAndLabels)
val precision = metrics.precision
println("Precision = " + precision)
```

###K-means

We continue implement existed algorithms with apache spark, this one is **K-means** use [IRIS](https://en.wikipedia.org/wiki/Iris_flower_data_set) dataset from [UCI](https://archive.ics.uci.edu/ml/datasets/Iris) machine learning repostory. 

As Kmeans is an unsupervised machine learning algorithm, we drop the label column of iris dataset to group dataset.

First of all, load data and remove header.

```scala
val csvPath = "/Users/wbcha/Desktop/Q1 Course/FP/MachineLearningSamples/extradata/iris.csv"
val rawData = sc.textFile(csvPath)
//remove the first line (csv head)
val dataWithoutHead = rawData.mapPartitionsWithIndex { (idx, iter) => if (idx == 0) iter.drop(1) else iter }
```

Split data with comma, remove the label from dataset to use kmeans:

```scala
val splitData = dataWithoutHead.map(s => (s.split(",")(0), s.split(",")(1), s.split(",")(2), s.split(",")(3)))
val parsedData = splitData.map(s => Vectors.dense(s._1.toDouble, s._2.toDouble, s._3.toDouble, s._4.toDouble)).cache()
```

Train a K-means model:

```scala
val numOfClusters = 3
val numIterations = 100
val clusters = KMeans.train(parsedData, numOfClusters, numIterations)
```

We set **NumOfClusters** equals to 3 because we already knows that there are 3 kinds of flowers in our dataset. And of course, we tried different numIterations, as we keep increase iteration time from 100, the model error remain the same.

Evaluate model error(Within Set Sum of Squared Errors):

```scala
val WSSSE = clusters.computeCost(parsedData)
println("Within Set Sum of Squared Errors = " + WSSSE)
```

Model error equals to 78.86.

###Naive Bayes

The Naive Bayes part has not been finished yet. So far, we have obtained the top features.
Get Documents from directory, get message from documents, as well as get and filter useful words from message

```scala
 def getDocuments(emailPath: String): RDD[String] =
    {
     val emailDocuments =  sc.wholeTextFiles(emailPath).filter(x => !x._1.toString.contains("cmd") && !x._1.toString.contains(".DS_Store"))
                                                       .map(x => x._2.mkString
                                                                     .substring(x._2.indexOf("\n\n"))
                                                                     .replace("\n", " ")
                                                                     .replace("\t", " ")
                                                                     .replace("3D", "")
                                                                     .replaceAll("[^a-zA-Z\\s]", "")
                                                                     .toLowerCase

                                                       )
     return emailDocuments
    }
```
Get stop words 
```scala
def getStopWords(stopWordsPath:String): List[String] =
    {
      val stopWordsSource = Source.fromFile(stopWordsPath, "latin1")
      val stopWordsLines = stopWordsSource.mkString.split("\n")
      return stopWordsLines.toList
    }
```
Get ham/spam top features
```scala
val hamDictionary = hamDocumentsTrain.flatMap(x => x.split(" ")).filter(s => s.nonEmpty &&!stopWords.contains(s))
val hamFeatures = hamDictionary.groupBy(w => w).mapValues(_.size).sortBy(_._2, ascending = false)
val hamTopFeatures = hamFeatures.map(x => x._1).take(featureAmount)
```

###Principal Component Analysis

Load data from csv and remove the first line:

```scala
val csvPath = "/Users/wbcha/Downloads/MachineLearning-master/Example Data/PCA_Example_1.csv"
val rawData = sc.textFile(csvPath)
//remove the first line (csv head)
val dataWithoutHead = rawData.mapPartitionsWithIndex { (idx, iter) => if (idx == 0) iter.drop(1) else iter }
```

Split data by comma, and group by date:

```scala
val dataRdd = dataWithoutHead.map(s => (s.split(",")(0), s.split(",")(1), s.split(",")(2).toDouble))
//group data by date
val groupByDate = dataRdd.groupBy(s => s._1).sortBy(s => s._1)
```

Generate 2 RDDs, *dateTuple* and *doubleTuple*, then combine them into a new RDD.

```scala
val dateTuples = groupByDate.map(s => (s._1))
val rawDoubleTuples = groupByDate.map(s => s._2)
val doubleTuples = rawDoubleTuples.map(s => s.map(t => t._3))
//combine dateTuples and doubleTuples
val finalRdd = dateTuples.zip(doubleTuples)
```

In order to compute Principal Components, we need to construct a *RowMatrix* in spark:

```scala
val rows = finalRdd.map(s => s._2).map{line =>
  val valuesString = line.mkString(",")
  val values = valuesString.split(",").map(_.toDouble)
  Vectors.dense(values)
}
```

The following code demonstrates how to compute principal components on a RowMatrix and use them to project the vectors into a low-dimensional space:

```scala
val matrix = new RowMatrix(rows)
val pc: Matrix = matrix.computePrincipalComponents(1)
println("Principal components are:\n" + pc)
```

We are able to get 24 principal components.

<h2 id='ML-Tricks-With-Spark'>Model Evaluation</h2>

###Confusion Matrix

A **Confusion Matrix** is a table that categorize predictions and it's real class. In Mllib, for multiclass classification, we can use this metrics to evaluate our model:

intact code can be seen in *evaluation.scala*, here, we use **logistic classifier** and **iris** dataset to predict the species of flowers.

```scala
val metrics = new MulticlassMetrics(predictionAndLabels)
```

###Precision and Recall

This measure is able to provide an indication of how interesting and relevant a model's result are.

**Precision** is the percentage of positive values that are truely positive, that is, precision equals the number of true positive divide the sum of true positive and false positive.

On the ohter hand, **Recall** indicates how complete the result are, that is the number of true positive over the sum of true positive and false negative.

```scala
val precision = metrics.precision
val recall = metrics.recall
```

###F-Score

As it seems to be a bit messy if we use both Precision and Recall, F-Score is a combination of these two metrics into a single number.

F-measure reduces model performance to a single number, it provides a convenient way to compare several models side-by-side.

```scala
val fscore = metrics.fMeasure
```

###ROC and AUC

The ROC curve (Receiver Operating Characteristic) is commonly used to examine the tradeoff between the detection of true positives while avoiding false positives. The closer the curve is to the perfect classifier, the area under the ROC Curve is larger (named AUC).

For binary classification, in MLlib, we can implement **ROC** like this:

```scala
val roc = metrics.roc
//area under ROC is AUC
val auROC = metrics.areaUnderROC
```

<h2 id='Scala-Functional'>Scala Functional</h2>

Scala is a full-blown functional language, and most of our code in Mllib is functional style. For example:

In Pca algorithm, in order to combine two data RDD, we applied zip function like this:

```scala
val finalRdd = dateTuples.zip(doubleTuples)
```

Function **zip** produces a new list by pairing successive elements
from two existing lists until either or both are exhausted.

We applied a bunch of **map** function during the data pre-processing task, for example:

```scala
val dataRdd = dataWithoutHead.map(s => (s.split(",")(0), s.split(",")(1), s.split(",")(2).toDouble))
```

Here, we splited the original RDD by comma and map the result into a new RDD for further process.







