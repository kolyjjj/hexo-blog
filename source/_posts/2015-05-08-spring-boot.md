title: "sprint boot"
date: 2015-05-08 23:00:51
categories: 
- Tech
tags: 
- spring
---

### What is Spring Boot?

{% blockquote Dan Woods  http://www.infoq.com/articles/microframeworks1-spring-boot &lt;&lt;microframeworks spring boot&gt;&gt; %}

Spring Boot is a brand new framework from the team at Pivotal, designed to simplify the bootstrapping and development of a new Spring application. The framework takes an opinionated approach to configuration, freeing developers from the need to define boilerplate configuration. In that, Boot aims to be a front-runner in the ever-expanding rapid application development space.

{% endblockquote %}
<!--more-->
### Benefits of using Spring Boot?

* no xml
* fast, only need to include one or two jars
* easy to integrate spring facilities, like spring data

### An example with gradle

{% codeblock lang:groovy %}

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:1.2.3.RELEASE")
    }
}

apply plugin: 'spring-boot'
apply plugin: 'idea'
apply plugin: 'java'

sourceCompatibility = 1.7

jar {
    baseName = 'spring-boot'
    version = '1.0-SNAPSHOT'
}


repositories {
    mavenCentral();
}

dependencies {
    compile("org.springframework.boot:spring-boot-starter-web:1.2.3.RELEASE")
    testCompile('junit:junit:4.12')
}

task wrapper(type: Wrapper) {
    gradleVersion = '2.4'
}

{% endcodeblock %}

### An official example to start with
<http://spring.io/guides/gs/spring-boot/>
