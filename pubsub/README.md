# Quarkiverse - Google Cloud Services - PubSub

This extension allows using Google Cloud PubSub inside your Quarkus application.

## Bootstrapping the project

First, we need a new project. Create a new project with the following command (replace the version placeholders with the correct ones):

```shell script
mvn io.quarkus:quarkus-maven-plugin:<quarkusVersion>:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=pubsub-quickstart \
    -Dextensions="resteasy-jackson,io.quarkiverse.googlecloudservices:quarkus-google-cloud-pubsub:${googleCloudServicesVersion}"
cd pubsub-quickstart
```

This command generates a Maven project, importing the Google Cloud PubSub extension.

If you already have your Quarkus project configured, you can add the `quarkus-google-cloud-pubsub` extension to your project by running the following command in your project base directory:
```shell script
./mvnw quarkus:add-extension -Dextensions="io.quarkiverse.googlecloudservices:quarkus-google-cloud-pubsub:${googleCloudServicesVersion}"
```

This will add the following to your pom.xml:

```xml
<dependency>
    <groupId>io.quarkiverse.googlecloudservices</groupId>
    <artifactId>quarkus-google-cloud-pubsub</artifactId>
    <version>${googleCloudServicesVersion}</version>
</dependency>
```

## Preparatory steps

To test PubSub you first need to create a topic named `test-topic`

You can create one with `gcloud`:

```
gcloud pubsub topics create test-topic
```

## Authentication

By default, PubSub mandates the usage of the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to define its credentials, so
you need to set this one instead of relying on the `quarkus.google.cloud.service-account-location` property. 

```
export GOOGLE_APPLICATION_CREDENTIALS=<your-service-account-file>
```

Another solution, is to inject a `CredentialsProvider` provided by the extension, and to use it inside the various PubSub 
builders and settings objects, when instantiating PubSub components. This can be seen on the example that follows.

## Some example

This is an example usage of the extension: we create a REST resource with a single endpoint that sends a message to the `test-topic` topic when hit.

We also register a consumer to the same topic at `@PostConstruct` time that logs all received messages on the topic so we can check that it works.

```java
import java.io.IOException;
import java.util.Optional;
import java.util.concurrent.TimeUnit;
import java.util.stream.StreamSupport;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import com.google.api.core.ApiFuture;
import com.google.api.core.ApiFutureCallback;
import com.google.api.core.ApiFutures;
import com.google.api.gax.core.CredentialsProvider;
import com.google.cloud.pubsub.v1.MessageReceiver;
import com.google.cloud.pubsub.v1.Publisher;
import com.google.cloud.pubsub.v1.Subscriber;
import com.google.cloud.pubsub.v1.SubscriptionAdminClient;
import com.google.cloud.pubsub.v1.SubscriptionAdminSettings;
import com.google.common.util.concurrent.MoreExecutors;
import com.google.protobuf.ByteString;
import com.google.pubsub.v1.ProjectName;
import com.google.pubsub.v1.ProjectSubscriptionName;
import com.google.pubsub.v1.PubsubMessage;
import com.google.pubsub.v1.PushConfig;
import com.google.pubsub.v1.Subscription;
import com.google.pubsub.v1.Topic;
import com.google.pubsub.v1.TopicName;

@Path("/pubsub")
public class PubSubResource {
    private static final Logger LOG = Logger.getLogger(PubSubResource.class);

    @ConfigProperty(name = "quarkus.google.cloud.project-id")
    String projectId;// Inject the projectId property from application.properties

    private TopicName topicName;
    private Subscriber subscriber;

    @Inject
    CredentialsProvider credentialsProvider;

    @PostConstruct
    void init() throws IOException {
        // Init topic and subscription, the topic must have been created before
        topicName = TopicName.of(projectId, "test-topic");
        ProjectSubscriptionName subscriptionName = initSubscription();

        // Subscribe to PubSub
        MessageReceiver receiver = (message, consumer) -> {
            LOG.infov("Got message {0}", message.getData().toStringUtf8());
            consumer.ack();
        };
        subscriber = Subscriber.newBuilder(subscriptionName, receiver).build();
        ;
        subscriber.startAsync().awaitRunning();
    }

    @PreDestroy
    void destroy() {
        // Stop the subscription at destroy time
        if (subscriber != null) {
            subscriber.stopAsync();
        }
    }

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public void pubsub() throws IOException, InterruptedException {
        // Init a publisher to the topic
        Publisher publisher = Publisher.newBuilder(topicName)
                .setCredentialsProvider(credentialsProvider)
                .build();
        try {
            ByteString data = ByteString.copyFromUtf8("my-message");// Create a new message
            PubsubMessage pubsubMessage = PubsubMessage.newBuilder().setData(data).build();
            ApiFuture<String> messageIdFuture = publisher.publish(pubsubMessage);// Publish the message
            ApiFutures.addCallback(messageIdFuture, new ApiFutureCallback<String>() {// Wait for message submission and log the result
                public void onSuccess(String messageId) {
                    LOG.infov("published with message id {0}", messageId);
                }

                public void onFailure(Throwable t) {
                    LOG.warnv("failed to publish: {0}", t);
                }
            }, MoreExecutors.directExecutor());
        } finally {
            publisher.shutdown();
            publisher.awaitTermination(1, TimeUnit.MINUTES);
        }
    }

    private ProjectSubscriptionName initSubscription() throws IOException {
        // List all existing subscriptions and create the 'test-subscription' if needed
        ProjectSubscriptionName subscriptionName = ProjectSubscriptionName.of(projectId, "test-subscription");
        SubscriptionAdminSettings subscriptionAdminSettings = SubscriptionAdminSettings.newBuilder()
                .setCredentialsProvider(credentialsProvider)
                .build();
        try (SubscriptionAdminClient subscriptionAdminClient = SubscriptionAdminClient.create(subscriptionAdminSettings)) {
            Iterable<Subscription> subscriptions = subscriptionAdminClient.listSubscriptions(ProjectName.of(projectId))
                    .iterateAll();
            Optional<Subscription> existing = StreamSupport.stream(subscriptions.spliterator(), false)
                    .filter(sub -> sub.getName().equals(subscriptionName.toString()))
                    .findFirst();
            if (!existing.isPresent()) {
                subscriptionAdminClient.createSubscription(subscriptionName, topicName, PushConfig.getDefaultInstance(), 0);
            }
        }
        return subscriptionName;
    }
}
```
