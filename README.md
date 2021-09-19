# Project Info

[![pipeline status](https://gitlab.com/bwgjoseph/km/badges/master/pipeline.svg)](https://gitlab.com/bwgjoseph/km/-/commits/master)
[![coverage report](https://gitlab.com/bwgjoseph/km/badges/master/coverage.svg)](https://gitlab.com/bwgjoseph/km/-/commits/master)

# Order

[Classes](https://docs.gradle.org/current/userguide/java_plugin.html#sec:java_tasks) is a `Java Plugin Lifecycle Task` which is an aggregate task that depends on `compileJava and processResources`. So this seem like a good place in inject `yarnBuild` task since almost all relevant java task depends on `classes` which means that whether we are calling `assemble, bootRun, or jib`, `yarnBuild` will always be called and run before compiling the server build. This would ensure that no matter in development, or in CI, the client production build will always be in the `server/src/main/resources/static` directory

Here's an listing to show the order of the task to run when calling `gradle assemble`

```log
1. :client:nodeSetup            (com.github.gradle.node.task.NodeSetupTask)
2. :client:yarnSetup            (com.github.gradle.node.yarn.task.YarnSetupTask)
3. :client:yarnInstall          (com.github.gradle.node.yarn.task.YarnTask)
4. :client:yarnBuild            (com.github.gradle.node.yarn.task.YarnTask)
5. :server:compileJava          (org.gradle.api.tasks.compile.JavaCompile)
6. :server:processResources     (org.gradle.language.jvm.tasks.ProcessResources)
7. :server:classes              (org.gradle.api.DefaultTask)
8. :server:bootJarMainClassName (org.springframework.boot.gradle.plugin.ResolveMainClassName)
9. :server:bootJar              (org.springframework.boot.gradle.tasks.bundling.BootJar)
10. :server:jar                 (org.gradle.api.tasks.bundling.Jar)
11. :server:assemble            (org.gradle.api.DefaultTask)
```

Calling `gradle bootRun`

```log
1. :client:nodeSetup            (com.github.gradle.node.task.NodeSetupTask)
2. :client:yarnSetup            (com.github.gradle.node.yarn.task.YarnSetupTask)
3. :client:yarnInstall          (com.github.gradle.node.yarn.task.YarnTask)
4. :client:yarnBuild            (com.github.gradle.node.yarn.task.YarnTask)
5. :server:compileJava          (org.gradle.api.tasks.compile.JavaCompile)
6. :server:processResources     (org.gradle.language.jvm.tasks.ProcessResources)
7. :server:classes              (org.gradle.api.DefaultTask)
8. :server:bootRunMainClassName (org.springframework.boot.gradle.plugin.ResolveMainClassName)
9. :server:bootRun              (org.springframework.boot.gradle.tasks.run.BootRun)
```

Calling `gradle jib`

```log
1. :client:nodeSetup        (com.github.gradle.node.task.NodeSetupTask)
2. :client:yarnSetup        (com.github.gradle.node.yarn.task.YarnSetupTask)
3. :client:yarnInstall      (com.github.gradle.node.yarn.task.YarnTask)
4. :client:yarnBuild        (com.github.gradle.node.yarn.task.YarnTask)
5. :server:compileJava      (org.gradle.api.tasks.compile.JavaCompile)
6. :server:processResources (org.gradle.language.jvm.tasks.ProcessResources)
7. :server:classes          (org.gradle.api.DefaultTask)
8. :server:jib              (com.google.cloud.tools.jib.gradle.BuildImageTask)
```

It is critical for `jib` to ensure that `yarnBuild` is run before server gets compiled and build as the client production build needs to be part of the image

As for copying the client production build into server, this is done at `package.json` as part of the `prebuild, and postbuild` process. As this is done as part of build process, this will be almost transparent and does not rely on setting up another `gradle task`