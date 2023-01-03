def customImage = ''
pipeline {
    agent any
    stages {
        stage("Build docker image and push") {
            steps {
                script {
                    withCredentials([file(credentialsId: 'keyjson', variable: 'keyjson')]) {
                        configFileProvider([configFile(fileId: 'b114ffa2-8221-42da-8099-d5f121546f28', targetLocation: 'JsonConfig')]) {
                            config = readJSON file: 'JsonConfig'
                            sh "cat $keyjson | docker login --username json_key --password-stdin ${config['registry_url']}"
                        }   
                    }
                    docker.withRegistry("https://${config.registry_url}") {
                        customImage = docker.build("${config.registry_url}/${config.registry_id}/${config.registry_repo_name}/${config.image_name}:${env.BUILD_ID}", "--no-cache .")
                        customImage.push()
                        customImage.push("latest")
                    }   
                }   
            }
        }
        stage('Push tagged image') {
        when { buildingTag() }
            steps {
                script { 
                    docker.withRegistry("https://${config.registry_url}") {
                        customImage.push("$TAG_NAME")
                    }
                }
            }
        }
        stage('Deploy') {
            when { buildingTag() }
            steps {
                script {
                    sh "sed -i 's+registry_url+${config.registry_url}/${config.registry_id}/${config.registry_repo_name}/${config.image_name}:${TAG_NAME}+' ./app/deploy/kubernetes/main.yml"   
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'kubeconfig')]) {  
                        def getDeploy = sh(returnStdout: true, script: "kubectl get deployment --kubeconfig ${kubeconfig} --no-headers=true 2>&1").trim()
                        def isExistDeploy = sh (returnStdout: true, script: "echo ${getDeploy}").contains('coolapp')          
                        if (isExistDeploy == true){
                            /*  UPDATE APP CONTAINER WITH LATEST VERSION ON KUBERNETES */
                            sh "kubectl rollout restart deploy coolapp-deployment --kubeconfig ${kubeconfig}"
                        }
                        else {
                            /* FIRST DEPLOY APP CONTAINER ON KUBERNETES */
                            sh "kubectl apply -f ./app/deploy/kubernetes/main.yml --kubeconfig ${kubeconfig}"
                        }
                    }
                }
            }
        }
    }
}
