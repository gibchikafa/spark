/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

pipeline {
  agent { label 'local' }

  options {
    disableConcurrentBuilds()
    skipDefaultCheckout(true)
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'BRANCH_TO_BUILD', defaultValue: 'branch-4.1', description: 'Git branch to build.')
    booleanParam(name: 'FORCE_UPDATE', defaultValue: true, description: 'Pass -U to Maven to refresh snapshot and cached dependency resolution.')
  }

  environment {
    BUILD_PROFILES = 'kubernetes,hadoop-provided,parquet-provided,hive,hadoop-cloud'
    DISTRIBUTION_PROFILES = 'kubernetes,hadoop-provided,parquet-provided,hive,hadoop-cloud,bigtop-dist'
    DOCKER_IMAGE = 'maven:3.9.9-eclipse-temurin-17'
    HOST_MAVEN_REPO = '/home/jenkinsmaster/.m2'
    DISTRIBUTION_REPOSITORY = '/opt/repository/master'
    SPARK_SQL_DEPS_REPOSITORY = '/opt/repository/master/spark-sql-dependencies/spark-4.1'
    MAVEN_LOCAL_REPO = '/maven-repo/repository'
    MAVEN_OPTS = '-Xmx4G'
    MAVEN_SETTINGS = "${WORKSPACE}@tmp/mvn-settings.xml"
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        checkout([$class: 'GitSCM',
          branches: [[name: "${params.BRANCH_TO_BUILD}"]],
          userRemoteConfigs: [[
            url: 'git@github.com:gibchikafa/spark.git',
            credentialsId: 'id_rsa'
          ]]
        ])
      }
    }

    stage('Prepare Maven') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'a0770738-4ef3-4acc-a6ba-097ee6c85b44', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
          sh '''#!/bin/bash -eu
            rm -rf "$WORKSPACE/.m2" "$HOST_MAVEN_REPO/repository/org/apache/spark"
            mkdir -p "$(dirname "$MAVEN_SETTINGS")" "$HOST_MAVEN_REPO/repository"
            cat > "$MAVEN_SETTINGS" <<EOF
<settings>
  <localRepository>${MAVEN_LOCAL_REPO}</localRepository>
  <servers>
    <server>
      <id>HopsEE</id>
      <username>$USERNAME</username>
      <password>$PASSWORD</password>
    </server>
    <server>
      <id>Hops</id>
      <username>$USERNAME</username>
      <password>$PASSWORD</password>
    </server>
    <server>
      <id>hops-releases</id>
      <username>$USERNAME</username>
      <password>$PASSWORD</password>
    </server>
    <server>
      <id>Spark</id>
      <username>$USERNAME</username>
      <password>$PASSWORD</password>
    </server>
    <server>
      <id>spark</id>
      <username>$USERNAME</username>
      <password>$PASSWORD</password>
    </server>
  </servers>
</settings>
EOF
          '''
        }
      }
    }

    stage('Resolve Version') {
      steps {
        sh '''#!/bin/bash -eu
          perl -0ne 'if (m{<artifactId>spark-parent_2\\.13</artifactId>\\s*<version>([^<]+)</version>}) { print "$1"; exit }' pom.xml > version.log
          echo "POM_VERSION=$(cat version.log)"
        '''
      }
    }

    stage('Build and Deploy') {
      steps {
        sh '''#!/bin/bash -eu
          UPDATE_ARG=""
          if [ "$FORCE_UPDATE" = "true" ]; then
            UPDATE_ARG="-U"
          fi

          docker run --rm \
            -u "$(id -u):$(id -g)" \
            -v "$WORKSPACE:$WORKSPACE" \
            -v "$(dirname "$MAVEN_SETTINGS"):$(dirname "$MAVEN_SETTINGS")" \
            -v "$HOST_MAVEN_REPO:/maven-repo" \
            -w "$WORKSPACE" \
            -e GITHUB_ACTIONS=true \
            -e HOME=/tmp \
            -e MAVEN_CONFIG=/tmp/maven-config \
            -e MAVEN_LOCAL_REPO="$MAVEN_LOCAL_REPO" \
            -e MAVEN_OPTS="$MAVEN_OPTS" \
            -e MAVEN_SETTINGS="$MAVEN_SETTINGS" \
            -e BUILD_PROFILES="$BUILD_PROFILES" \
            -e UPDATE_ARG="$UPDATE_ARG" \
            "$DOCKER_IMAGE" \
            bash -lc '
              set -eu
              export PATH="$JAVA_HOME/bin:$PATH"
              JAVA_VERSION="$("$JAVA_HOME/bin/java" -XshowSettings:properties -version 2>&1 | awk -F"= " "/java.specification.version =/{print \\$2; exit}")"
              if [ "$JAVA_VERSION" != "17" ]; then
                echo "Java 17 is required, but JAVA_HOME=$JAVA_HOME reports java.specification.version=$JAVA_VERSION" >&2
                exit 1
              fi
              test -x "$JAVA_HOME/bin/javadoc"
              ./build/mvn -s "$MAVEN_SETTINGS" -Dmaven.repo.local="$MAVEN_LOCAL_REPO" $UPDATE_ARG \
                clean deploy -DskipTests -DdeployAtEnd=true -P"$BUILD_PROFILES"

              # Also publish the same artifacts to the "spark" Nexus repository.
              # No "clean" so the just-built artifacts are reused; altDeploymentRepository
              # overrides the pom distributionManagement (id must match a server in settings.xml).
              # deployAtEnd=true so a mid-reactor failure never leaves a partial set uploaded.
              ./build/mvn -s "$MAVEN_SETTINGS" -Dmaven.repo.local="$MAVEN_LOCAL_REPO" \
                deploy -DskipTests -DdeployAtEnd=true -P"$BUILD_PROFILES" \
                -DaltDeploymentRepository=spark::https://nexus.hops.works/repository/spark
            '
        '''
      }
    }

    stage('Publish Spark SQL Dependencies') {
      steps {
        sh '''#!/bin/bash -eu
          SPARK_VERSION="$(cat version.log)"
          mkdir -p "$SPARK_SQL_DEPS_REPOSITORY"

          cp "sql/core/target/spark-sql_2.13-${SPARK_VERSION}.jar" "$SPARK_SQL_DEPS_REPOSITORY/"
          cp "connector/kafka-0-10-sql/target/spark-sql-kafka-0-10_2.13-${SPARK_VERSION}.jar" "$SPARK_SQL_DEPS_REPOSITORY/"
          cp "connector/avro/target/spark-avro_2.13-${SPARK_VERSION}.jar" "$SPARK_SQL_DEPS_REPOSITORY/"

          ls -l \
            "$SPARK_SQL_DEPS_REPOSITORY/spark-sql_2.13-${SPARK_VERSION}.jar" \
            "$SPARK_SQL_DEPS_REPOSITORY/spark-sql-kafka-0-10_2.13-${SPARK_VERSION}.jar" \
            "$SPARK_SQL_DEPS_REPOSITORY/spark-avro_2.13-${SPARK_VERSION}.jar"
        '''
      }
    }

    stage('Build Distribution') {
      steps {
        sh '''#!/bin/bash -eu
          UPDATE_ARG=""
          if [ "$FORCE_UPDATE" = "true" ]; then
            UPDATE_ARG="-U"
          fi

          docker run --rm \
            -u "$(id -u):$(id -g)" \
            -v "$WORKSPACE:$WORKSPACE" \
            -v "$(dirname "$MAVEN_SETTINGS"):$(dirname "$MAVEN_SETTINGS")" \
            -v "$HOST_MAVEN_REPO:/maven-repo" \
            -w "$WORKSPACE" \
            -e GITHUB_ACTIONS=true \
            -e HOME=/tmp \
            -e MAVEN_CONFIG=/tmp/maven-config \
            -e MAVEN_LOCAL_REPO="$MAVEN_LOCAL_REPO" \
            -e MAVEN_OPTS="$MAVEN_OPTS" \
            -e MAVEN_SETTINGS="$MAVEN_SETTINGS" \
            -e DISTRIBUTION_PROFILES="$DISTRIBUTION_PROFILES" \
            -e UPDATE_ARG="$UPDATE_ARG" \
            "$DOCKER_IMAGE" \
            bash -lc '
              set -eu
              export PATH="$JAVA_HOME/bin:$PATH"
              JAVA_VERSION="$("$JAVA_HOME/bin/java" -XshowSettings:properties -version 2>&1 | awk -F"= " "/java.specification.version =/{print \\$2; exit}")"
              if [ "$JAVA_VERSION" != "17" ]; then
                echo "Java 17 is required, but JAVA_HOME=$JAVA_HOME reports java.specification.version=$JAVA_VERSION" >&2
                exit 1
              fi
              ./dev/make-distribution.sh --name without-hadoop-with-hive --tgz \
                -s "$MAVEN_SETTINGS" -Dmaven.repo.local="$MAVEN_LOCAL_REPO" $UPDATE_ARG \
                -P"$DISTRIBUTION_PROFILES"
            '

          mkdir -p "$DISTRIBUTION_REPOSITORY"
          cp spark-*-bin-without-hadoop-with-hive.tgz "$DISTRIBUTION_REPOSITORY/"
          ls -l spark-*-bin-without-hadoop-with-hive.tgz "$DISTRIBUTION_REPOSITORY"/spark-*-bin-without-hadoop-with-hive.tgz
        '''
      }
    }
  }

  post {
    always {
      sh '''#!/bin/bash
        rm -f "$MAVEN_SETTINGS"
      '''
      archiveArtifacts artifacts: 'version.log,spark-*-bin-without-hadoop-with-hive.tgz', allowEmptyArchive: true
    }
  }
}
