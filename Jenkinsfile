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

/*
    agent any
    agent { label 'master' }
    tools {
        jdk "jdk8"
    }
    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipDefaultCheckout(true)
    }
*/
    environment {
        PATH = "/app/apache-maven-3.6.1/bin:/app/node-v10.16.0-linux-x64/bin:/app/bin:$PATH"
        // BuildTag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    }
    parameters {
        extendedChoice(
        name: 'Module',
        description: '模块',
        type: 'PT_CHECKBOX',
        visibleItemCount: 10,
        multiSelectDelimiter: ',',
        quoteValue: false,
        value:'springboot,springcloud-eureka-server,springcloud-eureka-client,tomcat,vue,test',
        defaultValue: '',
        saveJSONParameterToFile: false
        )
        string(name: 'Branch', description: '分支', defaultValue: 'master')
    }

    stages {
        stage('Get Code') {
            steps {
                // git branch: "$Branch",  credentialsId: 'gitlab', url: 'http://gitlab.k8s.maimaiti.site/root/jenkins-demo.git'
                checkout scm
                withCredentials([usernamePassword(credentialsId: 'harbor', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                    sh "docker login -u ${dockerHubUser} -p ${dockerHubPassword} harbor.k8s.maimaiti.site"
                }
                script {
                    env.BuildTag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                }
            }
        }

        // stage('Pre Deploy'){
        //     steps{
        //         script{
        //             env.InputMap = input (
        //                 message: '准备发布到哪个环境？',
        //                 parameters:[
        //                     choice(name: 'ENV', choices: ['dev', 'sit','uat','prd','default'], description: '选择发布到什么环境？'),
        //                     string(name: 'myparam', defaultValue: '', description: '')
        //                 ]
        //             )
        //         }
        //     }
        // }

        stage('Deploy springboot') {
            when {
                expression { return "$params.Module".contains('springboot')}
            }
            steps {
                sh '''
                    cd springboot/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/library/jenkins-demo-springboot:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status deployment jenkins-demo-springboot
                '''
            }
        }
        stage('Deploy springcloud-eureka-server') {
            when {
                expression { return "$params.Module".contains('springcloud-eureka-server')}
            }
            steps {
                sh '''
                    cd springcloud/eureka-server/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/library/jenkins-demo-springcloud-eureka-server:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status statefulset eureka
                '''
            }
        }
        stage('Deploy springcloud-eureka-client') {
            when {
                expression { return "$params.Module".contains('springcloud-eureka-client')}
            }
            steps {
                sh '''
                    cd springcloud/eureka-client/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/library/jenkins-demo-springcloud-eureka-client:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status deployment eureka-client
                '''
            }
        }


        stage('Deploy tomcat') {
            when {
                expression { return "$params.Module".contains('tomcat')}
            }
            steps {
                sh '''
                    cd tomcat/
                    mvn -Dmaven.test.skip=true clean package
                    imageName=harbor.k8s.maimaiti.site/library/jenkins-demo-tomcat:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status deployment jenkins-demo-tomcat
                '''
            }
        }
        stage('Deploy vue') {
            when {
                expression { return "$params.Module".contains('vue')}
            }
            steps {
                    // source /etc/profile
                sh '''
                    cd vue/
                    alias cnpm="npm --registry=https://registry.npm.taobao.org --cache=/app/.npm/.cache/cnpm --disturl=https://npm.taobao.org/dist --userconfig=/app/.cnpmrc"
                    cnpm install; cnpm run build; tar zcf dist.tar.gz -C dist/ .
                    imageName=harbor.k8s.maimaiti.site/library/jenkins-demo-vue:${BuildTag}
                    docker build -t $imageName .
                    docker push $imageName
                    docker rmi $imageName
                    sed -i "s/<BUILD_TAG>/${BuildTag}/" k8s.yaml
                    kubectl --kubeconfig=/app/.kube/config -n kube-system apply -f k8s.yaml --record
                    kubectl --kubeconfig=/app/.kube/config -n kube-system rollout status deployment jenkins-demo-vue
                '''
            }
        }
        stage('Deploy test local') {
            when {
                expression { return "$params.Module".contains('test')}
            }
            steps {
                dir('test') {
                    //  sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
                    /*
                        echo ${InputMap["ENV"]}
                        echo ${InputMap.ENV}
                        sleep 600000
                    */
                    sh '''
                        printenv | grep -E 'BuildTag|PATH'
                    '''
                }

                // container('maven') {
                // sh 'mvn -version'
                // }

            }
        }
    }
}


