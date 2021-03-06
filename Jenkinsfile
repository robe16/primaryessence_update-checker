echo "Running Build ID: ${env.BUILD_ID}"

String commit_id
String docker_volumes
String deployLogin
String docker_img_name
def docker_img

node {

    deleteDir()

    stage("parameters") {
        // Parameters passed through from the Jenkins Pipeline configuration
        string(defaultValue: 'https://github.com/robe16/primary_essence.observation_notifier.git', description: 'GitHub URL for checking out project', name: 'githubUrl')
        string(defaultValue: 'primary_essence.observation_notifier', description: 'Name of application for Docker image and container', name: 'appName')
        string(defaultValue: '*', description: 'Location of config json file on host device', name: 'fileConfig')
        string(defaultValue: '*', description: 'Location of history json file on host device', name: 'fileHistory')
        string(defaultValue: '*', description: 'Location of log file on host device', name: 'folderLog')
        //
        docker_volumes = ["-v ${params.fileConfig}:/primary_essence/observation_notifier/config/config.json",
                          "-v ${params.fileHistory}:/primary_essence/observation_notifier/history/history.json",
                          "-v ${params.folderLog}:/primary_essence/observation_notifier/log/logfiles/"].join(" ")
        //
        deployLogin = "${params.deploymentUsername}@${params.deploymentServer}"
        //
    }

    if (params["deploymentServer"]!="*" && params["deploymentUsername"]!="*" && params["serverIp"]!="*" && params["fileConfig"]!="*" && params["fileHistory"]!="*" && params["folderLog"]!="*") {

        stage("checkout") {
            git url: "${params.githubUrl}"
            sh "git rev-parse HEAD > .git/commit-id"
            commit_id = readFile('.git/commit-id').trim()
            echo "Git commit ID: ${commit_id}"
        }

        docker_img_name_build_id = "${params.appName}:${env.BUILD_ID}"
        docker_img_name_latest = "${params.appName}:latest"

        stage("build") {
            try {sh "docker image rm ${docker_img_name_latest}"} catch (error) {}
            sh "docker build -t ${docker_img_name_build_id} ."
            sh "docker tag ${docker_img_name_build_id} ${docker_img_name_latest}"
        }

        stage("deploy"){
            //
            String docker_img_tar = "docker_img.tar"
            //
            try {
                sh "rm ~/${docker_img_tar}"                                                                 // remove any old tar files from cicd server
            } catch(error) {
                echo "No ${docker_img_tar} file to remove."
            }
            sh "docker save -o ~/${docker_img_tar} ${docker_img_name_build_id}"                               // create tar file of image
            sh "scp -v -o StrictHostKeyChecking=no ~/${docker_img_tar} ${deployLogin}:~"                    // xfer tar to deploy server
            sh "ssh -o StrictHostKeyChecking=no ${deployLogin} \"docker load -i ~/${docker_img_tar}\""      // load tar into deploy server registry
            sh "ssh -o StrictHostKeyChecking=no ${deployLogin} \"docker tag ${docker_img_name_build_id} ${docker_img_name_latest}\""
            sh "ssh -o StrictHostKeyChecking=no ${deployLogin} \"rm ~/${docker_img_tar}\""                  // remove the tar file from deploy server
            sh "rm ~/${docker_img_tar}"                                                                     // remove the tar file from cicd server
            //
        }

        stage("start container"){
            // Stop existing container if running
            sh "ssh ${deployLogin} \"docker rm -f ${params.appName} && echo \"container ${params.appName} removed\" || echo \"container ${params.appName} does not exist\"\""
            // Start new container
            sh "ssh ${deployLogin} \"docker run --restart unless-stopped -e TZ=Europe/London -d ${docker_volumes} --name ${params.appName} ${docker_img_name_latest}\""
        }

    } else {
        echo "Build cancelled as required parameter values not provided by pipeline configuration"
    }

}
