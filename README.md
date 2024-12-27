### Cómo Implementar Microservicios en Paralelo con Jenkins 

### **Introducción**

En un entorno que exige agilidad y eficiencia en el desarrollo de software, la implementación de microservicios plantea un desafío clave para los equipos DevOps. Herramientas de código abierto como **Jenkins** ofrecen una solución práctica y económica frente a alternativas comerciales, permitiendo automatizar procesos críticos mientras conservan la flexibilidad característica del open source. 

Cuando se integra con **Helm Charts**, Jenkins optimiza la gestión y el despliegue simultáneo de microservicios en clústeres **GKE**, mejorando la eficiencia y minimizando errores manuales.

Este enfoque está alineado con las métricas clave definidas en el informe **[DORA (Accelerate State of DevOps)](https://services.google.com/fh/files/misc/2024_final_dora_report.pdf)**, que valida cuatro indicadores fundamentales para medir el éxito en la entrega de software:

- **Tiempo de ejecución del cambio:** mide cuánto tarda un cambio de código en implementarse exitosamente en producción.  
- **Frecuencia de implementación:** refleja la frecuencia con la que se realizan despliegues en producción.  
- **Tasa de errores de cambio:** indica el porcentaje de implementaciones que generan fallos y requieren correcciones.  
- **Tiempo de recuperación de una implementación fallida:** evalúa la rapidez con que se resuelven los fallos de despliegue.

La integración de Jenkins y Helm Charts contribuye directamente a mejorar estos indicadores, acelerando los ciclos de despliegue y fortaleciendo la resiliencia y el rendimiento de los equipos DevOps.

### **Desarrollo**

El enfoque GitOps permite gestionar configuraciones y despliegues de forma eficiente mediante un repositorio centralizado. Por ejemplo, al unificar las configuraciones de todos los microservicios en un único lugar, puedes realizar ajustes como actualizar el tag de la imagen de un servicio directamente desde este repositorio. Cuando se efectúa un cambio (por ejemplo, en GitHub, Gitlab , Bitbucket), herramientas como Jenkins lo detectan automáticamente, activando el pipeline correspondiente. Esto garantiza que solo los servicios con modificaciones sean actualizados, asegurando un flujo de trabajo automatizado, organizado y alineado con las mejores prácticas de DevOps. Este artículo se enfoca exclusivamente en el despliegue de microservicios en Google Kubernetes Engine (GKE), utilizando el mismo clúster configurado con Jenkins para administrar los pipelines. Dichos pipelines abarcan todo el proceso, desde la construcción de la aplicación hasta el almacenamiento de imágenes en Artifact Registry de Google Cloud.

### Helm

Un Helm Chart es un paquete preconfigurado que facilita la gestión de aplicaciones y recursos en Kubernetes. Permite definir, instalar y actualizar aplicaciones de forma organizada y repetible, utilizando archivos YAML para simplificar el despliegue, manejo de dependencias y actualizaciones, garantizando consistencia en el clúster.

<div style="text-align: center;">
  <img src="https://helm.sh/img/helm.svg" alt="Helm Logo" style="max-width: 100%; height: auto;">
</div>

### Google Kubernetes Engine (GKE)

Google Kubernetes Engine (GKE) es un servicio gestionado que facilita el despliegue y escalado de aplicaciones en contenedores. Al integrar Jenkins y Helm, se optimiza el despliegue de microservicios en paralelo: Jenkins automatiza el proceso de construcción y despliegue, mientras que Helm simplifica la gestión de configuraciones en GKE, mejorando la eficiencia y reduciendo errores en el flujo CI/CD.

<div style="text-align: center;">
  <img src="/asset/logo_GKE.png" alt="GKE Logo" style="max-width: 100%; height: auto;">
</div>

### Jenkins 

Jenkins es una herramienta de integración y entrega continua (CI/CD) de código abierto que automatiza la construcción, prueba y despliegue de aplicaciones. Facilita la integración frecuente de cambios en el código, detectando problemas temprano y mejorando la eficiencia y calidad del software. Su amplia gama de plugins lo hace altamente extensible, permitiendo integración con diversas herramientas y tecnologías en el ecosistema DevOps.

<div style="text-align: center;">
  <img src="https://www.jenkins.io/images/logos/jenkins/jenkins.svg" alt="Jenkins Logo" style="max-width: 100%; height: auto;">
</div>

### Preparación

Para lograr una implementación exitosa de microservicios en un clúster de Google Kubernetes Engine (GKE), es fundamental cumplir con varios requisitos previos. Primero, se debe contar con un clúster GKE configurado y en funcionamiento. Luego, es necesario instalar Jenkins en este clúster para automatizar los procesos de integración y entrega continua (CI/CD). A continuación, se debe configurar Jenkins para trabajar de forma eficiente con Kubernetes, incluyendo la configuración de nodos "slaves" encargados de ejecutar las cargas de trabajo. Para optimizar la ejecución de estos nodos, se debe utilizar una imagen Docker preconfigurada con herramientas clave como kubectl, Helm, google-cloud-sdk y Snyk, la cual servirá como base para el pod template en GKE, asegurando que los nodos "slaves" tengan todo lo necesario para interactuar con Kubernetes y Google Cloud.

```shell
    ## ./config-jenkins/Dockerfile
    FROM ubuntu:20.04

    ENV KUBE_VERSION=1.18.0
    ENV HELM_VERSION=3.5.4
    ENV SNYK_VERSION=v1.662.0


    ENV DEBIAN_FRONTEND=noninteractive

    RUN apt-get update && \
        apt-get install git curl wget gnupg -y

    RUN apt-get update && \
        apt-get install -y curl gnupg && \
        echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] http://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
        curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key --keyring /usr/share/keyrings/cloud.google.gpg  add - && \
        apt-get update -y && \
        apt-get install google-cloud-sdk -y

    RUN wget -q https://github.com/snyk/snyk/releases/download/${SNYK_VERSION}/snyk-linux -O /usr/local/bin/snyk && \
        chmod +x /usr/local/bin/snyk && \
        wget -q https://storage.googleapis.com/kubernetes-release/release/v${KUBE_VERSION}/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl && \
        chmod +x /usr/local/bin/kubectl && \
        wget -q https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz -O - | tar -xzO linux-amd64/helm > /usr/local/bin/helm && \
        chmod +x /usr/local/bin/helm && \
        chmod g+rwx /root && \
        mkdir /config && \
        chmod g+rwx /config && \
        helm repo add "stable" "https://charts.helm.sh/stable" --force-update

    RUN apt-get update && \
        apt-get install -y google-cloud-cli-gke-gcloud-auth-plugin

    WORKDIR /config

    CMD sh
```

Para garantizar la seguridad y eficiencia en la implementación de microservicios, es recomendable almacenar la imágen Docker utilizadas para los pods slaves en un repositorio privado. Proveedores como Artifact  Registry (GAR) ofrecen soluciones robustas para gestionar y proteger estas imágenes.



### Estructura del Repositorio

```shell
    ├───config-jenkins
    │       Dockerfile
    │       Jenkinsfile
    └───microservices
        ├───adservice
        │   │   .helmignore
        │   │   Chart.yaml
        │   │   values-dev.yaml
        │   │
        │   └───templates
        │           deployment.yaml
        │           service.yaml
        │
        ├───cartservice
        │   │   .helmignore
        │   │   Chart.yaml
        │   │   values-dev.yaml
        │   │
        │   └───templates
        │           deployment.yaml
        │           service.yaml
        │          .
        │          .                
```

### A continuación, se explica paso a paso el script de Jenkinsfile:

##### 1. Configuración  Inicial

- Este paso establece las reglas globales del pipeline y el entorno donde se ejecutará. Se define que solo se conservarán los últimos 10 builds para optimizar el espacio en el servidor. Además, se añaden marcas de tiempo a los registros para facilitar la depuración.
El pipeline opera en un nodo etiquetado como jenkins-slave. También se incluye un control para que el usuario pueda forzar la ejecución del pipeline incluso si no hay cambios detectados en los archivos relevantes.

  ```shell
  pipeline {
    options {
        buildDiscarder logRotator(numToKeepStr: '10') 
        timestamps() 
    }
    agent {
        label "jenkins-slave"
    }
    parameters {
        booleanParam(name: 'force', defaultValue: false, description: 'Force the pipeline to proceed regardless of changes')
    }
  ```
##### 2. Stage: Checkout Code Changes 
-  Aquí se verifica si hay modificaciones en los archivos críticos que justifican la ejecución del pipeline. Este paso solo se ejecuta cuando no se detectan cambios en los archivos values-dev.yaml de los microservicios, y el parámetro de fuerza no está activado.
Si no hay cambios significativos, el pipeline se detiene y registra un mensaje indicando que no hay actualizaciones necesarias, evitando ejecuciones innecesarias.
  ```shell
    stage("Checkout Code Changes") {
      when {
          not {
              anyOf {
                  expression { params.force }
                  changeset "microservices/**/values-dev.yaml"
              }
          }
      }
      steps {
          script {
              currentBuild.result = 'ABORTED' // Marca el build como abortado.
              echo "No changes detected in values-dev.yaml, canceling the build"
              echo "Deploy aborted due to no changes detected"
          }
      }
  }
  ```


##### 3. Checkout Commit and Branch

- En este punto, se extraen datos clave del repositorio Git para el build en curso. Se ejecutan comandos para obtener el hash corto del commit actual y el nombre de la rama activa. Estos valores se imprimen en los registros para facilitar el seguimiento y análisis del estado del código fuente.

  ```shell
    stage("Checkout Commit and Branch") {
      when {
          allOf {
              expression { currentBuild.result != "ABORTED" }
          }
      }
      steps {
          script {
              echo "======== Commit and Branch ========"
              env.GIT_COMMIT = sh(returnStdout: true, script: "git rev-parse --short=10 HEAD").trim()
              env.GIT_BRANCH = sh(returnStdout: true, script: "git rev-parse --abbrev-ref HEAD").trim()
              echo "=====> Current Git Commit: ${env.GIT_COMMIT}"
              echo "=====> Current Git Branch: ${env.GIT_BRANCH}"
          }
      }
  }

  ```

##### 4. Stage: Checkout Microservices

- Este paso identifica los microservicios impactados por los cambios en el código y valida que estén listos para el despliegue. Se ejecuta un comando que lista los directorios modificados dentro del repositorio.

   Cada microservicio identificado pasa por una serie de verificaciones:
    - Se asegura de que tenga un archivo `values-dev.yaml`.
    - Se comprueba que este archivo defina una etiqueta de imagen (`image.repository`).

  Si alguno de estos requisitos no se cumple, el pipeline se detiene con un mensaje de error, indicando claramente el problema.


  ```shell
    stage("Checkout Microservices") {
      when {
          expression { currentBuild.result != 'ABORTED' }
      }
      steps {
          script {
              echo "========Checking Microservices ========"
              env.MICROSERVICES = sh(returnStdout: true, script: "git diff-tree ...").trim()
              def microservicesList = env.MICROSERVICES.split('\n')
              if (!env.MICROSERVICES) {
                  error "No microservices detected for deployment."
                  currentBuild.result = 'ABORTED'
              }
              microservicesList.each { microservice -> 
                  if (!fileExists("microservices/${microservice}/values-dev.yaml")) {
                      error "The values-dev.yaml file does not exist for microservice: ${microservice}"
                      currentBuild.result = 'ABORTED'
                  }
                  def yamlContent = readYaml file: "microservices/${microservice}/values-dev.yaml"
                  if (!yamlContent?.image?.repository) {
                      error "The image repository is not defined for microservice: ${microservice} in values-dev.yaml"
                  }
              }
           }
        }
     }

  ```

##### 5. Stage: Deploy to DEV Environment
- Aquí se implementan los microservicios en el entorno de desarrollo. El pipeline se conecta al clúster de Kubernetes mediante gcloud y divide los microservicios en 4 grupos para realizar un despliegue paralelo, lo que optimiza el tiempo total de ejecución.
Se utiliza la función auxiliar deployMicroservice para manejar cada implementación. Una vez completado, se verifica el estado de los pods en Kubernetes para asegurarse de que el despliegue fue exitoso.Los grupos definidos en **deployJobs** se ejecutan en paralelo utilizando el método parallel. Esto permite que los despliegues sean más rápidos al realizarse simultáneamente.

  ```shell
      stage("Deploy to DEV Environment") {
      when {
        expression { currentBuild.result != 'ABORTED' }
      }            
      steps {
        container('gcloud') {
            script {
                echo "======== Deploying to DEV ========"
                sh("gcloud container clusters get-credentials jenkins-master")
                
                def microservicesList = env.MICROSERVICES.split('\n').toList()
                int numGroups = 4
                def groupSize = (int) Math.ceil((microservicesList.size() / (double) numGroups))
                def groupedMicroservices = microservicesList.collate(groupSize)
                def deployJobs = [:]
                groupedMicroservices.eachWithIndex { group, idx -> 
                    def groupName = "Group ${idx + 1} Deployments"
                    deployJobs[groupName] = {
                        group.each { microservice -> deployMicroservice(microservice) }
                    }
                }
                parallel deployJobs
                sh("kubectl -n dev get pods")
              }
          }
        }
      }
  ```

##### 6. Función Auxiliar: deployMicroservice

- Esta función es responsable de desplegar cada microservicio utilizando Helm. Incluye opciones avanzadas como --wait y --atomic para garantizar que los despliegues sean consistentes y seguros.
En caso de error, se capturan detalles del pod afectado y los registros más recientes para facilitar la resolución del problema.

  ```shell
      def deployMicroservice(microservice) {
      try {
        sh """ 
        if ! helm upgrade ${microservice} ./microservices/${microservice} --install ...; then
            kubectl -n dev describe pod -l version=${env.GIT_COMMIT}
            kubectl -n dev logs --tail=2000 -l version=${env.GIT_COMMIT}
            exit 1
        fi
        """
      } catch (Exception e) {
          echo "Error deploying ${microservice}: ${e.getMessage()}"
      }
    }
  ```

#### Jenkinsfile Completo 


  ```shell
pipeline {
    options {
        buildDiscarder logRotator(numToKeepStr: '10')
        timestamps()
    }
    
    agent {
        label "jenkins-slave"
    }
        parameters {
        booleanParam(name: 'force', defaultValue: false, description: 'Force the pipeline to proceed regardless of changes')
    }
    
    stages {
        stage("Checkout Code Changes") {
            when {
                not {
                    anyOf {
                        expression { params.force }
                        changeset "microservices/**/values-dev.yaml"
                    }
                }
            }
            steps {
                script{                    
                       currentBuild.result = 'ABORTED'   
                       echo " No changes detected in values-dev.yaml , canceling the build "
                       echo " Deploy aborted due to no changes detected "
                }
            }
        }
        stage("Checkout Commit and Branch") {
            when {
                allOf {
                    expression { currentBuild.result != "ABORTED" }
                }
            }
            steps {
                script {
                    echo "======== Commit and Branch ========"
                    env.GIT_COMMIT = sh(returnStdout: true, script: "git rev-parse --short=10 HEAD").trim()
                    env.GIT_BRANCH = sh(returnStdout: true, script: "git rev-parse --abbrev-ref HEAD").trim()
                    echo "=====> Current Git Commit: ${env.GIT_COMMIT}"
                    echo "=====> Current Git Branch: ${env.GIT_BRANCH}"
                }
            }
        }
        stage("Checkout Microservices") {
            when{
                expression { currentBuild.result != 'ABORTED' }
            }
            steps {
                script {
                    echo "========Checking Microservices ========"
                    env.MICROSERVICES = sh(returnStdout: true, script: "git diff-tree --no-commit-id --name-only -r HEAD -- microservices | grep -v 'Jenkinsfile' | grep -v 'ingress' | xargs -n1 dirname | xargs -n1 basename | sort -u").trim()
                    def microservicesList = env.MICROSERVICES.split('\n')
                    if (!env.MICROSERVICES) {
                        error "No microservices detected for deployment."
                        currentBuild.result = 'ABORTED'
                    }

                    echo "Microservice(s) to be deployed:"
                    echo "${env.MICROSERVICES}"

                    microservicesList.each { microservice ->
                        if (!fileExists("microservices/${microservice}/values-dev.yaml")) {
                            error "The values-dev.yaml file does not exist for microservice: ${microservice}"
                            currentBuild.result = 'ABORTED'
                        }
                        def yamlContent = readYaml file: "microservices/${microservice}/values-dev.yaml"
                        if (!yamlContent?.image?.repository) {
                            error "The image repository  is not defined for microservice: ${microservice} in values-dev.yaml"
                        } else {
                            echo "Microservice: ${microservice}, Image repository: ${yamlContent.image.repository}"
                        }
                    }
                }
            }
        }

        stage("Deploy to DEV Environment") {
            when{
                expression { currentBuild.result != 'ABORTED' }
            }            
             steps {
                container('gcloud') {
                        script {
                            echo "======== Deploying to DEV ========"
                            sh("gcloud container clusters get-credentials jenkins-master")
                            
                            def microservicesList = env.MICROSERVICES.split('\n').toList()
                                                        
                            int numGroups = 4 
                            def groupSize = (int) Math.ceil((microservicesList.size() / (double) numGroups))
                            
                            def groupedMicroservices = microservicesList.collate(groupSize)

                            def deployJobs = [:]

                            groupedMicroservices.eachWithIndex { group, idx ->
                                def groupName = "Group ${idx + 1} Deployments"
                                deployJobs[groupName] = {
                                    group.each { microservice ->
                                        deployMicroservice(microservice)
      
                              }
                                }
                            }                            
                            parallel deployJobs
                            
                            sh("kubectl -n dev get pods")
                        }
                }
           }
        }
    }
}


def deployMicroservice(microservice) {
    try {
            sh """  if ! helm upgrade ${microservice} ./microservices/${microservice} --install --wait --atomic --timeout=600s -n dev -f ./microservices/${microservice}/values-dev.yaml --set podLabels.version=${env.GIT_COMMIT} --set podLabels.buildID=${env.GIT_COMMIT}-${env.BUILD_NUMBER}; then
                    kubectl -n dev describe pod -l version=${env.GIT_COMMIT}
                    kubectl -n dev logs --tail=2000 -l version=${env.GIT_COMMIT}
                    exit 1
                    fi
                """
    } catch (Exception e) {
        echo "Error deploying ${microservice}: ${e.getMessage()}"
    }
}

  ```

#### Ejemplo del Pipeline Configurado con Ejecución Paralela


Este diagrama ilustra cómo el pipeline divide y ejecuta los trabajos de despliegue en paralelo, mejorando la eficiencia y reduciendo el tiempo total de procesamiento. Cada grupo representa un conjunto de microservicios que se implementan simultáneamente, lo que permite gestionar múltiples despliegues de forma escalable y organizada.

<div style="text-align: center; margin: 20px 0;"> <img src="/asset/paralel.png" alt="Ejecución del Pipeline" style="width: 80%; max-width: 100%; height: auto; border: 1px solid #ddd; border-radius: 8px; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);"> </div>


#### Beneficios

- El uso de esta tecnología ofrece ventajas clave que impactan positivamente tanto en las empresas como en los usuarios finales. A continuación, se detallan algunos de los beneficios más destacados.
  - **Reducción de Costos Operativos:** Al automatizar procesos y eliminar la necesidad de intervención manual o intermediarios, las empresas pueden reducir significativamente sus costos operativos.
  - **Mayor Eficiencia en los Procesos:** La automatización y el uso de herramientas avanzadas permiten mejorar la productividad de los equipos, ya que se enfocan en tareas de mayor valor agregado. Esto se traduce en un aumento de la eficiencia operativa, menor tiempo de respuesta y menos errores humanos.


####  Retos o limitaciones

- Aunque esta tecnología ofrece múltiples beneficios, también enfrenta desafíos que deben considerarse para una implementación efectiva. 
    - **Requerimientos de Infraestructura:** El despliegue paralelo incrementa la carga sobre los recursos de infraestructura, como nodos en el clúster Kubernetes y almacenamiento. Las organizaciones deben garantizar que cuentan con los recursos necesarios para soportar el aumento en la demanda.
    - **Diagnóstico y Resolución de Problemas:** Cuando ocurren fallos en el despliegue paralelo, identificar la causa raíz puede ser desafiante. El análisis de logs y errores se complica debido a la simultaneidad de los procesos.

### **Conclusión**

La implementación paralela de microservicios con Jenkins, respaldada por herramientas como Helm y Kubernetes, representa un paso esencial hacia la optimización del desarrollo y despliegue de aplicaciones modernas. Este enfoque permite a las organizaciones reducir costos operativos, mejorar la eficiencia y garantizar un flujo de trabajo más ágil y confiable .La implementación de este tipo de soluciones no solo es una ventaja competitiva sino también una necesidad en un entorno tecnológico en constante evolución. Es el momento de dar el salto hacia una infraestructura de desarrollo más eficiente y escalable cuando trabajamos con multiples microservicios.


### **Recursos y referencias**

- [Google Cloud Skills Boost: Deploy Jenkins to Kubernetes in Google Cloud](https://www.cloudskillsboost.google/focuses/1776?parent=catalog)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Google Kubernetes Engine (GKE) Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [Install Jenkins in Kubernetes](https://www.jenkins.io/doc/book/installing/kubernetes/)




