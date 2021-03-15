//执行Helm的方法
def helmDeploy(Map args) {
    if(args.init){
        println "Helm 初始化"
        println "添加私有helm仓库"
        sh "helm repo add stable http://${args.username}:${args.password}@${args.url}/repository/${args.repo_name}/"
        println "更新仓库信息"
        sh "helm repo update"
        println "查看仓库列表"
        sh "helm repo list"
    } else if (args.dry_run) {
        println "尝试 Helm 部署，验证是否能正常部署"
        sh "helm upgrade --install ${args.name} --namespace ${args.namespace} --set ${images},${tag} stable/${args.template} --dry-run --debug"
    } else {
        println "正式 Helm 部署"
        sh "helm upgrade --install ${args.name} --namespace ${args.namespace} --set ${images},${tag} stable/${args.template}"
    }
}

// jenkins slave 执行流水线任务
def label = "jnlp-test"
podTemplate(label: label,cloud: 'kubernetes' ){
    node (label) {
        stage('Git阶段'){
            echo "Git 阶段"
            git branch: "master" ,changelog: true, credentialsId: 'github' , url: "git@code.aliyun.com:dongye/test1.git"
        }
        stage('Maven阶段'){
            echo "Maven 阶段"
            container('maven') {
                //这里引用上面设置的全局的 settings.xml 文件，根据其ID将其引入并创建该文件
                configFileProvider([configFile(fileId: "global-maven-settings", targetLocation: "settings.xml")]){
                    sh "mvn clean install -Dmaven.test.skip=true --settings settings.xml"
                }
            }
        }
        stage('Docker阶段'){
            echo "Docker 阶段"
            container('docker') {
                // 读取pom参数
                echo "读取 pom.xml 参数"
                pom = readMavenPom file: './pom.xml'
                // 设置镜像仓库地址
                hub = "docker.dongye.com"
                // 设置仓库项目名
                project_name = "test"
                echo "编译 Docker 镜像"
                //此方法是设置docker仓库地址，然后选择存了用户名、密码的凭据ID进行验证。注意，只有在此方法之中才生效。
                docker.withRegistry("http://${hub}", "docker-registry-credential") {
                    echo "构建镜像"
                    // 设置推送到docker仓库的test项目下，并用pom里面设置的项目名与版本号打标签
                    def customImage = docker.build("${hub}/${project_name}/${pom.artifactId}:${pom.version}")
                    echo "推送镜像"
                    customImage.push()
                    echo "删除镜像"
                    sh "docker rmi ${hub}/${project_name}/${pom.artifactId}:${pom.version}"
                }
            }
        }
        stage('Helm阶段'){
            container('helm-kubectl') {
                // 提供 kubectl 执行的环境，其中得设置存储了 token 的凭据ID和 kubernetes api 地址
                withKubeConfig([credentialsId: "09468c56-d0f5-4393-9c4c-572f4a4b2f95",serverUrl: "https://kubernetes.default.svc.dongye.local"]) {
                    // 设置参数
                    images = "Deployment.Containers.Image=${hub}/${project_name}/${pom.artifactId}"
                    tag = "Deployment.Containers.ImageTag=${pom.version}"
                    template = "mychart"   //helm 模板名称
                    repo_url = "192.168.38.250:30111" //仓库地址
                    repo_name = "helm"  //仓库名称
                    repo_username = "admin"  //仓库用户名
                    repo_password = "123456" //仓库密码
                    app_name = "${pom.artifactId}" //服务名称
                    // 检测是否存在yaml文件
                    // def values = ""
                    // if (fileExists('values.yaml')) {
                    //     values = "-f values.yaml"
                    // }
                    // 执行 Helm 方法
                    echo "Helm 初始化"
                    helmDeploy(init: true ,url: "${repo_url}", repo_name: "${repo_name}",username: "${repo_username}", password: "${repo_password}");
                    echo "Helm 执行部署测试"
                    helmDeploy(init: false ,dry_run: true ,name: "${app_name}" ,namespace: "public-service" ,image: "${images}" ,tag: "${tag}" ,template: "${template}")
                    echo "Helm 执行正式部署"
                    helmDeploy(init: false ,dry_run: false ,name: "${app_name}" ,namespace: "public-service",image: "${images}" ,tag: "${tag}" ,template: "${template}")
                }
            }
        }
    }
}