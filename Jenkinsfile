pipeline {
    agent {
      label "jenkins-gradle"
    }
    environment {
      ORG               = 'venkatvar'
      APP_NAME          = 'javaspringboot-549'
      CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    }
    stages {
      stage('CI Build and push snapshot') {
        when {
          branch 'PR-*'
        }
        environment {
          PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container('gradle') {
            // TODO 
            //sh "mvn versions:set -DnewVersion=$PREVIEW_VERSION"
            sh "gradle clean build"
            sh 'export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml'

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }

          dir ('./charts/preview') {
           container('gradle') {
             sh "make preview"
             sh "jx preview --app $APP_NAME --dir ../.."
           }
          }
        }
      }
      stage('Build Release') {
        when {
          branch env.BRANCH_NAME
        }
        steps {
          container('gradle') {
            // ensure we're not on a detached head
            sh "git checkout ${env.BRANCH_NAME}"
            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            // TODO
            //sh "mvn versions:set -DnewVersion=\$(cat VERSION)"
          }
          dir ('./charts/javaspringboot-549') {
            container('gradle') {
              sh "make tag"
            }
          }
          container('gradle') {
            sh 'gradle clean build'

            sh 'export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml'

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
      stage('Promote to Environments') {
        when {
          branch env.BRANCH_NAME
        }
        steps {
          dir ('./charts/javaspringboot-549') {
            container('gradle') {
              sh 'jx step changelog --version v\$(cat ../../VERSION)'

              // release the helm chart
              sh 'jx step helm release'

              // promote through all 'Auto' promotion Environments
              sh 'jx step helm apply --namespace=jx-staging --name=javaspringboot-549'
            }
          }
        }
      }
    }
    post {
        always {
            cleanWs()
        }
    }
  }
