//def version, mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"
import groovy.json.JsonSlurper
def mvnCmd = "mvn"
def version="1.0"
def app_name="angular-test"
def quay_host="docker.io"
def quay_org="vizuri"

def nextVersionFromGit(scope) {
    def latestVersion = sh returnStdout: true, script: 'git describe --tags "$(git rev-list --tags=*.*.* --max-count=1 2> /dev/null)" 2> /dev/null || echo 0.0.0'
    echo "latestVersion: ${latestVersion}"
    def (major, minor, patch) = latestVersion.tokenize('.').collect { it.toInteger() }
    
    def nextVersion
    switch (scope) {
        case 'major':
            nextVersion = "${major + 1}.0.0"
            break
        case 'minor':
            nextVersion = "${major}.${minor + 1}.0"
            break
        case 'patch':
            nextVersion = "${major}.${minor}.${patch + 1}"
            break
    }
    nextVersion
}

pipeline {

  agent {
    label 'nodejs'
  }
  stages {
      stage("Checkout") {
        steps {
          sshagent (credentials: ['github-jenins']) {
	          sh  """
	            echo '->> In Checkout <<-'
	            echo "Branch Name ${BRANCH_NAME}"
	            git checkout ${BRANCH_NAME}
	            git fetch --tags
	            echo '->> Done Checkout <<-'
	          """
			}
		}
      }
    

    stage('Build App') {
      steps {
        script {
        sh '''
          npm install -g @angular/cli 
          ng build --prod --base-href="/"'
       '''
      }
    }
    }
 
    stage('Build Container') {
      steps {
            container("buildah") {
              withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'rh-credentials',
usernameVariable: 'RH_USERNAME', passwordVariable: 'RH_PASSWORD'], [$class: 'UsernamePasswordMultiBinding', credentialsId: 'quay-credentials',
usernameVariable: 'QUAY_USERNAME', passwordVariable: 'QUAY_PASSWORD']]) {
                sh  """
                  echo '->> In Buildah ${app_name}-${version} <<-'
                  buildah login -u $RH_USERNAME -p $RH_PASSWORD registry.redhat.io
                  buildah login -u $QUAY_USERNAME -p $QUAY_PASSWORD ${quay_host}                 
                  buildah bud -t ${quay_host}/${quay_org}/${app_name}:${version} .
                  buildah push ${quay_host}/${quay_org}/${app_name}:${version}
                  echo '->> Done Buildah <<-'
                """
            }
          }
        }
    }
   
    stage('Helm Package') {
     when {
        expression { ENVIRONMENT != 'prod' }
      }      
      steps {     
        container("buildah") {
            sh  """
              echo '->> In Helm Package <<-'
              helm package helm/ --version=${version} --app-version=${version} 
              echo '->> Done Helm Package <<-'
            """
        }
      }
    }
    stage('Deploy') {
      when {
        expression { ENVIRONMENT != 'prod' }
      }
      steps {
        container("buildah") { 
          sh  """
            echo '->> In Helm Install DEV ${app_name}-${version} <<-'
            helm upgrade --install --set env=${ENVIRONMENT} ${ENVIRONMENT}-${app_name} ${app_name}-${version}.tgz --namespace=default
            echo '->> Done Helm Install <<-'
          """	            
        }
      }
    }	 	
    
  }
}
