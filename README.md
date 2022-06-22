# CICDPractice With Gitlab

In this project I tried to create a pipeline for a sample project where you build and test your CI/CD using Gitlab.

## Start with creating CI/CD with android

open your project build.gradle file and set the classpath dependency in buildscript.

Reference: https://github.com/Triple-T/gradle-play-publisher

buildscript {
    dependencies {
        classpath 'com.github.triplet.gradle:play-publisher:1.1.5'
    }
}
now open app/build.gradle file and add plugin.

plugins {
    id 'com.github.triplet.play'
}

now set signingConfigs in app/build.gradle

android {
......
    defaultConfig {
        .....
    }

    signingConfigs {
        release {
   
            storeFile file("release-keystore.jks") //keystor
            storePassword "password"
            keyAlias "alias"
            keyPassword "password"
        }
    }
...........
}
Note: keystore file in the same directory as the app/build.gradle file.

now add signingConfigs.release in buildType under release.

buildTypes {
    release {
        signingConfig signingConfigs.release
         ........
    }
}
Now your Android Studio Project is ready for “Gradle Play Publisher plugin” .Now we need to set Google Play Console Authentication.

First we need to setup Google Service Account.

now open Google Play Console, https://play.google.com/console and create account (if you don’t have account).

then select the Setup tab > API access


click on Create new service account.


then a pop-up will appear.

now click on Google Cloud Platform.


This will open a new tab in your browser.

then click on CREATE SETVICE ACCOUNT


then in next screen in service account details section setup service account name and description(optional),

then in Grant this service account access to project section select Role: Owner then click continue, then next step is optional .

now click on Done.


now after this we back to previous page and we see there is new service account is created.

now click on menu of new service account then click on Manage keys option


then in next screen click on ADD KEY dropdown then click Create new key.

Click on create new Key
Click on Create new key
after that a popup will appear, select key type> JSON than click on CREATE


After that you see a json file of your key will be download in your pc, then save this json file for future use.

then back to your Google Play Console where you left popup screen,

Click on Done for creating new service account.


After that you will see there is new service account will be create in Service account secretion.

then click on Grant permissions and give the permission to user.


After that the new screen will appear where you can see user details and permissions.

you can check Set access expiry date if you want (optional).

in Account permissions section check Admin(all permission)(optional).

then at the end click on Invite user.




then click on Sent Invitation button in pop up,

Now you can see you service account is appear in users section,

Step 4: Setup Google play api with you project
New if you remember that we save our key json file in some kind of place,

now copy this json file and paste it in your project>app directory.

After that go to your Android Studio then open app/build.gradle file and add this code,

android {
.....
}
play {
    serviceAccountCredentials = file('pc-api-5901144735381435704-900-e2a645b4efce.json') // Json file in you app directory
    track = 'alpha' // set track for playstore  like 'production','beta','alpha'
}

dependencies {
.....
}
after that click on Sync Now and sync the project .

Reference: https://docs.gitlab.com/ee/ci/yaml/

First we have to create .gitlab-ci.yml inside you project directory,

.gitlab-ci.yml is the basic config file to ensure your Android app compiles and passes unit and functional tests.

you can find .gitlab-ci.yml latest template from here.

https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Android.latest.gitlab-ci.yml

here is my .gitlab-ci.yml

image: openjdk:11-jdk

variables:

  # ANDROID_COMPILE_SDK is the version of Android you're compiling with.
  # It should match compileSdkVersion.
  ANDROID_COMPILE_SDK: "32"

  # ANDROID_BUILD_TOOLS is the version of the Android build tools you are using.
  # It should match buildToolsVersion.
  ANDROID_BUILD_TOOLS: "32.1.0-rc1"

  # It's what version of the command line tools we're going to download from the official site.
  # Official Site-> https://developer.android.com/studio/index.html
  # There, look down below at the cli tools only, sdk tools package is of format:
  #        commandlinetools-os_type-ANDROID_SDK_TOOLS_latest.zip
  # when the script was last modified for latest compileSdkVersion, it was which is written down below
  ANDROID_SDK_TOOLS: "7583922"

# Packages installation before running script
before_script:
  - apt-get --quiet update --yes
  - apt-get --quiet install --yes wget tar unzip lib32stdc++6 lib32z1

  # Setup path as ANDROID_SDK_ROOT for moving/exporting the downloaded sdk into it
  - export ANDROID_SDK_ROOT="${PWD}/android-home"
  # Create a new directory at specified location
  - install -d $ANDROID_SDK_ROOT
  # Here we are installing androidSDK tools from official source,
  # (the key thing here is the url from where you are downloading these sdk tool for command line, so please do note this url pattern there and here as well)
  # after that unzipping those tools and
  # then running a series of SDK manager commands to install necessary android SDK packages that'll allow the app to build
  - wget --output-document=$ANDROID_SDK_ROOT/cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-${ANDROID_SDK_TOOLS}_latest.zip
  # move to the archive at ANDROID_SDK_ROOT
  - pushd $ANDROID_SDK_ROOT
  - unzip -d cmdline-tools cmdline-tools.zip
  - pushd cmdline-tools
  # since commandline tools version 7583922 the root folder is named "cmdline-tools" so we rename it if necessary
  - mv cmdline-tools tools || true
  - popd
  - popd
  - export PATH=$PATH:${ANDROID_SDK_ROOT}/cmdline-tools/tools/bin/

  # Nothing fancy here, just checking sdkManager version
  - sdkmanager --version

  # use yes to accept all licenses
  - yes | sdkmanager --licenses || true
  - sdkmanager "platforms;android-${ANDROID_COMPILE_SDK}"
  - sdkmanager "platform-tools"
  - sdkmanager "build-tools;${ANDROID_BUILD_TOOLS}"

  # Not necessary, but just for surity
  - chmod +x ./gradlew

stages:
  - build
  - test

build:
  stage: build
  script:
    - ./gradlew assembleDebug # command to build and debug
  only:
    - main #set trigger for CICD if push or merge in master branch
  artifacts:
    paths:
      - ./app/build/outputs/ # set artifact path which store your APK file

unitTests:
  stage: test
  script:
    - ./gradlew test
you can use this file for testing (optional).

To understand the important part of this .gitbal-ci.yml file that dynamically creates files from the CI/CD Variable, you must read reference mention below.

Reference: https://about.gitlab.com/blog/2018/10/24/setting-up-gitlab-ci-for-android-projects/

As you see I have define two stages in my .gitbal-ci.yml build and test stage,

For in our build stage we simply build and debug our app, after build stage is passed then our test stage is going to be running and then after passing test stage we can simply check out apk.

Lets implement this,

First configure project to GitLab repository and then commit changes into your master branch(mention in yml file) that you made in your project and push into master branch in GitLab repository.

After project is successfully pushed to GitLab than go to Gitlab and click on project>CI/CD>Pipelines


in next screen if you see this type of issue you need to Validate your account or create your own runner and use with this project.


I personally go with validate my account because its simple, after validation you just need to unable your shared runner,

simply go to Gitlab Project>Settings>CI/CD>Runners then Expand and enable shared runner,


otherwise you can create your own runner its simple I mention the link blow

Install GitLab Runner | GitLab

If you not see this issue then you will see your pipeline is running,

if you remember we have define two stages in .gitbal-ci.yml build and test

this is your build stage is running


after completion of build stage then test stage is going to be running

now test stage is running


now after completion and pass both build and test stages and branch can be Merged then you are finally ready and this is your success.


Photo by krakenimages on Unsplash
Here we can download the apk file from the Artifacts.

click on menu>build:archive


Now finally CI/CD is complete.👏
