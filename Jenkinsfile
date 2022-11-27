
node {
    stage("Git checkout"){
        checkout scm
    }

    stage("Docker registry login") {
      def yandex_congig_file = "/var/lib/jenkins/.config/yandex-cloud"  
      sh "cat ${yandex_congig_file}/key_diplom.json | docker login --username json_key --password-stdin ${params.registry_url}"
    }
    stage("Build and push image") {
      docker.withRegistry("https://${params.registry_url}") {
        def customImage = docker.build("${params.registry_url}/${params.registry_id}/${params.registry_repo_name}/${params.image}:${env.BUILD_ID}", "--no-cache .")
        customImage.push()
        customImage.push("latest")
      }
    }
    
    stage("Prepare app-service kubernetes manifest "){
      sh "sed -i 's+registry_url+${params.registry_url}/${params.registry_id}/${params.registry_repo_name}/${params.image}:latest+' ./app/deploy/kubernetes/main.yml"
    }
    
    stage("Deploy app-service on kubernetes") {
      def kuber_congig_file = "/var/lib/jenkins/.config/kubernetes/"  
      def tagging = sh(returnStdout: true, script: "git tag --contains").trim()
    
        if (tagging){
            def getDeploy = sh(returnStdout: true, script: "kubectl get deployment --kubeconfig /${kuber_congig_file}/config --no-headers=true 2>&1").trim()
            def isExistDeploy = sh (returnStdout: true, script: "echo ${getDeploy}").contains('coolapp')          
            if (isExistDeploy == true){
              /*  UPDATE APP CONTAINER WITH LATEST VERSION ON KUBERNETES */
                sh "kubectl rollout restart deploy coolapp-deployment --kubeconfig /${kuber_congig_file}/config"
            }
            else {
              /*  FIRST DEPLOY APP CONTAINER ON KUBERNETES */
              sh "kubectl apply -f ./app/deploy/kubernetes/main.yml --kubeconfig /${kuber_congig_file}/config"
            }
        }
        else{
          sh 'echo NO TAGS - NO CONTAINERS TO DEPLOY'
        }
    }
}
