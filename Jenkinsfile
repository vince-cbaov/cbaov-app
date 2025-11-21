// Jenkinsfile (place at repo root)
// Builds and deploys v1, v2, or v3 by parameter or Git tag.
// Assumes Docker Hub, Azure Web App for Containers, and optional NGINX reload.

pipeline {
  agent any

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    choice(name: 'VERSION', choices: ['auto','v1','v2','v3'], description: 'Version folder to build. "auto" derives from Git tag (v1/v2/v3).')
    booleanParam(name: 'RELOAD_NGINX', defaultValue: false, description: 'Reload NGINX reverse proxy after deployment.')
  }

  environment {
    REGISTRY_REPO = 'vincecbaov/cbaov-app'
    AZURE_RG      = 'vinlabs'
    AZURE_WEBAPP  = 'CBAOV-APP'
    REGISTRY_URL  = 'https://index.docker.io/v1/'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        script {
          def ref = sh(returnStdout: true, script: "git describe --tags --exact-match 2>/dev/null || true").trim()
          if (params.VERSION == 'auto') {
            if (ref in ['v1', 'v2', 'v3']) {
              env.BUILD_VERSION = ref
            } else {
              error("Auto version derivation failed. Tag must be v1, v2, or v3. Found: '${ref}'")
            }
          } else {
            env.BUILD_VERSION = params.VERSION
          }
          echo "Building version: ${env.BUILD_VERSION}"
        }
      }
    }

    stage('Validate version folder') {
      steps {
        sh """
          test -f ${env.BUILD_VERSION}/Dockerfile || { echo 'Missing Dockerfile in ${env.BUILD_VERSION}'; exit 1; }
        """
      }
    }

    stage('Build Docker image') {
      steps {
        ansiColor('xterm') {
          sh """
            docker build -t ${REGISTRY_REPO}:${env.BUILD_VERSION} ${env.BUILD_VERSION}
          """
        }
      }
    }

    stage('Push Docker image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          ansiColor('xterm') {
            sh """
              docker login -u "$DOCKER_USER" -p "$DOCKER_PASS"
              docker push ${REGISTRY_REPO}:${env.BUILD_VERSION}
            """
          }
        }
      }
    }

    stage('Deploy to Azure Web App') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'azure-sp-upass', usernameVariable: 'AZ_APP_ID', passwordVariable: 'AZ_SP_SECRET'),
                         string(credentialsId: 'azure-tenant', variable: 'AZ_TENANT_ID')]) {
          ansiColor('xterm') {
            sh """
              az login --service-principal -u "$AZ_APP_ID" -p "$AZ_SP_SECRET" --tenant "$AZ_TENANT_ID"
              az webapp config container set \
                --name ${AZURE_WEBAPP} \
                --resource-group ${AZURE_RG} \
                --docker-custom-image-name ${REGISTRY_REPO}:${env.BUILD_VERSION} \
                --docker-registry-server-url ${REGISTRY_URL}
              az webapp restart --name ${AZURE_WEBAPP} --resource-group ${AZURE_RG}
            """
          }
        }
      }
    }

    stage('Health check') {
      steps {
        ansiColor('xterm') {
          sh """
            curl --fail --ssl-no-revoke https://app.cbaov.com/health
            curl --fail --ssl-no-revoke https://staging.cbaov.com/health
          """
        }
      }
    }

    stage('Reload NGINX (optional)') {
      when { expression { return params.RELOAD_NGINX } }
      steps {
        sshagent(credentials: ['nginx-ssh']) {
          ansiColor('xterm') {
            sh 'ssh -o StrictHostKeyChecking=no vinadmin@nginx-server "sudo systemctl reload nginx"'
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployment of ${env.BUILD_VERSION} succeeded."
    }
    failure {
      echo "Deployment failed."
    }
  }
}
