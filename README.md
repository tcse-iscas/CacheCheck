# **CacheCheck**

CacheCheck can effectively detect cache-related bugs in Spark applications.
See paper [CacheCheck](http://www.tcse.cn/~wsdou/papers/2020-issta-cachecheck.pdf) to learn more details.

## **1. Run CacheCheck**
#### 1.1 Build CacheCheck
Enter the main directory, and build it by 
```bash
mvn package -DskipTests
``` 
A runnable jar file "core-1.0-SNAPSHOT.jar" is generated under `core/target/`.
#### 1.2 Instrument Spark
The trace collection code (i.e., the modification to Spark) is located in `instrument/`. Specially, the trace collection code starts with the comment "`// Start trace collection in CacheCheck`", and ends with the comment "`// End trace collection in CacheCheck`". 
First, you need to replace `$SPARK_HOME/core/src/main/scala/org/apache/spark/rdd/RDD.scala` with `cachecheck/core/instrument/RDD.scala` and `$SPARK_HOME/core/src/main/scala/org/apache/spark/SparkContext.scala` with `cachecheck/core/instrument/SparkContext.scala`.  

We use Spark-2.4.3 in our experiment. If you want to run on other versions, you'd better to manually add the instrumented code in the comment blocks, since directly replacing files may be incompatiable.

Then, you can build a Spark with the trace collection by running the command in `$SPARK_HOME/`: 
```bash
mvn package -DskipTests
```
#### 1.3 Collect Traces
While the application runs on Spark, the instrumented code can collect traces and store them in `$SPARK_HOME/trace/`.
In our experiment, we use Spark's build-in examples and six word count examples to drive Spark running. 
Taking SparkPi as the example, it can run by the command 
```bash
$SPARK_HOME/bin/run-example SparkPi
```
We provide six word count examples in directory `wordcount`. You can add the whole directory in `SPARK_HOME/examples/src/main/scala/org/apache/spark/examples`, and then compile example module, and run the similar command, such as
```bash
$SPARK_HOME/bin/run-example wordcount.MissingPersist
```  
In our paper, we mainly ran examples in GraphX, MLLib, and Spark SQL. They can also be run by similar command, such as 
```bash
$SPARK_HOME/bin/run-example graphx.ConnectedComponentsExample
```  
Considering there are too many examples to run, we provide some one-click tools for easy configuration and execution. See details in [Code Structure](#code-structure).  
#### 1.4 Perform Detection
The detection is performed by 
```bash
java -jar cachecheck/core/target/core-1.0-SNAPSHOT.jar $TraceDir $AppName [-d]
```
`$TraceDir` is the directory that stores traces, i.e., `$SPARK_HOME/trace/`. `$AppName` is the name of the application, which is usually set in the application code. It is also the file name of the intermediate files got in step 3. For SparkPi, its application name is `Spark Pi`. `-d` is an option about debug mode. In default, CacheCheck delete all intermediate files after detection. If you want keep them, add `-d` please.  
After the detection, a bug report, named `$AppName.report`, is generated  in `$SPARK_HOME/trace/`.
If you add `-d`, the bug report will be `$AppName.report`. It has more information for inspecting the bug.

## **2. Code Structure**
CacheCheck mainly has two modules, i.e., `core` and `tools`. `core` module realizes the algorithms and the approach introduced in our paper. `tools` module provides three tools, i.e., `ExampleRunner`, `CachecheckRunner`, and `Deduplicator` for easy and automatic detection. After [Build Cachecheck](#1-build-cachecheck), three runnable jars are generated under `cachecheck/tools/traget/`. They are `tools-examplerunner.jar`, `tools-cachecheckrunner.jar`, and `tools-deduplicator.jar`.  
#### 2.1 ExampleRunner
ExampleRunner can automatically run Spark's build-in examples. It requires a configuration file, which is a xml file just like `example-list-all.xml` in `cachecheck/tools/resource`. In this file, you can denote which examples to run.  
The execution command is 
```bash
java -jar cachecheck/tools/target/tools-examplerunner.jar $ExampleList $SparkDir
```
`$ExampleList` is the path of the configuration file. `$SparkDir` is the base directory of Spark, e.g., `$SPARK_HOME`.  
#### 2.2 CacheCheckRunner
CacheCheckRunner can automatically analyze all the traces under the same directory and get the bug reports.  
The execution command is 
```bash
java -jar cachecheck/tools/target/tools-cachecheckrunner.jar  $TraceDir
```
`$TraceDir` is the diretocry that traces locate in, e.g., `$SPARK_HOME/trace`.  
#### 2.3 Deduplicator
Deduplicator can collect all the bug reports under the same directory, make deduplication and generate a summary bug report. The command is 
```bash
java -jar cachecheck/tools/target/tools-deduplicator.jar  $ReportDir
``` 
`$ReportDir` is the directory storing bug reports. When finishing, Deduplicator will generate a `summary.report` file under the same directory.
***

## **3. Detected Unknown bugs**
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js">
    window.onload=function(){
        console.log("begin!");
        $.ajax({
            type:"get",
            url:"./statistics",
            data:{},
            async: false,
            success:function (data) {
                //console.log(data);
                var rows = data.split("\n");
                //console.log("data num:" + rows.length);
                for(var i=0;i < rows.length;i++){
                    var x=document.getElementById('data').insertRow();
                    var row = rows[i];
                    var values = row.split("\t");
                    if(values.length > 1) {
                        var cell=x.insertCell();
                        cell.innerHTML = i + 1;
                        for(var j=0;j < values.length;j++){
                            //console.log("value num:" + values.length);
                            var value = values[j];
                            cell=x.insertCell();
                            if(j == 1 && value.indexOf("http") > -1) {
                                //console.log("http index value:" + value.indexOf("http"));
                                    var link, issueName;
                                    if (value.indexOf(",") > 0) {
                                        link = value.split(",")[0];
                                        issueName = value.split(",")[1];
                                    } else {
                                        link = value;
                                        issueName = value.substr(value.lastIndexOf("/")+1);
                                    }
                                    //console.log("link:" + link + " issueName:"+ issueName);
                                    //cell.innerHTML = $("<a href=\"" + link + "\">" + issueName + "</a>").html();
                                    var htmlString = "<a href=\"" + link + "\">" + issueName + "</a>";
                                    console.log("htmlString:"+htmlString);
                                    cell.innerHTML = htmlString;
                            } else {
                                cell.innerHTML=values[j];}
                        }
                    }
                }
           }
        });
        console.log("end!");
    }
</script>
<table>
  <tbody><tr>
    <th>ID</th>
    <th>Project</th>
    <th>Issue ID</th>
    <th>Bug type</th>
    <th>Location</th>
    <th>Related RDD variable</th>
    <th>Status</th>
    <th>Fixed</th>
  </tr>
<tr><td>1</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29809">SPARK-29809</a></td><td>Missing persist</td><td>Word2Vec.fit()</td><td>dataset</td><td>Confirmed</td><td>Yes</td></tr><tr><td>2</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29810">SPARK-29810</a></td><td>Missing persist</td><td>RandomForest.run()</td><td>retaggedInput</td><td>Confirmed</td><td>Yes</td></tr><tr><td>3</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29811">SPARK-29811</a></td><td>Missing persist</td><td>RandomForestRegressor.train()</td><td>oldDataset</td><td>Confirmed</td><td>Yes</td></tr><tr><td>4</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29812">SPARK-29812</a></td><td>Missing persist</td><td>MulticlassificationEvaluator.evaluate()</td><td>predictionAndLabels</td><td>Confirmed</td><td>Yes</td></tr><tr><td>5</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29813">SPARK-29813</a></td><td>Missing persist</td><td>PrefixSpan.findFrequentItems()</td><td>data</td><td>Confirmed</td><td>Yes</td></tr><tr><td>6</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29814">SPARK-29814</a></td><td>Missing persist</td><td>PCA.fit()</td><td>sources</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>7</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29815">SPARK-29815</a></td><td>Missing persist</td><td>CrossValidator.fit()</td><td>dataset.toDF.rdd</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>8</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29817">SPARK-29817</a></td><td>Missing persist</td><td>LDAOptimizer.initialize()</td><td>docs</td><td>Confirmed</td><td>Yes</td></tr><tr><td>9</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29824">SPARK-29824</a></td><td>Missing persist</td><td>GBTClassifier.train()</td><td>trainDataset</td><td>Confirmed</td><td>Yes</td></tr><tr><td>10</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29826">SPARK-29826</a></td><td>Missing persist</td><td>ChiSqSelector.fit()</td><td>data</td><td>Confirmed</td><td>Yes</td></tr><tr><td>11</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29828">SPARK-29828</a></td><td>Missing persist</td><td>ALS.train()</td><td>ratings</td><td>Confirmed</td><td>Yes</td></tr><tr><td>12</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29816">SPARK-29816</a></td><td>Missing persist</td><td>BinaryClassificationMetrics.recallByThreshold()</td><td>scoreAndLabels.combineByKey</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>13</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29827">SPARK-29827</a></td><td>Missing persist</td><td>BisectingKMeans.run()</td><td>input</td><td>Confirmed</td><td>Yes</td></tr><tr><td>14</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29827">SPARK-29827</a></td><td>Unnecessary persist</td><td>BisectingKMeans.run()</td><td>norms</td><td>Confirmed</td><td>Yes</td></tr><tr><td>15</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29827">SPARK-29827</a></td><td>Missing persist</td><td>BisectingKMeans.run()</td><td>assignments</td><td>Confirmed</td><td>Yes</td></tr><tr><td>16</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29856">SPARK-29856</a></td><td>Unnecessary persist</td><td>RandomForest.run()</td><td>baggedInput</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>17</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29856">SPARK-29856</a></td><td>Unnecessary persist</td><td>KMeans.fit()</td><td>instances</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>18</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29856">SPARK-29856</a></td><td>Unnecessary persist</td><td>BisectingKMeans.run()</td><td>indices</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>19</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29844">SPARK-29844</a></td><td>Missing persist</td><td>ASL.train()</td><td>userIdAndFactors</td><td>Confirmed</td><td>No. Difficulty</td></tr><tr><td>20</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29844">SPARK-29844</a></td><td>Missing unpersist</td><td>ASL.train()</td><td>itemIdAndFactors</td><td>Confirmed</td><td>No. Difficulty</td></tr><tr><td>21</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29844">SPARK-29844</a></td><td>Premature unpersist</td><td>ASL.train()</td><td>itemFactors</td><td>Confirmed</td><td>Yes</td></tr><tr><td>22</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29844">SPARK-29844</a></td><td>Lagging unpersist</td><td>ASL.train()</td><td>userInBlocks</td><td>Confirmed</td><td>Yes</td></tr><tr><td>23</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29844">SPARK-29844</a></td><td>Lagging unpersist</td><td>ASL.train()</td><td>userOutBlocks</td><td>Confirmed</td><td>Yes</td></tr><tr><td>24</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29844">SPARK-29844</a></td><td>Lagging unpersist</td><td>ASL.train()</td><td>itemOutBlocks</td><td>Confirmed</td><td>Yes</td></tr><tr><td>25</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29844">SPARK-29844</a></td><td>Lagging unpersist</td><td>ASL.train()</td><td>BlockRatings</td><td>Confirmed</td><td>Yes</td></tr><tr><td>26</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29781">SPARK-29781</a></td><td>Unnecessary persist</td><td>PeriodicCheckpointer.update()</td><td>newData</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>27</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29781">SPARK-29781</a></td><td>Lagging unpersist</td><td>PeriodicCheckpointer.update()</td><td>newData</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>28</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29823">SPARK-29823</a></td><td>Unnecessary persist</td><td>KMeans.run()</td><td>norms</td><td>Confirmed</td><td>Yes</td></tr><tr><td>29</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29823">SPARK-29823</a></td><td>Missing persist</td><td>KMeans.run()</td><td>zippedData</td><td>Confirmed</td><td>Yes</td></tr><tr><td>30</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29832">SPARK-29832</a></td><td>Unnecessary persist</td><td>IsotonicRegression.fit()</td><td>instances</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>31</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29872">SPARK-29872</a></td><td>Missing persist</td><td>SparkTC.main()</td><td>edges</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>32</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29873">SPARK-29873</a></td><td>Unnecessary persist</td><td>SparkTC.main()</td><td>tc</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>33</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29874">SPARK-29874</a></td><td>Missing persist</td><td>LogisticRegressionSummary</td><td>fMeasure</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>34</td><td>SQL</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29875">SPARK-29875</a></td><td>Missing persist</td><td>SparkSQLExample.main()</td><td>peopleDF</td><td>Confirmed</td><td>No. Difficulty</td></tr><tr><td>35</td><td>SQL</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29876">SPARK-29876</a></td><td>Missing persist</td><td>RDDRelation.main()</td><td>createDataFrame</td><td>Confirmed</td><td>No. Difficulty</td></tr><tr><td>36</td><td>GraphX</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Unnecessary persist</td><td>GraphImpl.mapVertices()</td><td>vertices</td><td>Pending</td><td></td></tr><tr><td>37</td><td>GraphX</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Unnecessary persist</td><td>GraphImpl.mapVertices()</td><td>newEdges</td><td>Pending</td><td></td></tr><tr><td>38</td><td>GraphX</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Missing persist</td><td>ReplicatedVertexView</td><td>zipPartitions</td><td>Pending</td><td></td></tr><tr><td>39</td><td>GraphX</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Unnecessary persist</td><td>GraphImpl.partitionBy()</td><td>newEdges</td><td>Pending</td><td></td></tr><tr><td>40</td><td>GraphX</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Unnecessary persist</td><td>GraphImpl.subgraph()</td><td>vertices</td><td>Pending</td><td></td></tr><tr><td>41</td><td>GraphX</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Unnecessary persist</td><td>GraphImpl.aggregateMessagesWithActiveSet()</td><td>vertices</td><td>Pending</td><td></td></tr><tr><td>42</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Missing unpersist</td><td>GraphLoader.edgeListFile()</td><td>edges</td><td>Pending</td><td></td></tr><tr><td>43</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29878">SPARK-29878</a></td><td>Premature unpersist</td><td>FPGrowth.genericFit()</td><td>items</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>44</td><td>SQL</td><td>A custom case</td><td>Missing persist</td><td>CacheTableTest.main()</td><td>data</td><td>Confirmed</td><td>Yes</td></tr><tr><td>45</td><td>SQL</td><td><a href="http://issues.apache.org/jira/browse/SPARK-30444">SPARK-30444</a></td><td>Missing persist</td><td>SparkSQLExample</td><td>df</td><td>Pending</td><td></td></tr><tr><td>46</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31216">SPARK-31216</a></td><td>Lagging unpersist</td><td>GradientBoostedTrees.boost()</td><td>predErrorCheckpointer</td><td>Pending</td><td></td></tr><tr><td>47</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31216">SPARK-31216</a></td><td>Lagging unpersist</td><td>GradientBoostedTrees.boost()</td><td>input</td><td>Pending</td><td></td></tr><tr><td>48</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29872">SPARK-29872</a></td><td>Lagging unpersist</td><td>LogisticRegression.train()</td><td>instances</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>49</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29872">SPARK-29872</a></td><td>Lagging unpersist</td><td>CrossValidator.fit()</td><td>validationDataset</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>50</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29872">SPARK-29872</a></td><td>Lagging unpersist</td><td>TrainValidationSplit.fit()</td><td>trainingDataset</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>51</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29872">SPARK-29872</a></td><td>Lagging unpersist</td><td>OneVsRest.fit()</td><td>trainingDataset</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>52</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31217">SPARK-31217</a></td><td>Unnecessary persist</td><td>BinaryClassificationMetrics</td><td>cumulativeCounts</td><td>Pending</td><td></td></tr><tr><td>53</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29813">SPARK-29813</a></td><td>Unnecessary persist</td><td>PrefixSpan.run()</td><td>dataInternalRepr</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>54</td><td>MLLib</td><td>Fixed in 2.4.4 before submitted</td><td>Lagging persist</td><td>GradientBoostedTrees.boost()</td><td>input</td><td>Not submit</td><td>Yes</td></tr><tr><td>55</td><td>MLLib</td><td>Fixed in 2.4.4 before submitted</td><td>Missing persist</td><td>LDA.fit()</td><td>oldData</td><td>Not submit</td><td>Yes</td></tr><tr><td>56</td><td>MLLib</td><td>Fixed in 2.4.4 before submitted</td><td>Missing persist</td><td>LinearSVC.train()</td><td>instances</td><td>Not submit</td><td>Yes</td></tr><tr><td>57</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31218">SPARK-31218</a></td><td>Missing persist</td><td>BinaryClassificationMetrics.recallByThreshold()</td><td>counts</td><td>Pending</td><td></td></tr><tr><td>58</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31217">SPARK-31217</a></td><td>Missing persist</td><td>RegressionMetrics</td><td>summary</td><td>Pending</td><td></td></tr><tr><td>59</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31217">SPARK-31217</a></td><td>Missing persist</td><td>MulticlassMetrics</td><td>predictionAndLabels</td><td>Pending</td><td></td></tr><tr><td>60</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31217">SPARK-31217</a></td><td>Missing persist</td><td>MultilabelMetrics</td><td>predictionAndLabels</td><td>Pending</td><td></td></tr><tr><td>61</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-31217">SPARK-31217</a></td><td>Missing persist</td><td>RankingMetrics</td><td>predictionAndLabels</td><td>Pending</td><td></td></tr><tr><td>62</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29813">SPARK-29813</a></td><td>Missing persist</td><td>PrefixSpan.run()</td><td>data</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>63</td><td>MLLib</td><td><a href="http://issues.apache.org/jira/browse/SPARK-29813">SPARK-29813</a></td><td>Missing persist</td><td>PrefixSpan.genFreqPatterns()</td><td>data</td><td>Confirmed</td><td>No. Limited affect</td></tr><tr><td>64</td><td>MCL</td><td><a href="https://github.com/joandre/MCL_spark/issues/20">MCL_ISSUE-20</a></td><td>Missing persist</td><td>MCL.run()</td><td>graph.vertices.sortBy</td><td>Pending</td><td></td></tr><tr><td>65</td><td>MCL</td><td><a href="https://github.com/joandre/MCL_spark/issues/20">MCL_ISSUE-20</a></td><td>Missing persist</td><td>MCL.run()</td><td>M1</td><td>Pending</td><td></td></tr><tr><td>66</td><td>MCL</td><td><a href="https://github.com/joandre/MCL_spark/issues/20">MCL_ISSUE-20</a></td><td>Missing persist</td><td>MCL.run()</td><td>lookupTable</td><td>Pending</td><td></td></tr><tr><td>67</td><td>kBetweenness</td><td><a href="https://github.com/dmarcous/spark-betweenness/issues/6">KBETWEENNESS_ISSUE-6</a></td><td>Missing persist</td><td>kBetweenness.aggregateGraphletsBetweennessScores()</td><td>vertexKBcgraph</td><td>Pending</td><td></td></tr><tr><td>68</td><td>t-SNE</td><td><a href="https://github.com/saurfang/spark-tsne/issues/14">TSNE_ISSUE-14</a></td><td>Missing persist</td><td>MNIST.main()</td><td>data</td><td>Pending</td><td></td></tr><tr><td>69</td><td>t-SNE</td><td><a href="https://github.com/saurfang/spark-tsne/issues/14">TSNE_ISSUE-14</a></td><td>Missing persist</td><td>X2P.apply()</td><td>p_betas</td><td>Pending</td><td></td></tr><tr><td>70</td><td>t-SNE</td><td><a href="https://github.com/saurfang/spark-tsne/issues/14">TSNE_ISSUE-14</a></td><td>Unnecessary persist</td><td>X2P.apply()</td><td>norm</td><td>Pending</td><td></td></tr><tr><td>71</td><td>t-SNE</td><td><a href="https://github.com/saurfang/spark-tsne/issues/14">TSNE_ISSUE-14</a></td><td>Missing persist</td><td>SimpleTSNE.tsne()</td><td>P</td><td>Pending</td><td></td></tr><tr><td>72</td><td>t-SNE</td><td><a href="https://github.com/saurfang/spark-tsne/issues/14">TSNE_ISSUE-14</a></td><td>Missing persist</td><td>SimpleTSNE.tsne()</td><td>dataset</td><td>Pending</td><td></td></tr><tr></tr></tbody></table>