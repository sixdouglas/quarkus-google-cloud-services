# Quarkiverse - Google Cloud Services - Firestore

This extension allows to inject a `com.google.cloud.firestore.Firestore` object inside your Quarkus application.

## Bootstrapping the project

First, we need a new project. Create a new project with the following command (replace the version placeholders with the correct ones):

```shell script
mvn io.quarkus:quarkus-maven-plugin:<quarkusVersion>:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=firestore-quickstart \
    -Dextensions="resteasy-jackson,io.quarkiverse.googlecloudservices:quarkus-google-cloud-firestore:${googleCloudServicesVersion}"
cd firestore-quickstart
```

This command generates a Maven project, importing the Google Cloud Firestore extension.

If you already have your Quarkus project configured, you can add the `quarkus-google-cloud-firestore` extension to your project by running the following command in your project base directory:
```shell script
./mvnw quarkus:add-extension -Dextensions="io.quarkiverse.googlecloudservices:quarkus-google-cloud-firestore:${googleCloudServicesVersion}"
```

This will add the following to your pom.xml:

```xml
<dependency>
    <groupId>io.quarkiverse.googlecloudservices</groupId>
    <artifactId>quarkus-google-cloud-firestore</artifactId>
    <version>${googleCloudServicesVersion}</version>
</dependency>
```

## Some example

This is an example usage of the extension: we create a REST resource with a single endpoint that creates a 'persons' collection,
inserts three persons in it, then search for persons with last name Doe and returns them.

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.stream.Collectors;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import com.google.api.core.ApiFuture;
import com.google.api.core.ApiFutures;
import com.google.cloud.firestore.CollectionReference;
import com.google.cloud.firestore.Firestore;
import com.google.cloud.firestore.Query;
import com.google.cloud.firestore.QuerySnapshot;
import com.google.cloud.firestore.WriteResult;

@Path("/firestore")
public class FirestoreResource {
    @Inject
    Firestore firestore; // Inject Firestore

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String firestore() throws ExecutionException, InterruptedException {
        // Insert 3 persons
        CollectionReference persons = firestore.collection("persons");
        List<ApiFuture<WriteResult>> futures = new ArrayList<>();
        futures.add(persons.document("1").set(new Person(1L, "John", "Doe")));
        futures.add(persons.document("2").set(new Person(2L, "Jane", "Doe")));
        futures.add(persons.document("3").set(new Person(3L, "Charles", "Baudelaire")));
        ApiFutures.allAsList(futures).get();

        // Search for lastname=Doe
        Query query = persons.whereEqualTo("lastname", "Doe");
        ApiFuture<QuerySnapshot> querySnapshot = query.get();
        return querySnapshot.get().getDocuments().stream()
                .map(document -> document.getId() + " - " + document.getString("firstname") + " "
                        + document.getString("lastname") + "\n")
                .collect(Collectors.joining());
    }
}
```

NOTE: Here we let Firestore serialize the `Person` object, Firestore will use reflection for this.
So if you deploy your application as a GraalVM native image you will need to register the `Person` class for reflection.
This can be done by annotating it with `@RegisterForReflection`.