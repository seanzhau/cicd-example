def MAVEN = "mvn -s configuration/cicd-settings-nexus3.xml"
def templatePath = "https://code.aliyun.com/ipaas/template/raw/demo/create_apps_deployment.yaml"

pipeline {
  agent {
    label 'maven'
  }

  options {
    timeout(time: 1, unit: 'DAYS') 
  }

  stages {
    stage('Maven Build') {
      steps {
        sh 'printenv'
        sh "${MAVEN} install -DskipTests=true"
      }
    }

    stage('JUnit Test') {
      steps { 
        sh "${MAVEN} test"
        step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
      }
    }

    stage('Analysis with Sonarqube') {
      steps {
        script {
          sh "${MAVEN} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
        }
      }
    }

    stage('Publish to Nexus') {
      steps {
        sh "${MAVEN} deploy -DskipTests=true -P nexus3"
      }
    }

    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.TEAM) {
              openshift.selector("bc", env.APPS + "-build").startBuild("--from-file=target/openshift-tasks.war", "--wait=true")
            }
          }
        }
      }
    }

    stage('Create DEV') {
      when {
        expression {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-dev") {
              return !openshift.selector("dc", env.APPS).exists()
            }
          }
        }
      }

      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-dev") {
              if (openshift.selector("svc", env.APPS).exists()) { 
                openshift.selector("svc", env.APPS).delete()
              }
     
              if (openshift.selector("route", env.APPS).exists()) { 
                openshift.selector("route", env.APPS).delete()
              }

              def app = openshift.newApp(templatePath, "--param TEAM=" + env.TEAM + " --param APPS=" + env.APPS)
            }
          }
        }
      }
    }

    stage('Deploy DEV') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-dev") {
              def dc = openshift.selector("dc", env.APPS)

              timeout(time: 1, unit:'MINUTES') { 
                openshift.selector("dc", env.APPS).related('pods').untilEach(1) {
                  return (it.object().status.phase == "Running")
                }
              }
            }
          }
        }
      }
    }

    stage('Create STG') {
      when {
        expression {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-stg") {
              return !openshift.selector("dc", env.APPS).exists()
            }
          }
        }
      }

      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-stg") {
              if (openshift.selector("svc", env.APPS).exists()) { 
                openshift.selector("svc", env.APPS).delete()
              }
     
              if (openshift.selector("route", env.APPS).exists()) { 
                openshift.selector("route", env.APPS).delete()
              }

              def app = openshift.newApp(templatePath, "--param TEAM=" + env.TEAM + " --param APPS=" + env.APPS)
            }
          }
        }
      }
    }

    stage('Deploy STG') {
      steps {
        timeout(time: 12, unit:'HOURS') {
          input message: "Approval deploy to STG?", ok: "Proceed"
        }

        script {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-stg") {
              def dc = openshift.selector("dc", env.APPS)

              timeout(time: 2, unit:'MINUTES') { 
                openshift.selector("dc", env.APPS).related('pods').untilEach(1) {
                  return (it.object().status.phase == "Running")
                }
              }
            }
          }
        }
      }
    }

    stage('Create PRD') {
      when {
        expression {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-prd") {
              return !openshift.selector("dc", env.APPS).exists()
            }
          }
        }
      }

      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-prd") {
              if (openshift.selector("svc", env.APPS).exists()) { 
                openshift.selector("svc", env.APPS).delete()
              }
     
              if (openshift.selector("route", env.APPS).exists()) { 
                openshift.selector("route", env.APPS).delete()
              }

              def app = openshift.newApp(templatePath, "--param TEAM=" + env.TEAM + " --param APPS=" + env.APPS)
            }
          }
        }
      }
    }

    stage('Deploy PRD') {
      steps {
        timeout(time: 60, unit:'MINUTES') {
          input message: "Approval deploy to STG?", ok: "Proceed"
        }

        script {
          openshift.withCluster() {
            openshift.withProject(env.TEAM + "-prd") {
              def dc = openshift.selector("dc", env.APPS)

              timeout(time: 2, unit:'MINUTES') { 
                openshift.selector("dc", env.APPS).related('pods').untilEach(1) {
                  return (it.object().status.phase == "Running")
                }
              }
            }
          }
        }
      }
    }
  }
}