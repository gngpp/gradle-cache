# Intro

用于Docker环境下的`CI/CD`打包镜像，生成`Gradle依赖缓存层`，加快构建速度

- gradle.build

```java
task resolveDependencies {
    setDescription "Resolves all projects dependencies from the repository."
    setGroup "Build Server"

    doLast {
        rootProject.allprojects { project ->
            println ">> " + project
            Set<Configuration> configurations =
                    project.buildscript.configurations +
                            project.configurations
            configurations
                    .findAll { it.canBeResolved }
                    .forEach {
                        resolveDependencies(it)
                    }
        }
        copyDependencies()
    }
}

def resolveDependencies(Configuration it) {
    try {
        Set<File> files = it.resolve()
        DependencySet set = it.allDependencies
        if (set.size() > 0) {
            println ">>> " + it
            println ">>>> Dependencies "
            set.forEach {
                println ">>>>> " +
                        it.group + ":" +
                        it.name + ":" +
                        it.version
            }
            println ">>>> Files"
            files.forEach {
                println ">>>>> " + it.canonicalPath
            }
        }
    } catch (e) {
        println ">>> " + it
        println ">>>> " + e.message
    }
}

static def copyDependencies() {
    final String source =  System.getProperty("user.home") + "/.gradle/caches/modules-2/files-2.1"
    final String destRoot = System.getProperty("user.home") + "/.m2/repository"
    File rootFile = new File(source);
    for (File file : rootFile.listFiles()) {
        if (file.isDirectory() && file.getName()) {
            // Convert package name to file hierarchy
            String destDir = file.getName().replace(".", File.separator);
            copyDependenciesFiles(file, source + File.separator + file.getName(), destRoot + File.separator + destDir)
        }
    }
}

static void copyDependenciesFiles(File rootFile, String source, String destRoot) {
    for (File subFile : rootFile.listFiles()) {
        if (subFile.getName() != ".DS_Store") {
            if (subFile.isDirectory()) {
                copyDependenciesFiles(subFile, source, destRoot)
            } else {
                File md5Dir = new File(subFile.getParent())
                String copyPath = new File(md5Dir.getParent()).getAbsolutePath().replace(source, destRoot)
                File copyDir = new File(copyPath);
                if (!copyDir.exists()) {
                    copyDir.mkdirs()
                }
                File destFile = new File(copyDir, subFile.getName())
                // When copying, it is found that there may be multiple versions of the same package and the same version.
                // Print out here that there are multiple dependent packages but do not copy
                if (destFile.exists()) {
                    System.out.println("exists dependencies: " + subFile.getAbsolutePath());
                } else {
                    try {
                        System.out.println("source dirctrory: " + subFile.getAbsolutePath() + " >>>>> targer dirctrory: " + destFile.getAbsolutePath())
                        Files.copy(subFile.toPath(), destFile.toPath());
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

- 使用方法
```
./gradlew resolveDependencies --scan --info
```
