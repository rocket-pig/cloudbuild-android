# What it does?

- It builds a Docker container with [Cloud Source Repositories](https://cloud.google.com/source-repositories) & [Cloud Build](https://cloud.google.com/source-repositories/docs/integrating-with-cloud-build).
- It then publishes the container image as `eu.gcr.io/$PROJECT_ID/cloudbuild-android` to the [Container Registry](https://console.cloud.google.com/gcr/images).
- It has OpenJDK8, Android `sdkmanager`, Gradle wrapper.
- It also has an Android application for testing purposes.
- It supports publishing to Bucket & Firebase App Distribution.
- It supports Cloud KMS encryption for the access credentials.

# How to use it?

 - Import to [Cloud Source Repositories](https://source.cloud.google.com/repo/new) and setup a build [trigger](https://console.cloud.google.com/cloud-build/triggers) there.

 ![Cloud Build - Screenshot 01](https://raw.githubusercontent.com/syslogic/cloudbuild-android-builder/master/screenshots/screenshot_01.png)
 - After having successfully built it, a new container should show up below `eu.gcr.io/$PROJECT_ID/cloudbuild-android`.
 - This container can be used <b>in another</b> Android project (or another Git branch) `cloudbuild.yaml`, in order not to build it every time.
 - When the `Dockerfile` runs `./gradlew build`, the API level & build-tools specified in the `build.gradle` get installed.
 - When the `Dockerfile` runs `./gradlew`, only the Gradle wrapper gets installed.

# Variable Substitutions

Meanwhile one can install packages with the `sdkmanager`, when passing variables.<br/>
At the moment these are statically set in [`cloudbuild.yaml`](https://github.com/syslogic/cloudbuild-android/blob/master/cloudbuild.yaml#L17), but the code is there.

 - `_ANDROID_NDK_VERSION` ~ `21.1.6352462`
 - `_ANDROID_SDK_PLATFORM` ~ `29`
 - `_ANDROID_SDK_VERSION` ~ `6200805` 
 - `_BUILD_TOOLS_VERSION` ~ `28.0.3`
 - `_GRADLE_VERSION` ~ `5.6.4`

# Usage examples

These examples assume that you already have the image in your project's private container registry.

Hostname `eu.gcr.io` (also bucket name `eu.artifacts`) can be replaced with `us.gcr.io` or `gcr.io`.

a) This uploads debug APK files with `gsutil` to `gs://eu.artifacts.$PROJECT_ID.appspot.com/android/`:

````
# cloudbuild.yaml

steps:

- name: eu.gcr.io/$PROJECT_ID/cloudbuild-android
  id: 'docker-pull'
  args: ['cp', '-a', '.', '/persistent_volume']
  volumes:
  - name: data
    path: /persistent_volume

- name: gcr.io/cloud-builders/docker
  id: 'gradle-build'
  volumes:
  - name: data
    path: /persistent_volume
  args: ['run', '-v', 'data:/workspace', '--rm', 'eu.gcr.io/$PROJECT_ID/cloudbuild', '/bin/sh', '-c', 'cd /workspace && ls -la && ./gradlew mobile:assembleDebug && mv mobile/build/outputs/apk/debug/mobile-debug.apk mobile/build/outputs/apk/debug/$REPO_NAME-$SHORT_SHA-debug.apk && ls -la mobile/build/outputs/apk/debug/$REPO_NAME-$SHORT_SHA-debug.apk']

- name: gcr.io/cloud-builders/gsutil
  id: 'publish-gsutil'
  args: ['cp', '/persistent_volume/mobile/build/outputs/apk/debug/$REPO_NAME-$SHORT_SHA-debug.apk', 'gs://eu.artifacts.$PROJECT_ID.appspot.com/android/']
  volumes:
  - name: data
    path: /persistent_volume

timeout: 1200s
````
b) Cloud KMS can be used decrypt files; this requires IAM `roles/cloudkms.cryptoKeyEncrypterDecrypter` for the service account:

 ![Cloud Build - Screenshot 02](https://github.com/syslogic/cloudbuild-android/raw/master/screenshots/screenshot_02.png)

The first step mounts volume `data`. The second step runs `gcloud kms decrypt` (there are scripts in the `/scripts` directory, for encrypting the [`*.enc`](https://github.com/syslogic/cloudbuild-android/tree/master/credentials) files). The Gradle task in the third step runs `mobile:assembleRelease mobile:appDistributionUploadRelease`, which uploads a signed release APK to Firebase App Distribution. This requires a separate service account with a `google-service-account.json`, because it is not possible to access the Cloud Build service account credentials.
````
# cloudbuild.yaml

steps:

- name: eu.gcr.io/$PROJECT_ID/cloudbuild-android
  id: 'docker-pull'
  args: ['cp', '-a', '.', '/persistent_volume']
  volumes:
  - name: data
    path: /persistent_volume

- name: gcr.io/cloud-builders/gcloud
  id: 'kms-decode'
  entrypoint: 'bash'
  waitFor: ['docker-pull']
  args:
    - '-c'
    - |
      mkdir -p /persistent_volume/.android
      gcloud kms decrypt --ciphertext-file=credentials/keystore.properties.enc --plaintext-file=/persistent_volume/keystore.properties --location=global --keyring=android-gradle --key=default
      gcloud kms decrypt --ciphertext-file=credentials/google-service-account.json.enc --plaintext-file=/persistent_volume/credentials/google-service-account.json --location=global --keyring=android-gradle --key=default
      gcloud kms decrypt --ciphertext-file=credentials/google-services.json.enc --plaintext-file=/persistent_volume/mobile/google-services.json --location=global --keyring=android-gradle --key=default
      gcloud kms decrypt --ciphertext-file=credentials/debug.keystore.enc --plaintext-file=/persistent_volume/.android/debug.keystore --location=global --keyring=android-gradle --key=default
      gcloud kms decrypt --ciphertext-file=credentials/release.keystore.enc --plaintext-file=/persistent_volume/.android/release.keystore --location=global --keyring=android-gradle --key=default
      rm -v ./credentials/*.enc
  volumes:
    - name: data
      path: /persistent_volume

- name: gcr.io/cloud-builders/docker
  id: 'firebase-distribution'
  waitFor: ['kms-decode']
  env: [
     'BUILD_NUMBER=$BUILD_ID'
  ]
  volumes:
    - name: data
      path: /persistent_volume
  args: ['run', '-v', 'data:/workspace', '--rm', 'eu.gcr.io/$PROJECT_ID/cloudbuild', '/bin/sh', '-c', 'cd /workspace && ls -la && ./gradlew mobile:assembleRelease mobile:appDistributionUploadRelease']

timeout: 1200s

````
The output:

````
Step #2 - "firebase-distribution": > Task :mobile:appDistributionUploadRelease
Step #2 - "firebase-distribution": Using APK path in the outputs directory: /workspace/mobile/build/outputs/apk/release/mobile-release.apk.
Step #2 - "firebase-distribution": Uploading APK to Firebase App Distribution...
Step #2 - "firebase-distribution": Getting appId from output of google services plugin
Step #2 - "firebase-distribution": Using service credentials file specified by the serviceCredentialsFile property in your app's build.gradle file: /workspace/credentials/google-service-account.json
Step #2 - "firebase-distribution": This APK has not been uploaded before.
Step #2 - "firebase-distribution": Uploading the APK.
Step #2 - "firebase-distribution": Uploaded APK successfully 202
Step #2 - "firebase-distribution": Added release notes successfully 200
Step #2 - "firebase-distribution": Added testers/groups successfully 200
Step #2 - "firebase-distribution": App Distribution upload finished successfully!
Step #2 - "firebase-distribution": 
Step #2 - "firebase-distribution": BUILD SUCCESSFUL in 1m 42s
Step #2 - "firebase-distribution": 28 actionable tasks: 28 executed
Finished Step #2 - firebase-distribution"
PUSH
DONE
````

# Contributions

Please notice the [`❤ Sponsor`](https://www.paypal.me/syslogic) button above.

 # Status
 
 ![Cloud Build Status](https://storage.googleapis.com/automatize-ci-badges/builds/cloud-build-badge/branches/master.svg)
 
# Also see
 - Blog [Simplify your CI process with GitHub and Google Cloud Build](https://github.blog/2018-07-26-simplify-your-ci-process/)
 - Marketplace [Google Cloud Build](https://github.com/marketplace/google-cloud-build) for GitHub integration.
 - [Google Cloud Build](https://github.com/GoogleCloudBuild) (official).
 
