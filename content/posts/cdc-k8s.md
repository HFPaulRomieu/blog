+++
title = 'Change Data Capture, Debezium and dangerous database cleanups'
date = 2024-07-14T12:34:55+01:00
draft = false
+++
## Introduction

I have been wondering for a while what my second blog post's topic would be about. Where would I get inspiration from?

And then it dawned on me...

I have an actual day job I can use for inspiration!
![captain obvious](/images/captain_obvious.gif)

I am going to write about the process of archiving data from a relational database in which I have enabled Change Data Capture through Debezium.

## Context

Lately I have been developing a full stack application which is powered by a relational database. One of the hard requirements
of the application was to visualise the business metrics on a dashboard **in real time**. 

If you ask any stakeholder how fresh they want their data they will 99.99% answer real-time fresh. Although this has become 
an actual meme in the real world, in my case I could not avoid it as the application itself is time critical and the fresher the data 
the more errors the end user would avoid.

But if I have one advice for other engineers, this would be to **always challenge a request for real-time data**
![real time meme](/images/real_time_meme.jpeg)

## The problem

Moving past the funny memes, I had to come up with an architecture that would provide both dashboards with real time data
and also somehow take the burden off from the application's operational database. We don't want to perform CRUD database operations
and at the same time calculate aggregations over the same tables. That would slow things down for both operations
and the dashboards users in the long run.

## The proposed solution

Although this problem can be probably (or partially) solved by using an RDS cluster (e.g. AWS aurora) that will have write and read replicas,
I preferred to use Kafka as a middleware instead


Enter Change Data Capture: this is a high level design of what I came up with
[![cdc](/images/cdc.png)](/images/cdc.png)

This is fairly straightforward: a debezium connector captures row-level changes in the database and produces kafka events
in a kafka topic. Then anyone who wants to visualise that data just has to consume them, or in this case here, use a source connector
like benthos, persist them in a database and then use the right database plugin for Grafana to execute the SQL queries of choice.

So far so good: users create data during the application's operations, we visualise them in Grafana and everyone is happy.

## Deleting data as part of the normal operations

But what happens if the application user deletes some data as part of normal operations? How will that be reflected in the visualisations?

To avoid showing deleted data in the dashboards, before the data land into a kafka, I wrote a small go program (kafka consumer and producer) that essentially 
checks debezium's payload and if the "after" field is null then I add to the kafka event a "isDeleted" boolean flag set to true.
Subsequently, in the SQL that powers the dashboard I exclude such events with 
```sql
SELECT ... FROM table WHERE isDeleted=false
```

Someone would perhaps wonder "and why didn't you apply slowly changing dimensions type 2 in your operational database".
I guess I want to be miserable! But I also wanted to handle this on the kafka layer, maybe as an experiment.

## Archiving data

In order to archive data in the long term, I have used a kafka source connector that sinks the kafka events to an object storage service.
Since this happens at every given moment, we don't have to worry about archiving the database data periodically.

But we still need to clean up the database from stale data. If we delete the data though, that small go program will suddenly flag 
everything as deleted, and then, at the analytical layer, an analyst could get confused, thinking that everything got deleted
during operations.

How are we going to solve this? We are just going to stop debezium, perform the archival and then resume it.

## Stopping debezium temporarily 

It turns out that there is not an SQL way to stop debezium (I wish there was something similar to a heartbeat table in which you would
insert a record with a boolean flag and that would stop WAL watching. Maybe a cool future debezium feature?). The only way 
we can achieve that is to pause the whole connector. 

Luckily in my case, the connector is deployed in kubernetes using strimzi. If there is one great thing about k8s resources
is that you can interact with them easily with REST API calls.

So the task is clear: schedule a cron job that will pause the connector, execute the archival and then resume the connector.

## K8s cron job

Although it is always tempting to create an airflow DAG for such things, I chose to schedule the job in kubernetes.
This is more performant in the sense that the whole application is already deployed in kubernetes so creating a cron job that would
access the same resources would require minimal effort. Also, in the end it would be faster because all the resources of interest along 
with the connector run in the same namespace.

Essentially we need to execute the following:

```bash
kubectl patch kafkaconnector {k8s_strimzi_resource} --type=merge -p='{"spec":{"pause": true}}'
```

```sql
DELETE FROM schema.table WHERE created_at < X
```

```bash
kubectl patch kafkaconnector {k8s_strimzi_resource} --type=merge -p='{"spec":{"pause": false}}'
```

Let's see how that code would look like in go...

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/util/retry"
	"log"
	"time"
)

const Namespace = "your_namespace"
const KafkaConnectorName = "your_kafka_connector_name"
const Group = "kafka.strimzi.io"
const Version = "your_version"
const Resource = "kafkaconnectors"

func PatchKafkaConnector(dynamicClient dynamic.Interface, gvr schema.GroupVersionResource, ctx context.Context, condition bool) error {
	patch := map[string]interface{}{
		"spec": map[string]interface{}{
			"pause": condition,
		},
	}

	patchBytes, err := json.Marshal(patch)
	if err != nil {
		return err
	}

	retryErr := retry.RetryOnConflict(retry.DefaultRetry, func() error {
		_, updateErr := dynamicClient.Resource(gvr).Namespace(Namespace).Patch(ctx, KafkaConnectorName, types.MergePatchType, patchBytes, v1.PatchOptions{})
		return updateErr
	})
	if retryErr != nil {
		return err
	}

	fmt.Printf("KafkaConnector %s patched successfully to %t \n", KafkaConnectorName, condition)

	return nil

}

func main() {
	ctx := context.Background()
	config, err := rest.InClusterConfig()
	if err != nil {
		log.Fatalf("Failed to create dynamic client: %v", err)
	}

	clientset, err := dynamic.NewForConfig(config)
	if err != nil {
		log.Fatalf("Failed to create dynamic client: %v", err)
	}

	gvr := schema.GroupVersionResource{
		Group:    Group,
		Version:  Version,
		Resource: Resource,
	}

	err = PatchKafkaConnector(clientset, gvr, ctx, true)
	if err != nil {
		log.Fatalf("Failed to patch the connector: %v", err)
	}

	//your SQL cleanup query here
	db.Models.Cleanup()

	err = PatchKafkaConnector(clientset, gvr, ctx, false)
	if err != nil {
		log.Fatalf("Failed to patch the connector: %v", err)
	}
}
```

## Testing

We need to make sure that the code above works as expected. You can always test the SQL query 
with testcontainers or through setting up and tearing down a db image with the appropriate tables, data etc, so I am
going to focus only on the k8s part.
Luckily, the k8s community provides excellent interfaces (at least in go) and faker packages that allow me to mock my k8s cluster setup:

```go
package tests

import (
	"context"
	"cmd/cron/cleanup"
	"internal/assert"
	"fmt"
	"k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/fake"
	"k8s.io/client-go/kubernetes/scheme"
	"testing"
)

func setupMockKafkaConnector() dynamic.Interface {
	s := scheme.Scheme

	gvr := schema.GroupVersionResource{
		Group:    "kafka.strimzi.io",
		Version:  "v1beta2",
		Resource: "kafkaconnectors",
	}

	dynamicClient := fake.NewSimpleDynamicClient(s)

	obj := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "kafka.strimzi.io/v1beta2",
			"kind":       "KafkaConnector",
			"metadata": map[string]interface{}{
				"name":      "your_kafka_connector",
				"namespace": "your_namespace",
			},
			"spec": map[string]interface{}{
				"class": "io.debezium.connector.postgresql.PostgresConnector",
				"pause": false,
			},
		},
	}

	_, err := dynamicClient.Resource(gvr).Namespace(cleanup.Namespace).Create(context.TODO(), obj, v1.CreateOptions{})
	if err != nil {
		fmt.Printf("Failed to create KafkaConnector: %v", err)
		return nil
	}

	return dynamicClient
}

func TestPatchKafkaConnector(t *testing.T) {

	dynamicClient := setupMockKafkaConnector()
	gvr := schema.GroupVersionResource{
		Group:    cleanup.Group,
		Version:  cleanup.Version,
		Resource: cleanup.Resource,
	}
	_ = cleanup.PatchKafkaConnector(dynamicClient, gvr, context.TODO(), true)
	obj, err := dynamicClient.Resource(gvr).Namespace(cleanup.Namespace).Get(context.TODO(), cleanup.KafkaConnectorName, v1.GetOptions{})
	if err != nil {
		t.Fatalf("Failed to get KafkaConnector: %v", err)
	}

	pause, found, err := unstructured.NestedBool(obj.Object, "spec", "pause")
	if err != nil || !found {
		t.Fatalf("Failed to get 'pause' field: %v", err)
	}

	assert.Equal(t, pause, true)
}
```

And that's it! Now we can safely run our code and cleanup the database.