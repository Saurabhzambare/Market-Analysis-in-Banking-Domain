--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
====================
(0)Import necessary libraries
====================
import org.apache.spark._
import org.apache.spark.rdd.RDD
import org.apache.spark.util.IntParam
import org.apache.spark.sql.functions._
import org.apache.spark.sql._
import org.apache.spark.mllib.stat.Statistics
import org.apache.spark.sql.SparkSession
val spark = SparkSession.builder().appName("Spark SQL basic example").config("spark.some.config.option", "some-value").getOrCreate()
import spark.implicits._
import spark._
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
==============================
(1) Load data and create a Spark data frame
==============================
val df = sqlContext.read.format("com.databricks.spark.csv").option("header","true").option("inferSchema","true").option("delimiter",",").load("/user/zambaresaurabh10hotmail/Market_Analysis")
df.show()
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
======================================================
(2) Give marketing success rate (No. of people subscribed / total no. of entries)   
	- Give marketing failure rate
======================================================
val suc = df.filter($"y" === "yes").count.toFloat / df.count.toFloat *100
val fail = df.filter($"y" === "no").count.toFloat / df.count.toFloat *100
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
========================================================
(3) Give the maximum, mean, and minimum age of the average targeted customer
========================================================
import org.apache.spark.sql.functions.{min, max, avg}
df.agg(max($"age"),min($"age"), avg($"age")).show()
df.registerTempTable("bank")
df.select(avg($"balance")).show()
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
=================================================================
(4) Check the quality of customers by checking average balance, median balance of customers
=================================================================
val median = sqlContext.sql("SELECT percentile_approx(balance, 0.5) FROM bank").show()
val age = sqlContext.sql("select age, count(*) as number from bank where y='yes' group by age order by number desc ").show()
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
==========================================
(5) Check if age matters in marketing subscription for deposit
==========================================
val marital = sqlContext.sql("select marital, count(*) as number from bank where y='yes' group by marital order by number desc ").show()
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
=============================================================
(6) Check if age and marital status together mattered for a subscription to deposit scheme
=============================================================
val age_marital = sqlContext.sql("select age, marital, count(*) as number from bank where y='yes' group by age,marital order by number desc ").show()
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
==========================================================
(7) Do feature engineering for the bank and find the right age effect on the campaign
==========================================================
import scala.reflect.runtime.universe
import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import org.apache.spark.sql.DataFrame
import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.functions.mean
import org.apache.spark.ml.feature.StringIndexer
val ageRDD = sqlContext.udf.register("ageRDD",(age:Int) => {if (age < 20) "Teen" else if (age > 20 && age <= 32) "Young" else if (age > 33 && age <= 55) "Middle Aged" else "Old"})
val banknewDF = df.withColumn("age",ageRDD(df("age")))
banknewDF.registerTempTable("bank_new")
val age_target = sqlContext.sql("select age, count(*) as number from bank_new where y='yes' group by age order by number desc ").show()
val ageInd = new StringIndexer().setInputCol("age").setOutputCol("ageIndex")
var strIndModel = ageInd.fit(banknewDF)
strIndModel.transform(banknewDF).select("age","ageIndex").show(5)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------