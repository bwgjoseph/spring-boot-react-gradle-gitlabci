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

~~As for copying the client production build into server, this is done at `package.json` as part of the `prebuild, and postbuild` process. As this is done as part of build process, this will be almost transparent and does not rely on setting up another `gradle task`~~

Copying of client production files are now using gradle task so that it's possible to make use of `input` and `output` task to determine if the task needs to run, and also because that using `prebuild` and `postbuild` script proven a little difficult in `CI` becaues cannot use `../server....` as the path

## Unresolved Issue

The linkage between `yarnCopyClientToServer` and `yarnBuild` is not done quite correctly.

If `App.tsx` is changed, then the following should have happened

```log
> Task :server:bootJarMainClassName
Excluding []
Excluding []
Excluding []
Caching disabled for task ':server:bootJarMainClassName' because:
  Build cache is disabled
Task ':server:bootJarMainClassName' is not up-to-date because:
  Input property 'classpath' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\asset-manifest.json has changed.
  Input property 'classpath' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\index.html has changed.
  Input property 'classpath' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\static\js\main.bbf2a6fa.chunk.js has been removed.
:server:bootJarMainClassName (Thread[Execution worker for ':' Thread 2,5,main]) completed. Took 0.16 secs.
:server:bootJar (Thread[Execution worker for ':',5,main]) started.

> Task :server:bootJar
Caching disabled for task ':server:bootJar' because:
  Build cache is disabled
Task ':server:bootJar' is not up-to-date because:
  Input property 'classpath' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\asset-manifest.json has changed.
  Input property 'classpath' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\index.html has changed.
  Input property 'classpath' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\static\js\main.bbf2a6fa.chunk.js has been removed.
:server:bootJar (Thread[Execution worker for ':',5,main]) completed. Took 0.436 secs.
:server:jar (Thread[Execution worker for ':',5,main]) started.

> Task :server:jar
Caching disabled for task ':server:jar' because:
  Build cache is disabled
Task ':server:jar' is not up-to-date because:
  Input property 'rootSpec$1' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\asset-manifest.json has changed.
  Input property 'rootSpec$1' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\index.html has changed.
  Input property 'rootSpec$1' file Z:\Development\workspace\gitlab\km\server\build\resources\main\static\static\js\main.bbf2a6fa.chunk.js has been removed.
:server:jar (Thread[Execution worker for ':',5,main]) completed. Took 0.067 secs.
:server:assemble (Thread[Execution worker for ':',5,main]) started.

> Task :server:assemble
Skipping task ':server:assemble' as it has no actions.
:server:assemble (Thread[Execution worker for ':',5,main]) completed. Took 0.0 secs.
```

but it doesn't, until the task `assemble` is ran the second time. On the first time, `yarnBuild` will detect NOT UP-TO-DATE and will rebuild the client project but does not seem to copy the latest build to server with `yarnCopyClientToServer` task

If the task is not linked correctly, it will also throw this warning/error

```log
> Task :server:processResources
Execution optimizations have been disabled for task ':server:processResources' to ensure correctness due to the following reasons:
  - Gradle detected a problem with the following location: 'Z:\Development\workspace\gitlab\km\server\src\main\resources'. Reason: Task ':server:processResources' uses this output of task ':client:yarnCopyClientToServer' without declaring an explicit or implicit dependency. This can lead to incorrect results being produced, depending on what order the tasks are executed. Please refer to https://docs.gradle.org/7.2/userguide/validation_problems.html#implicit_dependency for more details about this problem.
Invalidating VFS because task ':server:processResources' failed validation
Not watching anything anymore
Watching 0 directory hierarchies to track changes
Caching disabled for task ':server:processResources' because:
  Build cache is disabled
Task ':server:processResources' is not up-to-date because:
  Incremental execution has been disabled to ensure correctness. Please consult deprecation warnings for more details.
Watching 1 directory hierarchies to track changes
:server:processResources (Thread[Execution worker for ':',5,main]) completed. Took 0.297 secs.
:server:classes (Thread[Execution worker for ':',5,main]) started.
```

## Resources

- [Authoring Tasks](https://docs.gradle.org/current/userguide/more_about_tasks.html)
- [Making your Gradle tasks incremental](https://medium.com/@vanniktech/making-your-gradle-tasks-incremental-7f26e4ef09c3)
- [Integrating Java and npm Builds Using Gradle](https://dzone.com/articles/integrating-java-and-npm-builds-using-gradle)
- [Introducing Incremental Build Support](https://blog.gradle.org/introducing-incremental-build-support)