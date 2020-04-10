pipeline {
    agent {
        kubernetes {
        label 'jnlp'
        yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    some-label: some-label-value
spec:
  containers:
  - name: jnlp
    image: 'harbor.k8s.maimaiti.site/library/jnlp-slave:3.27-1-myv4'
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    resources:
        requests:
            cpu: 50m
            memory: 1000Mi
    volumeMounts:
    - name: "volume-0"
      mountPath: "/app"
    - name: "volume-1"
      mountPath: "/var/run/docker.sock"
  volumes:
  - name: "volume-0"
    persistentVolumeClaim:
      claimName: "jnlp"
  - name: "volume-1"
    hostPath:
      path: "/var/run/docker.sock"
        """
        }
    }

    environment {
        PATH = "/app/apache-maven-3.6.1/bin:/app/node-v10.16.0-linux-x64/bin:/app/bin:$PATH"
    }
    parameters {
        extendedChoice(
        name: 'Module',
        description: '模块',
        type: 'PT_CHECKBOX',
        visibleItemCount: 100,
        multiSelectDelimiter: ',',
        quoteValue: false,
        value:'eureka-server,eureka-client,user-service,api-gateway',
        defaultValue: '',
        saveJSONParameterToFile: false
        )
        string(name: 'Branch', description: '分支', defaultValue: 'master')
    }

    stages {
        stage('Get Code') {
            steps {
                checkout scm
                withCredentials([usernamePassword(credentialsId: 'harbor', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                    sh "docker login -u ${dockerHubUser} -p ${dockerHubPassword} harbor.k8s.maimaiti.site"
                }
                script {
                    env.BuildTag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                }
            }
        }

        stage('Deploy eureka-server') {
            when {
                expression { return "$params.Module".contains('eureka-server')}
            }
            steps {
                sh '''
                    cd eureka-server/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/mydemo2/eureka-server:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status statefulset eureka
                '''
            }
        }
        stage('Deploy eureka-client') {
            when {
                expression { return "$params.Module".contains('eureka-client')}
            }
            steps {
                sh '''
                    cd eureka-client/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/mydemo2/eureka-client:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status deployment eureka-client
                '''
            }
        }
        stage('Deploy user-service') {
            when {
                expression { return "$params.Module".contains('user-service')}
            }
            steps {
                sh '''
                    cd user-service/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/mydemo2/user-service:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status deployment user-service
                '''
            }
        }
        stage('Deploy api-gateway') {
            when {
                expression { return "$params.Module".contains('api-gateway')}
            }
            steps {
                sh '''
                    cd api-gateway/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/mydemo2/api-gateway:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status deployment api-gateway
                '''
            }
        }
    }
}


