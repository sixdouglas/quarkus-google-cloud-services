# Quarkiverse - Quarkus Google Cloud Services
<!-- ALL-CONTRIBUTORS-BADGE:START - Do not remove or modify this section -->
[![All Contributors](https://img.shields.io/badge/all_contributors-8-orange.svg?style=flat-square)](#contributors-)<!-- ALL-CONTRIBUTORS-BADGE:END -->
[![version](https://img.shields.io/maven-central/v/io.quarkiverse.googlecloudservices/quarkus-google-cloud-services-bom)](https://repo1.maven.org/maven2/io/quarkiverse/googlecloudservices/)
[![Build](https://github.com/quarkiverse/quarkus-google-cloud-services/workflows/Build/badge.svg)](https://github.com/quarkiverse/quarkus-google-cloud-services/actions?query=workflow%3ABuild)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

This repository hosts Quarkus extensions for different Google Cloud Services.

You can find the documentation in the [Google Cloud Services Quarkiverse documentation site](https://quarkiverse.github.io/quarkiverse-docs/quarkus-google-cloud-services/main/).

The following services are implemented:
- [BigQuery](bigquery)
- [Bigtable](bigtable)
- [Firestore](firestore)
- [PubSub](pubsub)
- [Secret Manager](secret-manager)
- [Spanner](spanner)
- [Storage](storage)

They all share an optional common configuration property to set the project ID:
```
quarkus.google.cloud.project-id=<your-project-id>
```

If the project ID is not set, the extensions will default to using `ServiceOptions.getDefaultProjectId()` 
that will use the default project detected via Application Default Credentials.

All these extensions work with applications built as native image executables.

These extensions work well within the various Google Cloud Functions extensions available inside Quarkus as they directly authenticate via the built-in credentials.

## Authenticating to Google Cloud

There are several ways to authenticate to Google Cloud, 
it depends on where your application runs (inside our outside Google Cloud Platform) and for which service.

The current authentication flow is as follows:
- Check the `quarkus.google.cloud.service-account-location` property, if it exists, use the service account file from this location.
- Check the `quarkus.google.cloud.service-account-encoded-key` property, if it exists, use the service account base64 encoded content.
- Check the access token returned as part of OpenId Connect Authorization Code Grant response after a user has authenticated with
  Google OpenId Connect provider (see [Quarkus OpenId Connect for Web Applications](https://quarkus.io/guides/security-openid-connect-web-authentication)).
  This access token can be used to access Google Services on behalf of the currently authenticated user
  but will be ignored if the `quarkus.google.cloud.accessTokenEnabled` property is set to `false`.
- Use `GoogleCredentials.getApplicationDefault()` that will search for credentials in multiple places:
    - Credentials file pointed to by the `GOOGLE_APPLICATION_CREDENTIALS` environment variable.
    - Credentials provided by the Google Cloud SDK `gcloud auth application-default login` command.
    - Google Cloud managed environment (Google App Engine, Google Cloud Functions, GCE, ...) built-in credentials.
    
**Google PubSub and Google Bigtable must be authenticated using the `GOOGLE_APPLICATION_CREDENTIALS` environment variable only.**

## Google Cloud services emulators: mocking Google Cloud credentials

If you plan to use one of the Google Cloud services emulators (for running on localhost, or for testing purpose), on a non-authenticated environment, 
you'll need to mock the Google Cloud authentication, and optionally the `CredentialsProvider` if you're using it (otherwise it will be removed by Quarkus CDI engine).

For testing, this can be done by creating a CDI producer that will produce a mocked bean (with Quarkus mock support and Mockito) 
to replace the `GoogleCloudCredentials` and the `CredentialsProvider`.

```java
import javax.enterprise.context.ApplicationScoped;
import javax.enterprise.inject.Default;
import javax.enterprise.inject.Produces;
import javax.inject.Singleton;

import org.mockito.Mockito;

import com.google.auth.oauth2.GoogleCredentials;
import com.google.api.gax.core.CredentialsProvider;
import com.google.api.gax.core.NoCredentialsProvider;

import io.quarkus.test.Mock;

@Mock
@ApplicationScoped
public class GoogleCredentialsMockProducer {

  @Produces
  @Singleton
  @Default
  public GoogleCredentials googleCredential() {
    return Mockito.mock(GoogleCredentials.class);
  }

  // only needed if you're injecting it inside one of your CDI beans
  @Produces
  @Singleton
  @Default
  public CredentialsProvider credentialsProvider() {
    return NoCredentialsProvider.create();
  }
}
```
    
## Example applications

Example applications can be found inside the integration-test folder:
- [main](integration-tests/main): RESTEasy endpoints using all the Google Cloud Services extensions, to be deployed as a standalone JAR.
- [google-cloud-functions](integration-tests/google-cloud-functions): A Google Cloud HTTP function using Google Cloud Storage. 
- [app-engine](integration-tests/app-engine): A RESTEasy endpoint using Google Cloud Storage, to be deployed inside Google App Engine.
    
## Contributing

Contributions are always welcome, but better create an issue to discuss them prior to any contributions.

## Contributors ✨

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tr>
    <td align="center"><a href="https://www.loicmathieu.fr"><img src="https://avatars2.githubusercontent.com/u/1819009?v=4?s=100" width="100px;" alt=""/><br /><sub><b>Loïc Mathieu</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=loicmathieu" title="Code">💻</a> <a href="#maintenance-loicmathieu" title="Maintenance">🚧</a></td>
    <td align="center"><a href="https://github.com/sberyozkin"><img src="https://avatars3.githubusercontent.com/u/467639?v=4?s=100" width="100px;" alt=""/><br /><sub><b>sberyozkin</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=sberyozkin" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/dzou"><img src="https://avatars1.githubusercontent.com/u/3209274?v=4?s=100" width="100px;" alt=""/><br /><sub><b>Daniel Zou</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=dzou" title="Code">💻</a></td>
    <td align="center"><a href="http://ynagai.info"><img src="https://avatars1.githubusercontent.com/u/1780156?v=4?s=100" width="100px;" alt=""/><br /><sub><b>Yuki Nagai</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=uny" title="Documentation">📖</a></td>
    <td align="center"><a href="http://madsopheim.com"><img src="https://avatars.githubusercontent.com/u/1844557?v=4?s=100" width="100px;" alt=""/><br /><sub><b>Mads Opheim</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=madsop" title="Code">💻</a> <a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=madsop" title="Documentation">📖</a></td>
    <td align="center"><a href="https://github.com/PeterUlb"><img src="https://avatars.githubusercontent.com/u/13261215?v=4?s=100" width="100px;" alt=""/><br /><sub><b>PeterUlb</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=PeterUlb" title="Code">💻</a></td>
    <td align="center"><a href="https://www.4pixel.it"><img src="https://avatars.githubusercontent.com/u/3707628?v=4?s=100" width="100px;" alt=""/><br /><sub><b>Felipe Sabadini</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=felipesabadini" title="Code">💻</a></td>
  </tr>
  <tr>
    <td align="center"><a href="https://twitter.com/ppalaga"><img src="https://avatars.githubusercontent.com/u/1826249?v=4?s=100" width="100px;" alt=""/><br /><sub><b>Peter Palaga</b></sub></a><br /><a href="https://github.com/quarkiverse/quarkus-google-cloud-services/commits?author=ppalaga" title="Code">💻</a></td>
  </tr>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind welcome!
