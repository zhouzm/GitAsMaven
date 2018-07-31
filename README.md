

It is a fork of JeroenMols's [GitAsMaven](https://github.com/JeroenMols/GitAsMaven) to support Bitbucket Cloud OAuth 2.0 (Because the Bitbucket Cloud v1 API are deprecated).


# GitAsMaven
Gradle script to use Git as a private Maven repository.

Currently only BitBucket is supported, as the offer free private repositories. GitHub support easily be added in a similar script.


## Prerequisites
1. BitBucket repository with a `releases` as its main branch, as described in [this blogpost](http://jeroenmols.com/blog/2016/02/05/wagongit/).
2. Add an OAuth consumers with bitbucket settings and grant permissions
3. Update Gradle version to nightly version
  ```bash
  ./gradlew wrapper --gradle-version=4.10-20180730054456+0000
  ```
  
## Usage
Add following script to the top of the `build.gradle` file in your library and java project folder
  ```groovy
  task configured {
    project.ext.ACCESS_TOKEN = getAccessToken()
  }

  import groovy.json.JsonSlurper

  String getAccessToken() {
    def body = 'grant_type=client_credentials'
    def http = new URL('https://bitbucket.org/site/oauth2/access_token').openConnection() as HttpURLConnection
    def s = project.KEY + ':' + project.SECRET
    http.setRequestProperty ("Authorization", 'Basic ' + s.bytes.encodeBase64().toString());
    http.setRequestMethod("POST");
    http.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");
    http.setUseCaches(false);
    http.setDoInput(true);
    http.setDoOutput(true);
    http.outputStream.write(body.getBytes("UTF-8"))
    http.connect()

    def response = [:]

    if (http.responseCode == 200) {
      response = new JsonSlurper().parseText(http.inputStream.getText('UTF-8'))
      return response.access_token
    } else {
      response = new JsonSlurper().parseText(http.errorStream.getText('UTF-8'))
      println "response: ${response}"
      assert response.access_token != null
    }
  }
  ```
### uploadArchives

1. Add the following plugin to the top of the `build.gradle` file in your library folder

  ```groovy
  apply from: 'https://raw.githubusercontent.com/zhouzm/GitAsMaven/master/publish-bitbucket.gradle'
  ```

2. Create a `gradle.properties` file in the root of your project with the following parameters:

  ```groovy
  KEY=<bitbucket_consumer_key>
  SECRET=<bitbucket_consumer_secret>
  USERNAME=x-token-auth

  ARTIFACT_VERSION=<version_here>
  ARTIFACT_NAME=<libraryname_here>
  ARTIFACT_PACKAGE=<packagename_here>
  ARTIFACT_PACKAGING=jar //You could also use aar

  COMPANY=<bitbucket_team/company_here> //Simply your username if you're not part of a team
  REPOSITORY_NAME=<bitbucket_reponame_here>
  ```
  Note: Do not check this file into version control! 

3. Run the following command to upload a version to your Maven repository.

  ```bash
  ./gradlew uploadArchives
  ```
### Declaring Repositories
```groovy
repositories {
  jcenter()
  mavenCentral()
  maven {
    url 'https://api.bitbucket.org/2.0/repositories/'+COMPANY+'/maven_repository/src/releases'
    credentials(HttpHeaderCredentials) {
      name = "Authorization"
      value = 'Bearer ' + project.ACCESS_TOKEN
    }
    authentication {
      header(HttpHeaderAuthentication)
    }
  }
}

```
