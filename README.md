# Como instalar uma instância Jenkins com Helm

O Kubernetes (K8s) tornou-se uma das plataformas mais usadas para hospedar contêineres Docker. O Kubernetes oferece recursos avançados de orquestração, recursos de rede, segurança integrada, gerenciamento de usuários, alta disponibilidade, gerenciamento de volume, um amplo ecossistema de ferramentas de suporte e muito mais.

Uma ferramenta de suporte é o Helm , que fornece funcionalidade de gerenciamento de pacotes para o Kubernetes. Os aplicativos implantados pelo Helm são definidos em gráficos e o Jenkins fornece um gráfico do Helm para implantar uma instância do Jenkins no Kubernetes.

Nesta postagem, você aprenderá como instalar uma instância Jenkins com Helm e conectar agentes para executar tarefas de construção.

# Pré-requisitos

Para acompanhar este post, você precisa de um cluster Kubernetes e do cliente Helm.
Todos os principais provedores de nuvem oferecem clusters Kubernetes hospedados:

- AWS tem EKS
- O Azure tem AKS
- O Google Cloud tem GKE

Se você deseja executar um cluster Kubernetes de desenvolvimento em seu PC local, `kind` permite criar e destruir clusters para teste. A postagem `Criando clusters Kubernetes de teste com Kind` fornece instruções sobre como executar o Kubernetes localmente.

Você também deve ter o cliente Helm instalado. A `documentação do Helm` fornece instruções de instalação.

# Adicionando o repositório de gráficos do Jenkins

Os gráficos Jenkins Helm são fornecidos em <a href="https://charts.jenkins.io" rel="nofollow">https://charts.jenkins.io</a>. To make this chart repository available, run the following commands:</p>

<pre><code class="language-bash">helm repo add jenkins https://charts.jenkins.io
helm repo update
</code></pre>

# Como implantar uma instância simples do Jenkins

Para implantar uma instância do Jenkins com as configurações padrão, execute o comando:

<pre><code class="language-bash">helm upgrade --install myjenkins jenkins/jenkins
</code></pre>
<p>O comando <code>helm upgrade</code> é normalmente usado para atualizar uma versão existente. No entanto, o argumento <code>--install</code> garante que a versão seja criada se ela não existir. Isso significa que <code>helm upgrade --install</code> cria <em>e</em> atualiza uma versão, eliminando a necessidade de manipular comandos de instalação e atualização, dependendo se a versão existe ou não.</p>
<p>O nome da versão é <code>myjenkins</code>, e o argumento final <code>jenkins/jenkins</code> define o gráfico a ser instalado.</p>
<p>A saída é mais ou menos assim:</p>
<pre><code class="language-bash">$ upgrade do helm --install myjenkins jenkins/jenkins

Release &quot;myjenkins&quot; does not exist. Installing it now.
NAME: myjenkins
LAST DEPLOYED: Tue Oct 19 08:13:11 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:

1. Get your 'admin' user password by running:
   kubectl exec --namespace default -it svc/myjenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password &amp;&amp; echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
   echo http://127.0.0.1:8080
   kubectl --namespace default port-forward svc/myjenkins 8080:8080

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/

NOTE: Consider using a custom image with pre-installed plugins
</code></pre>

<p>O primeiro comando listado nas notas retorna a senha para o usuário <code>admin</code>:</p>
<pre><code class="language-bash">$ kubectl exec --namespace default -it svc/myjenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password &amp;&amp; eco
</code></pre>
<p>O segundo comando listado nas notas estabelece um túnel para o serviço no cluster Kubernetes.</p>
<p>No Kubernetes, um serviço é um recurso que configura a rede do cluster para expor um ou mais pods. O tipo de serviço padrão é <code>ClusterIP</code>, que expõe os pods por meio de um endereço IP privado. É esse endereço de IP privado que encapsulamos para obter acesso à IU da Web do Jenkins.</p>
<p>Um pod do Kubernetes é um recurso que hospeda um ou mais contêineres. Isso significa que a instância do Jenkins está sendo executada como um contêiner dentro de um pod:</p>
<pre><code class="language-bash">$ kubectl --namespace default port-forward svc/myjenkins 8080:8080
Encaminhamento de 127.0.0.1:8080 -&gt; 8080
Encaminhando de [::1]:8080 -&gt; 8080
</code></pre>
<p>Após o túnel ser estabelecido, abra <a href="http://localhost:8080" rel="nofollow">http://localhost:8080</a> em seu PC local e você será direcionado para a instância Jenkins no cluster Kubernetes. Faça login com o nome de usuário <code>admin</code> e a senha retornada pelo primeiro comando.</p>
<p>Agora você tem uma instância funcional, embora básica, do Jenkins em execução no Kubernetes.</p>
<h2 id="exposing-jenkins-through-a-public-ip-address">Expondo o Jenkins por meio de um endereço IP público</h2>
<p>Acessar o Jenkins por meio de um túnel é útil para depuração, mas não é uma ótima experiência para um servidor de produção. Para acessar o Jenkins por meio de um endereço IP disponível publicamente, você deve substituir a configuração padrão definida no gráfico. Existem centenas de valores que podem ser definidos e a lista completa está disponível executando o comando:</p>
<pre><code class="language-bash">helm show values jenkins/jenkins
</code></pre>
<p>Configurar o serviço que expõe o pod Jenkins como um <code>LoadBalancer</code> é a maneira mais fácil de acessar o Jenkins publicamente.</p>
<p>Um serviço do tipo <code>LoadBalancer</code> expõe pods por meio de um endereço IP público. Exatamente como esse endereço IP público é criado é deixado para o cluster. Por exemplo, plataformas Kubernetes hospedadas como EKS, AKS e GKE criam um balanceador de carga de rede para direcionar o tráfego para o cluster K8s.</p>
<p>Observe que os serviços <code>LoadBalancer</code> requerem configuração adicional ao usar um cluster Kubernetes de teste local, como os clusters criados por tipo. Consulte a <a href="https://kind.sigs.k8s.io/docs/user/loadbalancer/" rel="nofollow">documentação do tipo</a> para obter mais informações.</p>
<p>Para configurar o serviço como um <code>LoadBalancer</code>, crie um arquivo chamado <code>values.yaml</code> com o seguinte conteúdo:</p>
<pre><code class="language-yaml">controlador:
   serviceType: LoadBalancer
</code></pre>
<p>Em seguida, atualize a versão do Helm usando os valores definidos em <code>values.yaml</code> com o comando:</p>
<pre><code class="language-bash">atualização do helm --install -f values.yaml myjenkins jenkins/jenkins
</code></pre>
<p>A saída é alterada sutilmente com a adição de novas instruções para retornar o IP público do serviço:</p>
<pre><code class="language-bash">$ helm upgrade --install -f values.yaml myjenkins jenkins/jenkins
Release "myjenkins" has been upgraded. Happy Helming!
NAME: myjenkins
LAST DEPLOYED: Tue Oct 19 08:45:23 2021
NAMESPACE: default
STATUS: deployed
REVISION: 4
NOTES:

1. Get your 'admin' user password by running:
   kubectl exec --namespace default -it svc/myjenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
   NOTE: It may take a few minutes for the LoadBalancer IP to be available.
   You can watch the status of by running 'kubectl get svc --namespace default -w myjenkins'
   export SERVICE_IP=$(kubectl get svc --namespace default myjenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
  echo http://$SERVICE_IP:8080/login

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/

NOTE: Consider using a custom image with pre-installed plugins

Usando as novas instruções na etapa 2, execute o seguinte comando para obter o endereço IP público ou o nome do host do serviço:

kubectl get svc --namespace default myjenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}"

Implantei o Jenkins em um cluster EKS e este é o resultado do comando para minha infraestrutura:

<pre><code class="language-bash">$ kubectl get svc --namespace default myjenkins --template &quot;{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}&quot;
a84aa6226d6e5496882cfafdd6564a35-901117307.us-west-1.elb.amazonaws.com
</code></pre>

Para acessar o Jenkins, abra http://service_ip_or_hostname:8080 .

Você pode perceber que o Jenkins relata o seguinte erro ao acessá-lo por meio de seu endereço IP público:

It appears that your reverse proxy set up is broken.
<br>
<br>
![](https://i.octopus.com/blog/2022-01/jenkins-helm-install-guide/reverse-proxy-error.png)

Isso pode ser resolvido definindo a URL pública na controller.jenkinsUrlpropriedade, substituindo a84aa6226d6e5496882cfafdd6564a35-901117307.us-west-1.elb.amazonaws.compelo endereço IP ou nome do host da sua instância do Jenkins:

<pre><code class="language-yaml">controller:
  jenkinsUrl: http://a84aa6226d6e5496882cfafdd6564a35-901117307.us-west-1.elb.amazonaws.com:8080/
</code></pre>

## Instalando plug-ins adicionais

Liste todos os plug-ins adicionais a serem instalados no `controller.additionalPlugins` array:

<pre><code class="language-yaml">controller:
    additionalPlugins:
    - octopusdeploy:3.1.6
</code></pre>

O ID e a versão do plug-in são encontrados no `site do plug-in do Jenkins` :
![](https://i.octopus.com/blog/2022-01/jenkins-helm-install-guide/jenkins-plugin.png)

Essa abordagem é conveniente, mas a desvantagem é que a instância do Jenkins precisa entrar em contato com o site de atualização do Jenkins para recuperá-los como parte da primeira inicialização.

Uma abordagem mais robusta é baixar os plug-ins como parte de uma imagem personalizada, o que garante que os plug-ins sejam inseridos na imagem do Docker. Ele também permite que ferramentas adicionais sejam instaladas no controlador Jenkins. A `postagem anterior` contém detalhes sobre como criar e publicar imagens personalizadas do Docker.

Observe que a imagem personalizada do Docker deve ter os seguintes plug-ins instalados, além dos plug-ins personalizados. Esses plug-ins são necessários para o gráfico do Helm funcionar corretamente:

<ul>
<li>kubernetes</li>
<li>workflow-aggregator</li>
<li>git</li>
<li>configuration-as-code</li>
</ul>
Aqui está um exemplo `Dockerfile` incluindo os plugins obrigatórios:

<pre><code class="language-dockerfile">FROM jenkins/jenkins:lts-jdk11
USER root
RUN apt update &amp;&amp; \
    apt install -y --no-install-recommends gnupg curl ca-certificates apt-transport-https &amp;&amp; \
    curl -sSfL https://apt.octopus.com/public.key | apt-key add - &amp;&amp; \
    sh -c &quot;echo deb https://apt.octopus.com/ stable main &gt; /etc/apt/sources.list.d/octopus.com.list&quot; &amp;&amp; \
    apt update &amp;&amp; apt install -y octopuscli
RUN jenkins-plugin-cli --plugins octopusdeploy:3.1.6 kubernetes:1.29.2 workflow-aggregator:2.6 git:4.7.1 configuration-as-code:1.52
USER jenkins
</code></pre>
<p>To use the custom image, you define it in the <code>values.yml</code> with the following properties. This example uses the custom Jenkins image <a href="https://hub.docker.com/r/mcasperson/myjenkins" rel="nofollow">pushed to my DockerHub account</a>:</p>
<pre><code class="language-yaml">controller:
  image: &quot;docker.io/mcasperson/myjenkins&quot;
  tag: &quot;latest&quot;
  installPlugins: false
</code></pre>

Para usar a imagem personalizada, defina-a no `values.yml`com as seguintes propriedades. Este exemplo usa a imagem personalizada do Jenkins `enviada para minha conta do DockerHub` :

<pre><code class="language-yaml">controller:
  JCasC:
    configScripts:
      this-is-where-i-configure-the-executors: |
        jenkins:
          numExecutors: 5
</code></pre>

Você pode encontrar um exemplo Dockerfilede instalação de ferramentas para Java, DotNET Core, PHP, Python, Ruby e Go no `repositório jenkins-complete-image`.

# Configuração do Jenkins como código

Jenkins Configuration as Code (JCasC) é um `plug` -in que fornece um método opinativo para configurar o Jenkins por meio de arquivos YAML. Isso fornece uma alternativa para escrever`scripts Groovy` `referenciando diretamente a API Jenkins` , que é poderosa, mas requer que os administradores se sintam à vontade para escrever código.

JCasC é definido sob a `controller.JCasC.configScript` propriedade. As chaves filhas abaixo `configScript` têm nomes de sua escolha que consistem em letras minúsculas, números e hífens e servem como uma forma de resumir o bloco de texto que definem.

O valor atribuído a essas chaves são strings de várias linhas, que por sua vez definem um arquivo JCasC YAML. O caractere pipe ( |) fornece um método conveniente para definir uma string de várias linhas, mas não é significativo.

O resultado final dá a aparência de um documento YAML contínuo. Lembre-se de que o conteúdo que aparece após o caractere de barra vertical é simplesmente um valor de texto de várias linhas que também é YAML.

O exemplo abaixo configura o número de executores disponíveis para o controlador Jenkins, com o JCasC YAML definido sob uma chave exagerada chamada `this-is-where-i-configure-the-executors` para reforçar o fato de que essas chaves podem ter qualquer nome:<br>

<pre><code class="language-yaml">controller:
  JCasC:
    configScripts:
      this-is-where-i-configure-the-executors: |
        jenkins:
          numExecutors: 5
</code></pre>

Para comparação, a mesma configuração também pode ser obtida com o seguinte script Groovy salvo como `/usr/share/jenkins/ref/init.groovy.d/executors.groovy` uma imagem personalizada do Docker:

import jenkins.model.\*<br>
Jenkins.instance.setNumExecutors(5)
Mesmo este exemplo simples destaca os benefícios do JCasC:

- Cada propriedade JCasC é documentada em http://jenkinshost/configuration-as-code/reference (substitua jenkinshostpelo nome do host de sua própria instância Jenkins), enquanto escrever um script Groovy requer conhecimento da API Jenkins .
- A configuração JCasC é YAML vanilla, que é muito mais acessível do que scripts escritos em Groovy.
- JCasC é opinativo, fornecendo uma abordagem consistente para configuração comum. Os scripts Groovy podem resolver o mesmo problema de várias maneiras, o que significa que scripts com mais do que algumas linhas de código exigem a experiência de um engenheiro de software para serem compreendidos.<p>
  Apesar de todos os benefícios, o JCasC não é um substituto completo para definir as propriedades do sistema ou executar scripts Groovy. Por exemplo, o JCasC não suporta a capacidade de desabilitar o CSRF , o que significa que essa opção é exposta apenas por meio das propriedades do sistema.

# Fazendo backup de volumes Jenkins

Os volumes no Kubernetes são um pouco mais complicados do que os encontrados no Docker normal porque os volumes do K8s tendem a ser hospedados fora do nó que executa o pod. Isso ocorre porque os pods podem ser realocados entre nós e, portanto, precisam acessar volumes de qualquer nó.

Para complicar, ao contrário dos volumes do Docker, apenas os volumes especializados do Kubernetes podem ser compartilhados entre os pods. Esses volumes compartilhados são chamados de ReadWriteManyvolumes. Normalmente, porém, um volume do Kubernetes é usado apenas por um único pod e é chamado de `ReadWriteOnce` volumes.

O gráfico Jenkins Helm configura um `ReadWriteOnce` volume para hospedar o diretório inicial do Jenkins. Como esse volume só pode ser acessado pelo pod no qual está montado, todas as operações de backup devem ser executadas por esse pod.

Felizmente, o gráfico Helm oferece `opções abrangentes de backup` , com a capacidade de realizar backups e salvá-los em provedores de armazenamento em nuvem`

No entanto, você pode orquestrar backups simples e independentes da nuvem com dois comandos.

O primeiro comando é executado tardentro do pod para fazer backup do `/var/jenkins_hom` e diretório no `/tmp/backup.tar.gz` arquivo. Observe que o nome do pod `myjenkins-0` é derivado do nome da versão do Helm myjenkins:

> kubectl `exec` -c jenkins myjenkins-0 -- tar czf /tmp/backup.tar.gz /var/jenkins_home<br>
> O segundo comando copia o arquivo de backup do pod para sua máquina local:

> kubectl cp -c jenkins myjenkins-0:/tmp/backup.tar.gz ./backup.tar.gz<br>
> Neste ponto, backup.tar.gzpode ser copiado para um local mais permanente.

# Adicionando agentes Jenkins

Além de instalar o Jenkins em um cluster Kubernetes, você também pode criar agentes Jenkins dinamicamente no cluster. Esses agentes são criados quando novas tarefas são agendadas no Jenkins e são automaticamente limpos após a conclusão das tarefas.

As configurações padrão para agentes são definidas na `agent` propriedade no `values.yaml` arquivo. O exemplo abaixo define um agente com o rótulo Jenkins `default`, criado em pods prefixados com o nome `default`, e com limites de CPU e memória:

<pre><code class="language-yaml">agent:
  podName: default
  customJenkinsLabels: default
  resources:
    limits:
      cpu: &quot;1&quot;
      memory: &quot;2048Mi&quot;
</code></pre>

Agentes mais especializados são definidos sob a `additionalAgents` propriedade. Esses modelos de pod herdam os valores daqueles definidos na `agent` propriedade.

O exemplo abaixo define um segundo modelo de pod alterando o nome do pod e os rótulos Jenkins para `maven` e especificando uma nova imagem do Docker `jenkins/jnlp-agent-maven:latest`:

<pre><code class="language-yaml">agent:
  podName: default
  customJenkinsLabels: default
  resources:
    limits:
      cpu: &quot;1&quot;
      memory: &quot;2048Mi&quot;
additionalAgents:
  maven:
    podName: maven
    customJenkinsLabels: maven
    image: jenkins/jnlp-agent-maven
    tag: latest
</code></pre>

Para encontrar as definições do agente, navegue até Manage Jenkins , Manage Nodes and Clouds e, finalmente, Configure Clouds .

![](https://i.octopus.com/blog/2022-01/jenkins-helm-install-guide/k8s-cloud.png)

<p>Para usar os agentes para executar um pipeline, defina o, <code>agent</code> block like this:</p>
<pre><code class="language-groovy">pipeline {
  agent {
      kubernetes {
          inheritFrom 'maven'
      }
  }
  // ...
}
</code></pre>
<p>Por exemplo, este pipeline para um aplicativo Java usa o <code>maven</code> agent template:</p>

<pre><code class="language-groovy">pipeline {
  // This pipeline requires the following plugins:
  // * Pipeline Utility Steps Plugin: https://wiki.jenkins.io/display/JENKINS/Pipeline+Utility+Steps+Plugin
  // * Git: https://plugins.jenkins.io/git/
  // * Workflow Aggregator: https://plugins.jenkins.io/workflow-aggregator/
  // * Octopus Deploy: https://plugins.jenkins.io/octopusdeploy/
  // * JUnit: https://plugins.jenkins.io/junit/
  // * Maven Integration: https://plugins.jenkins.io/maven-plugin/
  parameters {
    string(defaultValue: 'Spaces-1', description: '', name: 'SpaceId', trim: true)
    string(defaultValue: 'SampleMavenProject-SpringBoot', description: '', name: 'ProjectName', trim: true)
    string(defaultValue: 'Dev', description: '', name: 'EnvironmentName', trim: true)
    string(defaultValue: 'Octopus', description: '', name: 'ServerId', trim: true)
  }
  tools {
    jdk 'Java'
  }
  agent {
      kubernetes {
          inheritFrom 'maven'
      }
  }
  stages {
    stage('Environment') {
      steps {
          echo &quot;PATH = ${PATH}&quot;
      }
    }
    stage('Checkout') {
      steps {
        // If this pipeline is saved as a Jenkinsfile in a git repo, the checkout stage can be deleted as
        // Jenkins will check out the code for you.
        script {
            /*
              This is from the Jenkins &quot;Global Variable Reference&quot; documentation:
              SCM-specific variables such as GIT_COMMIT are not automatically defined as environment variables; rather you can use the return value of the checkout step.
            */
            def checkoutVars = checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[url: 'https://github.com/mcasperson/SampleMavenProject-SpringBoot.git']]])
            env.GIT_URL = checkoutVars.GIT_URL
            env.GIT_COMMIT = checkoutVars.GIT_COMMIT
            env.GIT_BRANCH = checkoutVars.GIT_BRANCH
        }
      }
    }
    stage('Dependencies') {
      steps {
        // Download the dependencies and plugins before we attempt to do any further actions
        sh(script: './mvnw --batch-mode dependency:resolve-plugins dependency:go-offline')
        // Save the dependencies that went into this build into an artifact. This allows you to review any builds for vulnerabilities later on.
        sh(script: './mvnw --batch-mode dependency:tree &gt; dependencies.txt')
        archiveArtifacts(artifacts: 'dependencies.txt', fingerprint: true)
        // List any dependency updates.
        sh(script: './mvnw --batch-mode versions:display-dependency-updates &gt; dependencieupdates.txt')
        archiveArtifacts(artifacts: 'dependencieupdates.txt', fingerprint: true)
      }
    }
    stage('Build') {
      steps {
        // Set the build number on the generated artifact.
        sh '''
          ./mvnw --batch-mode build-helper:parse-version versions:set \
          -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.minorVersion}.\\${parsedVersion.incrementalVersion}.${BUILD_NUMBER}
        '''
        sh(script: './mvnw --batch-mode clean compile', returnStdout: true)
        script {
            env.VERSION_SEMVER = sh (script: './mvnw -q -Dexec.executable=echo -Dexec.args=\'${project.version}\' --non-recursive exec:exec', returnStdout: true)
            env.VERSION_SEMVER = env.VERSION_SEMVER.trim()
        }
      }
    }
    stage('Test') {
      steps {
        sh(script: './mvnw --batch-mode -Dmaven.test.failure.ignore=true test')
        junit(testResults: 'target/surefire-reports/*.xml', allowEmptyResults : true)
      }
    }
    stage('Package') {
      steps {
        sh(script: './mvnw --batch-mode package -DskipTests')
      }
    }
    stage('Repackage') {
      steps {
        // This scans through the build tool output directory and find the largest file, which we assume is the artifact that was intended to be deployed.
        // The path to this file is saved in and environment variable called JAVA_ARTIFACT, which can be consumed by subsequent custom deployment steps.
        script {
            // Find the matching artifacts
            def extensions = ['jar', 'war']
            def files = []
            for(extension in extensions){
                findFiles(glob: 'target/**.' + extension).each{files &lt;&lt; it}
            }
            echo 'Found ' + files.size() + ' potential artifacts'
            // Assume the largest file is the artifact we intend to deploy
            def largestFile = null
            for (i = 0; i &lt; files.size(); ++i) {
                if (largestFile == null || files[i].length &gt; largestFile.length) { 
                    largestFile = files[i]
                }
            }
            if (largestFile != null) {
                env.ORIGINAL_ARTIFACT = largestFile.path
                // Create a filename based on the repository name, the new version, and the original file extension. 
                env.ARTIFACTS = &quot;SampleMavenProject-SpringBoot.&quot; + env.VERSION_SEMVER + largestFile.path.substring(largestFile.path.lastIndexOf(&quot;.&quot;), largestFile.path.length())
                echo 'Found artifact at ' + largestFile.path
                echo 'This path is available from the ARTIFACTS environment variable.'
            }
        }
        // Octopus requires files to have a specific naming format. So copy the original artifact into a file with the correct name.
        sh(script: 'cp ${ORIGINAL_ARTIFACT} ${ARTIFACTS}')
      }
    }
    stage('Deployment') {
      steps {
        octopusPushPackage(additionalArgs: '', packagePaths: env.ARTIFACTS.split(&quot;:&quot;).join(&quot;\n&quot;), overwriteMode: 'OverwriteExisting', serverId: params.ServerId, spaceId: params.SpaceId, toolId: 'Default')
        octopusPushBuildInformation(additionalArgs: '', commentParser: 'GitHub', overwriteMode: 'OverwriteExisting', packageId: env.ARTIFACTS.split(&quot;:&quot;)[0].substring(env.ARTIFACTS.split(&quot;:&quot;)[0].lastIndexOf(&quot;/&quot;) + 1, env.ARTIFACTS.split(&quot;:&quot;)[0].length()).replace(&quot;.&quot; + env.VERSION_SEMVER + &quot;.zip&quot;, &quot;&quot;), packageVersion: env.VERSION_SEMVER, serverId: params.ServerId, spaceId: params.SpaceId, toolId: 'Default', verboseLogging: false, gitUrl: env.GIT_URL, gitCommit: env.GIT_COMMIT, gitBranch: env.GIT_BRANCH)
        octopusCreateRelease(additionalArgs: '', cancelOnTimeout: false, channel: '', defaultPackageVersion: '', deployThisRelease: false, deploymentTimeout: '', environment: params.EnvironmentName, jenkinsUrlLinkback: false, project: params.ProjectName, releaseNotes: false, releaseNotesFile: '', releaseVersion: env.VERSION_SEMVER, serverId: params.ServerId, spaceId: params.SpaceId, tenant: '', tenantTag: '', toolId: 'Default', verboseLogging: false, waitForDeployment: false)
        octopusDeployRelease(cancelOnTimeout: false, deploymentTimeout: '', environment: params.EnvironmentName, project: params.ProjectName, releaseVersion: env.VERSION_SEMVER, serverId: params.ServerId, spaceId: params.SpaceId, tenant: '', tenantTag: '', toolId: 'Default', variables: '', verboseLogging: false, waitForDeployment: true)
      }
    }
  }
}
</code></pre>
<p>Você pode confirmar que o agente foi criado no cluster durante a execução da tarefa executando:</p>
<pre><code class="language-bash">kubectl get pods
</code></pre>
<p>No exemplo abaixo, o pod <code>java-9-k0hmj-vcvdz-wknh4</code> está em processo de criação para executar o exemplo de pipeline acima:</p>
<pre><code class="language-bash">$ kubectl get pods
NAME                                     READY   STATUS              RESTARTS   AGE
java-9-k0hmj-vcvdz-wknh4                 0/1     ContainerCreating   0          1s
myjenkins-0                              2/2     Running             0          49m
</code></pre>
<h2 id="conclusion">Conclusão</h2>
<p>
Hospedar Jenkins e seus agentes em um cluster Kubernetes permite que você crie uma plataforma de construção escalável e responsiva, criando e destruindo agentes em tempo real para lidar com cargas de trabalho elásticas. E graças ao gráfico Jenkins Helm, instalar Jenkins e configurar os nós requer apenas algumas linhas de YAML.

Neste post você aprendeu como:</p>

<p>Nesta postagem você aprendeu como:</p>
<ul>
<li>Implantar Jenkins no Kubernetes</li>
<li>Expor Jenkins em um endereço IP público</li>
<li>Instalar plug-ins adicionais como parte do processo de instalação</li>
<li>Configurar o Jenkins por meio do JCasC</li>
<li>Faça backup do diretório inicial do Jenkins</li>
<li>Crie agentes do Kubernetes que são criados e destruídos conforme necessário</li>
</ul>
<p>Confira nossos outros posts sobre a instalação do Jenkins:</p>
<ul>
<li><a href="https://octopus.com/blog/jenkins-install-guide-windows-linux">How to install Jenkins on Windows and Linux</a></li>
<li><a href="https://octopus.com/blog/jenkins-docker-install-guide">How to install Jenkins on Docker</a></li>
</ul>
