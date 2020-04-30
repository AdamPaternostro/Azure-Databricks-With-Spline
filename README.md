# Azure-Databricks-With-Spline
Using Spline with Azure Databricks.  See: https://absaoss.github.io/spline/


## Azure
1. Create a Linux VM Ubuntu 18 (enable ssh)
2. In the portal set the NSGs (allow inbound ports 8080, 8529, 9091)
  - Note: check you IP tables on the VM if oyu experience connectively issues remotely.  Also, check curl locally first.
3. NOTE: You need to replace my IP address of 52.156.70.35 with yours!

## Prep VM
1. SSH to the VM
2. Install docker: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04
3. Install java: sudo apt install openjdk-8-jre-headless

## Run Arango DB
1. Create another SSH session
2. Run
```
sudo docker run -p 8529:8529 -e ARANGO_ROOT_PASSWORD=openSesame arangodb/arangodb:3.6.3
```
3. On your local computer login into 
  - http://52.156.70.35:8529
  - username: root
  - password: openSesame

4. Initalize Spline database
```
wget https://repo1.maven.org/maven2/za/co/absa/spline/admin/0.5.0/admin-0.5.0.jar
java -jar admin-0.5.0.jar db-init arangodb://root:openSesame@52.156.70.35:8529/spline
```

## Run Spline (REST Server)
1. Create another SSH session
2. Run the container
```
sudo docker container run \
  -e spline.database.connectionUrl="arangodb://root:openSesame@52.156.70.35:8529/spline" \
  -p 9091:8080 \
  absaoss/spline-rest-server
```
- Test on Linux locally: ```curl http://localhost:9091/about/version```
   - Sample Result: {"build":{"timestamp":"2020-04-28T16:40:35Z","version":"0.5.0"}}
- Test from your machine: ```curl http://52.156.70.35:9091/about/version```


## Run Spline (User Interface)
1. Create another SSH session
2. Run the container
```
sudo docker container run \
      -e spline.consumer.url="http://52.156.70.35:9091/consumer" \
      -p 9090:8080 \
      absaoss/spline-web-client
```
3. Open the interface from your computer: http://52.156.70.35:9090/app/dashboard


## Create a Databricks 
1. Create a Databricks Workspace in Azure
2. Open the workspace
3. Create a cluster and get the Cluster id from the URL (e.g. 0430-135102-edits521) from url: #/setting/clusters/0430-135102-edits521)
4. Click on the user in the top right and generate an access token
3. Install the Databricks CLI (locally, ideally on Linux on Windows you need to download the wget manually)
5. Configure Databricks CLI
```
databricks configure --token
```
6. Download the Spline files
```
wget https://repo1.maven.org/maven2/za/co/absa/spline/spark-agent-bundle-2.4/0.4.2/spark-agent-bundle-2.4-0.4.2.jar
```
7. Upload and install on Databricks
```
databricks fs cp spark-agent-bundle-2.4-0.4.2.jar "dbfs:/lib/spline/spark-agent-bundle-2.4-0.4.2.jar" --overwrite
databricks libraries install --cluster-id YOUR_CLUSTER_ID_HERE --jar "dbfs:/lib/spline/spark-agent-bundle-2.4-0.4.2.jar"
```
7. Create a Notebook (Scala) with 3 cells
```
System.setProperty("spline.mode", "REQUIRED")
System.setProperty("spline.persistence.factory", "za.co.absa.spline.persistence.mongo.MongoPersistenceFactory")
System.setProperty("spline.mongodb.url","arangodb://root:openSesame@52.156.70.35:8529/spline")
System.setProperty("spline.producer.url","http://52.156.70.35:9091/producer")
System.setProperty("spline.mongodb.name", "spline")
```

```
import za.co.absa.spline.harvester.SparkLineageInitializer._
spark.enableLineageTracking()
```

```
%python
rawData = spark.read.option("inferSchema", "true").json("/databricks-datasets/structured-streaming/events/")
rawData.createOrReplaceTempView("rawData")
sql("select r1.action, count(*) as actionCount from rawData as r1 join rawData as r2 on r1.action = r2.action group by r1.action").write.mode('overwrite').csv("/tmp/pyaggaction.csv")
```


## View the Lineage in Spline
1. Open the Spline UI:  http://52.156.70.35:9090/app/dashboard
2. You should see something like this.

3. Click on it and you should see:


## References
- https://absaoss.github.io/spline/
- https://cloudarchitected.com/2019/04/data-lineage-in-azure-databricks-with-spline/
-  https://github.com/algattik/databricks-lineage-tutorial
- https://www.slideshare.net/SparkSummit/spline-apache-spark-lineage-not-only-for-the-banking-industry-with-marek-novotny-jan-scherbaum
- https://medium.com/@reenugrewal/data-lineage-tracking-using-spline-on-atlas-via-event-hub-6816be0fd5c7
- https://repo1.maven.org/maven2/za/co/absa/spline/spark-agent-bundle-2.4/0.4.2/
- https://forums.databricks.com/questions/14366/install-spline-data-lineage-tracking-and-visualiza.html
- https://databricks.com/session/spline-apache-spark-lineage-not-only-for-the-banking-industry
-# https://github.com/AbsaOSS/spline-spark-agent/blob/release/0.5.0/core/src/main/scala/za/co/absa/spline/harvester/SparkLineageInitializer.scala
- https://blogs.knowledgelens.com/index.php/2019/11/20/spark-data-lineage-on-databricks-notebook-using-spline/