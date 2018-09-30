def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, 
  serviceAccount: 'jenkins',
  containers: [
    containerTemplate(
      name: 'docker', 
      image: 'docker:17.12.1-ce',
      ttyEnabled: true, 
      command: 'cat',
      envVars: [
        envVar(key: 'DOCKER_HOST', value: 'tcp://dind:2375')
      ]
    ),
    containerTemplate(
      name: 'gitbook', 
      image: 'fellah/gitbook', 
      ttyEnabled: true, 
      command: 'cat'
    ),
    containerTemplate(
      name: 'helm', 
      image: 'lachlanevenson/k8s-helm:v2.8.1', 
      ttyEnabled: true, 
      command: 'cat'
    ),
    containerTemplate(
      name: 'curl', 
      image: 'appropriate/curl', 
      ttyEnabled: true, 
      command: 'cat'
    ),
    containerTemplate(
      name: 'klar', 
      image: 'desdrury/klar', 
      ttyEnabled: true, 
      command: 'cat'
    )
  ],
  volumes: [
    emptyDirVolume(mountPath: '/var/lib/docker', memory: false)
  ]) {

    node(label) {
      
      // Checkout code    
      container('jnlp') {
        stage('Checkout code') {
          git credentialsId: 'desdrury-https-github', url: 'https://github.com/desdrury/kubernetes-in-7-days.git'
          env.commit = sh returnStdout: true, script: 'git rev-parse HEAD'
        }
      }

      // Generate website
      container('gitbook') {
        stage('Generate website') {
          sh 'gitbook install'
          sh 'gitbook build'
        }
      }

      // Build and push image
      container('docker') {
        stage('Build image') {
          env.version = sh returnStdout: true, script: 'cat build.number'            
          withEnv(['VERSION=' + env.version.trim(), 'COMMIT=' + env.commit.trim()]) {
            sh """
              docker build \
                -t docker-registry.do.citopro.com/kube7days/documentation:${VERSION}.${COMMIT}  \
                -t docker-registry.do.citopro.com/kube7days/documentation:latest \
                .
            """
          }
        }

        stage('Push image') {
          withDockerRegistry([credentialsId: 'docker-user', url: 'https://docker-registry.do.citopro.com']) {
            withEnv(['VERSION=' + env.VERSION.trim(), 'COMMIT=' + env.COMMIT.trim()]) {
              sh "docker push docker-registry.do.citopro.com/kube7days/documentation:${VERSION}.${COMMIT}"
              sh 'docker push docker-registry.do.citopro.com/kube7days/documentation:latest'
            }
          }

        }
      }

      // Scan image for vulnerabilities
      container('klar') {
        stage('Scan image') {
          withCredentials([usernamePassword(credentialsId: 'docker-user', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USER')]) {
            withEnv([ \
              'VERSION=' + env.version.trim(), \
              'COMMIT=' + env.commit.trim(), \
              'CLAIR_ADDR=https://clair.do.citopro.com:443', \
              'CLAIR_OUTPUT=High', \
              'CLAIR_THRESHOLD=21'
              ]) {
              sh "/klar docker-registry.do.citopro.com/kube7days/documentation:${VERSION}.${COMMIT}"
            }
          }
        }
      }

      // Package Helm Chart
      container('helm') {
        stage('Package Helm Chart') {
          dir('charts') {
            sh 'helm init --client-only'
            sh 'helm package kube7days'
          }
        }
      }

      // Publish Helm Chart to Chart Museum
      // IMPORTANT: This will not overwrite a Chart of the same version.
      //            If the Chart has changed then the Chart.yaml version field must be modified.
      container('curl') {
        stage('Publish Helm Chart') {
          dir('charts') {
            withCredentials([usernamePassword(credentialsId: 'chart-museum', passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {              
              def filename = sh returnStdout: true, script: 'ls *tgz'
              withEnv(['FILENAME=' + filename.trim()]) {
                sh 'curl --data-binary "@${FILENAME}" "https://${USER}:${PASSWORD}@chartmuseum.do.citopro.com/api/charts"'
              }
            }
          }
        }
      }

      // Deploy Helm Chart
      container('helm') {
        stage('Deploy Helm Chart') {
          dir('charts/kube7days') {
            env.chartVersion = sh returnStdout: true, script: 'grep version Chart.yaml | sed "s/version: //"'
          }
          withCredentials([usernamePassword(credentialsId: 'chart-museum', passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {
            sh 'helm repo add citopro "https://${USER}:${PASSWORD}@chartmuseum.do.citopro.com"'
          }
          sh 'helm repo update'
          withEnv(['CHART_VERSION=' + env.chartVersion.trim(), 'VERSION=' + env.version.trim(), 'COMMIT=' + env.commit.trim()]) {
            
            // Oauth protected
            output = sh returnStdout: true, script: """
              helm upgrade --install kube7days \
                --namespace production \
                --set image.tag=${VERSION}.${COMMIT} \
                citopro/kube7days --version ${CHART_VERSION}
            """

            slackSend color: 'good', message: "```" + output + "```"

            // No OAuth protection
            output = sh returnStdout: true, script: """
              helm upgrade --install kube7days-noauth \
                --namespace production \
                --set oauth=false \
                --set ingress.hosts[0]=kube7days.staging.do.citopro.com \
                --set ingress.tls[0].secretName=kube7days-noauth-tls \
                --set ingress.tls[0].hosts[0]=kube7days.staging.do.citopro.com \
                --set image.tag=${VERSION}.${COMMIT} \
                citopro/kube7days --version ${CHART_VERSION}
            """

            slackSend color: 'good', message: "```" + output + "```"

            // Packt
            output = sh returnStdout: true, script: """
              helm upgrade --install kube7days-packt \
                --namespace production \
                --set oauth=false \
                --set ingress.annotations."nginx\.ingress\.kubernetes\.io/auth-type"=basic \
                --set ingress.annotations."nginx\.ingress\.kubernetes\.io/auth-secret"=basic-auth \
                --set ingress.annotations."nginx\.ingress\.kubernetes\.io/auth-realm"='Authentication Required - kube7days' \
                --set ingress.hosts[0]=kube7days.info \
                --set ingress.tls[0].secretName=kube7days-packt-tls \
                --set ingress.tls[0].hosts[0]=kube7days.info \
                --set image.tag=${VERSION}.${COMMIT} \
                citopro/kube7days --version ${CHART_VERSION}
            """

            slackSend color: 'good', message: "```" + output + "```"
          }
        }
      }

    }
}
