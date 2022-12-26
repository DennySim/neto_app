node { 
    
    stage("Git checkout"){
        def gitUrl = scm.getUserRemoteConfigs()[0].getUrl();
        git branch: 'main', url: "${gitUrl}"
    }

    stage("Docker registry login") {
        withCredentials([file(credentialsId: 'keyjson', variable: 'keyjson')]) {
            configFileProvider([configFile(fileId: 'b114ffa2-8221-42da-8099-d5f121546f28', targetLocation: 'JsonConfig')]) {
                config = readJSON file: 'JsonConfig'
                sh "cat $keyjson | docker login --username json_key --password-stdin ${config['registry_url']}"
            }   
        }
    }
    stage("Build and push image") {
      docker.withRegistry("https://${config.registry_url}") {
        def customImage = docker.build("${config.registry_url}/${config.registry_id}/${config.registry_repo_name}/${config.image_name}:${env.BUILD_ID}", "--no-cache .")
        customImage.push()
        customImage.push("latest")
      }
    }
    stage("Prepare app-service kubernetes manifest "){
      sh "sed -i 's+registry_url+${config.registry_url}/${config.registry_id}/${config.registry_repo_name}/${config.image}:latest+' ./app/deploy/kubernetes/main.yml"
    }
    stage("Deploy app-service on kubernetes") {
        def tagging = sh(returnStdout: true, script: "git tag --contains").trim()
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'kubeconfig')]) {
            if (tagging){
                sh 'echo HAVE TAGS - HAVE CONTAINERS TO DEPLOY'
                def getDeploy = sh(returnStdout: true, script: "kubectl get deployment --kubeconfig ${kubeconfig} --no-headers=true 2>&1").trim()
                def isExistDeploy = sh (returnStdout: true, script: "echo ${getDeploy}").contains('coolapp')          
                if (isExistDeploy == true){
                    /*  UPDATE APP CONTAINER WITH LATEST VERSION ON KUBERNETES */
                    sh "kubectl rollout restart deploy coolapp-deployment --kubeconfig ${kubeconfig}"
                }
                else {
                    /*  FIRST DEPLOY APP CONTAINER ON KUBERNETES */
                    sh "kubectl apply -f ./app/deploy/kubernetes/main.yml --kubeconfig ${kubeconfig}"
                }
            }
            else{
                sh 'echo NO TAGS - NO CONTAINERS TO DEPLOY'
            }
        }      
    }    
}