# Cirrus CI Android Templates

Templates for getting started with Cirrus CI for Android.

## Installing Cirrus CI

- Go to the Cirrus CI Application on [GitHub Marketplace](https://github.com/marketplace/cirrus-ci) and setup a plan for your account or organization.
- Add a `.cirrus.yml` file in your project's root directory.

Now you're ready to use the following templates as a starting point for writing your CI tasks.

## Assembing APKs and Running Checks

The following task assembles the APKs from the `app` module and runs all checks (unit tests and Android Lint).

```
assemble_and_check_task:
  timeout_in: 10m
  # trigger the task on commits on the master branch or any pull request
  only_if: $CIRRUS_BRANCH == "master" || CIRRUS_PR != ""
  env:
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false -Dkotlin.incremental=false -Dkotlin.compiler.execution.strategy=in-process
  container:
    image: reactivecircus/android-sdk:latest
    cpu: 8
    memory: 16G
  assemble_and_check_script:
    ./gradlew app:assemble check
```

## Running Instrumented Tests

The following task runs instrumented tests from all modules on API 21, 23, and 28.

The task also uses the `skip` keyword to skip the task unless there are changes to any Kotlin, XML or Gradle sources in the commit or pull request.

```
connected_check_task:
  # custom task name
  name: Run Android instrumented tests (API $API_LEVEL)
  # timeout in 30 minutes
  timeout_in: 30m
  only_if: $CIRRUS_BRANCH == "master" || CIRRUS_PR != ""
  # skip the task unless sources have changed
  skip: "!changesInclude('.cirrus.yml', '*.gradle', '*.gradle.kts', '**/*.gradle', '**/*.gradle.kts', '*.properties', '**/*.properties', '**/*.kt', '**/*.xml')"
  env:
    matrix:
      - API_LEVEL: 21
      - API_LEVEL: 23
      - API_LEVEL: 28
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false -Dkotlin.incremental=false -Dkotlin.compiler.execution.strategy=in-process
  container:
    image: reactivecircus/android-emulator-${API_LEVEL}:latest
    kvm: true
    cpu: 8
    memory: 24G
  create_device_script:
    echo no | avdmanager create avd --force --name "api-${API_LEVEL}" --abi "${TARGET}/${ARCH}" --package "system-images;android-${API_LEVEL};${TARGET};${ARCH}"
  start_emulator_background_script:
    $ANDROID_HOME/emulator/emulator -avd "api-${API_LEVEL}" -no-window -gpu swiftshader_indirect -no-snapshot -noaudio -no-boot-anim -camera-back none
  assemble_instrumented_tests_script:
    ./gradlew assembleDebugAndroidTest
  wait_for_emulator_script:
    adb wait-for-device shell 'while [[ -z $(getprop sys.boot_completed) ]]; do sleep 3; done; input keyevent 82'
  disable_animations_script: |
    adb shell settings put global window_animation_scale 0.0
    adb shell settings put global transition_animation_scale 0.0
    adb shell settings put global animator_duration_scale 0.0
  run_instrumented_tests_script:
    ./gradlew connectedDebugAndroidTest
```

## Publishing Library Snapshot Release

The following config builds, tests and publishes a snapshot release for a multi-module library project. There are 3 tasks:

- `assemble_and_check_task` - build the AAR and run all checks on each module
- `connected_check_task` - run instrumented tests from all modules on API 21, 23, and 28.
- `publish_artifacts_task` - after successful execution of both `assemble_and_check_task` and `connected_check_task`, deploy the snapshot build to Nexus Sonatype repository

Sonatype credentials can be [encrypted on Cirrus CI](https://cirrus-ci.org/guide/writing-tasks/#encrypted-variables) and safely added to the `.cirrus.yml` file.

```
assemble_and_check_task:
  only_if: $CIRRUS_BRANCH == "master" || CIRRUS_PR != ""
  env:
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false
  container:
    image: reactivecircus/android-sdk:latest
    cpu: 8
    memory: 16G
  assemble_and_check_script:
    ./gradlew assemble check

connected_check_task:
  only_if: $CIRRUS_BRANCH == "master" || CIRRUS_PR != ""
  env:
    matrix:
      - API_LEVEL: 21
      - API_LEVEL: 23
      - API_LEVEL: 28
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false
  container:
    image: reactivecircus/android-emulator-${API_LEVEL}:latest
    kvm: true
    cpu: 8
    memory: 24G
  create_device_script:
    avdmanager create avd --force --name "api-${API_LEVEL}" --abi "${TARGET}/${ARCH}" --package "system-images;android-${API_LEVEL};${TARGET};${ARCH}" --device "Nexus 6"
  start_emulator_background_script:
    $ANDROID_HOME/emulator/emulator -avd "api-${API_LEVEL}" -no-window -no-snapshot -noaudio -no-boot-anim -camera-back none
  assemble_instrumented_tests_script:
    ./gradlew assembleDebugAndroidTest
  wait_for_emulator_script:
    adb wait-for-device shell 'while [[ -z $(getprop sys.boot_completed) ]]; do sleep 3; done; input keyevent 82'
  disable_animations_script:
    - adb shell settings put global window_animation_scale 0.0
    - adb shell settings put global transition_animation_scale 0.0
    - adb shell settings put global animator_duration_scale 0.0
  run_instrumented_tests_script:
    ./gradlew connectedDebugAndroidTest

publish_artifacts_task:
  depends_on:
    - assemble_and_check
    - connected_check
  env:
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false
    SONATYPE_NEXUS_USERNAME: ENCRYPTED[xxx]
    SONATYPE_NEXUS_PASSWORD: ENCRYPTED[xxx]
  container:
    image: reactivecircus/android-sdk:latest
    cpu: 8
    memory: 16G
  deploy_snapshot_script:
    ./gradlew clean androidSourcesJar androidJavadocsJar uploadArchives --no-daemon --no-parallel
```

## Local Gradle Build Cache

The following task uses the [cache instruction](https://cirrus-ci.org/guide/writing-tasks/#cache-instruction) to cache the `~/.gradle/cache` directory:


```
assemble_and_check_task:
  only_if: $CIRRUS_BRANCH == "master" || CIRRUS_PR != ""
  env:
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false
  container:
    image: reactivecircus/android-sdk:latest
    cpu: 8
    memory: 16G
  gradle_cache:
    folder: ~/.gradle/caches
    # replace buildSrc/dependencies.gradle with whatever file you put your dependency versions in
    fingerprint_script: cat buildSrc/dependencies.gradle && cat gradle/wrapper/gradle-wrapper.properties
  assemble_and_check_script:
    ./gradlew assemble check
  # cleanup non-deterministic files to avoid unnecessary cache upload 
  cleanup_script:
    - rm -rf ~/.gradle/caches/[0-9].*
    - rm -rf ~/.gradle/caches/transforms-1
    - rm -rf ~/.gradle/caches/journal-1
    - rm -rf ~/.gradle/caches/jars-3/*/buildSrc.jar
    - find ~/.gradle/caches/ -name "*.lock" -type f -delete
```

## Remote Gradle Build Cache

Cirrus CI also supports Gradle's remote HTTP build cache.

Follow the [instructions](https://docs.gradle.org/current/userguide/build_cache.html#sec:build_cache_configure_remote) to setup remote build cache for your project.

No change to the `.cirrus.yml` file is required.

## Archiving Test Results

The following task stores the test results as **JUnit XML artifacts** which can be accessed from the build dashboard.

```
assemble_and_check_task:
  only_if: $CIRRUS_BRANCH == "master" || CIRRUS_PR != ""
  env:
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false
  container:
    image: reactivecircus/android-sdk:latest
    cpu: 8
    memory: 16G
  assemble_and_check_script:
    ./gradlew assemble check
  always:
    junit_artifacts:
      path: "**/test-results/**/*.xml"
      type: text/xml
      format: junit
```

## Delaying task execution - Required PR Labels

The following task will only be run after the required `initial-review` label has been added to the pull request.

```
connected_check_task:
  required_pr_labels: initial-review
  only_if: $CIRRUS_BRANCH == "master" || CIRRUS_PR != ""
  env:
    matrix:
      - API_LEVEL: 21
      - API_LEVEL: 23
      - API_LEVEL: 28
    JAVA_TOOL_OPTIONS: -Xmx6g
    GRADLE_OPTS: -Dorg.gradle.daemon=false -Dkotlin.incremental=false -Dkotlin.compiler.execution.strategy=in-process
  container:
    image: reactivecircus/android-emulator-${API_LEVEL}:latest
    kvm: true
    cpu: 8
    memory: 24G
  create_device_script:
    echo no | avdmanager create avd --force --name "api-${API_LEVEL}" --abi "${TARGET}/${ARCH}" --package "system-images;android-${API_LEVEL};${TARGET};${ARCH}"
  start_emulator_background_script:
    $ANDROID_HOME/emulator/emulator -avd "api-${API_LEVEL}" -no-window -gpu swiftshader_indirect -no-snapshot -noaudio -no-boot-anim -camera-back none
  assemble_instrumented_tests_script:
    ./gradlew assembleDebugAndroidTest
  wait_for_emulator_script:
    adb wait-for-device shell 'while [[ -z $(getprop sys.boot_completed) ]]; do sleep 3; done; input keyevent 82'
  disable_animations_script: |
    adb shell settings put global window_animation_scale 0.0
    adb shell settings put global transition_animation_scale 0.0
    adb shell settings put global animator_duration_scale 0.0
  run_instrumented_tests_script:
    ./gradlew connectedDebugAndroidTest
```
