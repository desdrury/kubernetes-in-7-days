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
            withEnv(['DOCKERLOGIN=' + env.dockerLoginCmd.trim(), 'VERSION=' + env.version.trim(), 'COMMIT=' + env.commit.trim()]) {
              sh '$DOCKERLOGIN'
              sh "docker push docker-registry.do.citopro.com/kube7days/documentation:${VERSION}.${COMMIT}"
              sh 'docker push docker-registry.do.citopro.com/kube7days/documentation:latest'
            }
          }

        }
      }

    //   // Scan image for vulnerabilities
    //   container('klar') {
    //     stage('Scan image') {
    //       withEnv(['DOCKERLOGIN=' + env.dockerLoginCmd.trim(), \
    //         'VERSION=' + env.version.trim(), \
    //         'COMMIT=' + env.commit.trim(), \
    //         'CLAIR_ADDR=https://clair.kube.momenton.com.au:443', \
    //         'CLAIR_OUTPUT=High', \
    //         'CLAIR_THRESHOLD=21', \
    //         'DOCKER_USER=AWS' \
    //         ]) {
    //         dockerPassword = sh returnStdout: true, script: "echo $DOCKERLOGIN | cut -d' ' -f6"
    //         sh "DOCKER_PASSWORD=" + dockerPassword.trim() + " /klar 841938870680.dkr.ecr.ap-southeast-2.amazonaws.com/infrastructure-docs:${VERSION}.${COMMIT}"
    //       }  
    //     }
    //   }

    //   // Package Helm Chart
    //   container('helm') {
    //     stage('Package Helm Chart') {
    //       dir('charts') {
    //         sh 'helm init --client-only'
    //         sh 'helm package infrastructure-docs'
    //       }
    //     }
    //   }

    //   // Publish Helm Chart to Chart Museum
    //   // IMPORTANT: This will not overwrite a Chart of the same version.
    //   //            If the Chart has changed then the Chart.yaml version field must be modified.
    //   container('curl') {
    //     stage('Publish Helm Chart') {
    //       dir('charts') {
    //         withCredentials([usernamePassword(credentialsId: 'chart-museum', passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {              
    //           def filename = sh returnStdout: true, script: 'ls *tgz'
    //           withEnv(['FILENAME=' + filename.trim()]) {
    //             sh 'curl --data-binary "@${FILENAME}" "https://${USER}:${PASSWORD}@chartmuseum.kube.momenton.com.au/api/charts"'
    //           }
    //         }
    //       }
    //     }
    //   }

    //   // Deploy Helm Chart
    //   container('helm') {
    //     stage('Deploy Helm Chart') {
    //       dir('charts/infrastructure-docs') {
    //         env.chartVersion = sh returnStdout: true, script: 'grep version Chart.yaml | sed "s/version: //"'
    //       }
    //       withCredentials([usernamePassword(credentialsId: 'chart-museum', passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {
    //         sh 'helm repo add momenton "https://${USER}:${PASSWORD}@chartmuseum.kube.momenton.com.au"'
    //       }
    //       sh 'helm repo update'
    //       withEnv(['CHART_VERSION=' + env.chartVersion.trim(), 'VERSION=' + env.version.trim(), 'COMMIT=' + env.commit.trim()]) {
    //         output = sh returnStdout: true, script: """
    //           helm upgrade --install infrastructure-docs \
    //             --namespace production \
    //             --set image.tag=${VERSION}.${COMMIT} \
    //             momenton/infrastructure-docs --version ${CHART_VERSION}
    //         """

    //         slackSend color: 'good', message: "```" + output + "```"
    //       }
    //     }
    //   }

    }
}
