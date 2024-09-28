+++
title = 'Aggregating JaCoCo Coverage Reports (Gradle)'
categories = ['gradle']
date = 2024-09-26T23:55:52-07:00
draft = false
tags = ['gradle', 'jacoco']
+++
In a multi-module gradle projects, it may be necessary to aggregate coverage reports across
modules since the tests and the classes being covered by these tests may not be in same module.

Gradle introduced the [JaCoCo Report Aggregation Plugin](https://docs.gradle.org/current/userguide/jacoco_report_aggregation_plugin.html)
to aggregate coverage report. However, it requires creating a standalone project just to include
collect all test reports.

Instead, the coverage collection can be done in root build.gradle like this

```groovy
plugins {
    id 'base'
    id 'jacoco-report-aggregation'
}

dependencies {
    subprojects {
        pluginManager.withPlugin('java') {
            jacocoAggregation project
        }
    }
}

subprojects {
    //...
    testing {
        suites {
            test {
                useJUnitJupiter()
            }
        }
    }
    //...
}

reporting {
    reports {
        testCodeCoverageReport(JacocoCoverageReport) {
            testType = TestSuiteType.UNIT_TEST
        }
    }
}
```
References:
- https://github.com/gradle/gradle/issues/23223#issuecomment-1564067472
