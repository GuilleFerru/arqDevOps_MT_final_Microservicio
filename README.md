
# Aplicación Tipo Microservicio

Este repositorio contiene una aplicación basada en microservicios, con un pipeline automatizado en **Jenkins** que utiliza **Maven** para la construcción, pruebas y verificación de código, **SonarQube** para el análisis de calidad de código, y **Docker** para la creación y publicación de imágenes en DockerHub.

## Descripción General

El pipeline de Jenkins definido en el archivo `Jenkinsfile` sigue un enfoque de integración continua (CI), automatizando la construcción, análisis, verificación y publicación de la aplicación en un entorno Docker.

##  Infraestructura en AWS

Se ha creado una infraestructura automatizada en AWS utilizando **Terraform**. Esta infraestructura incluye una instancia EC2 sobre la cual se desplegaron varios servicios clave mediante **Ansible**:

- **Jenkins**: Para la gestión de pipelines de integración continua.
- **SonarQube**: Para el análisis de calidad de código y la implementación de reglas de control de calidad.

Puedes acceder al repositorio con la configuración de la infraestructura y su código fuente en este [enlace](https://github.com/GuilleFerru/arqDevOps_MT_final_aws.git).

## Estructura del Pipeline

El pipeline se compone de las siguientes etapas:

1. **Validate**:
   - Ejecuta `mvn validate` para comprobar que el proyecto está correctamente configurado.

2. **Compile**:
   - Compila el código fuente utilizando `mvn compile`.

3. **Test**:
   - Ejecuta las pruebas unitarias con `mvn clean test` y genera reportes de pruebas (`JUnit`) y cobertura de código (`Jacoco`).

4. **Verify**:
   - Verifica el proyecto utilizando `mvn verify` para asegurarse de que cumple con los requisitos del build.

5. **Package**:
   - Genera el artefacto empaquetado (`JAR`) usando `mvn package` sin ejecutar las pruebas.

6. **SonarQube**:
   - Analiza la calidad del código utilizando **SonarQube** y reporta los resultados a la rama correspondiente.

   ![Sonar Qube Result](./images/SonarQube_result.PNG)

7. **Quality Gate**:
   - Espera el resultado del **Quality Gate** de SonarQube. Si no se cumplen las condiciones de calidad (como una cobertura de código superior al 60%), el pipeline se aborta.

   ![Quality Gate Conditions](./images/QualityGate_conditions.PNG)

8. **Docker Image**:
   - Construye y publica una imagen Docker en **DockerHub**, utilizando `buildx` para soportar múltiples plataformas (linux/amd64 y linux/arm64).
   - La imagen es etiquetada tanto con la versión del artefacto como con `latest`.

### Configuración de Jenkins

El pipeline utiliza los siguientes plugins de Jenkins:

- **JUnit**: Para la generación de reportes de pruebas.
- **Jacoco**: Para la cobertura de código.
- **SonarQube**: Para el análisis de calidad de código.
- **Docker Pipeline**: Para la construcción y publicación de imágenes Docker.

![Jenkins Pipeline Result](./images/JenkinsPipeline_result.PNG)

### Ejecución del Pipeline

Para ejecutar este pipeline, asegúrate de tener configurados los siguientes elementos en Jenkins:

- **Credenciales de DockerHub**: Configura las credenciales con el ID `dockerhub-credentials`.
- **SonarQube**: Configura un servidor SonarQube y asegúrate de que el pipeline esté conectado con el mismo mediante el ID `sonarqube`.

![Jenkins Credentials](./images/Jenkins_credentials.PNG)

### Publicación de la Imagen Docker

El pipeline crea y publica una imagen Docker en **DockerHub**. La imagen es etiquetada con la versión del artefacto y `latest`:

![DockerHub Result](./images/DockerHub_result.PNG)

```bash
docker pull guilleferru/<artifactId>:<version>
docker pull guilleferru/<artifactId>:latest
```

```groovy
pipeline {
    agent none
    stages {
        stage('Maven Stages') {
            agent {
                docker {
                    image 'maven:3.8.8-eclipse-temurin-17-alpine'
                }
            }
            stages {
                stage('Validate') {
                    steps {
                        sh 'mvn validate -B -ntp'
                    }
                }

                stage('Compile') {
                    steps {
                        sh 'mvn compile -B -ntp'
                    }
                }

                stage('Test') {
                    steps {
                        sh 'mvn clean test -B -ntp'
                    }
                    post {
                        success {
                            junit 'target/surefire-reports/*.xml'
                            jacoco(execPattern: 'target/jacoco.exec')
                        }
                    }
                }

                stage('Verify') {
                    steps {
                        sh 'mvn verify -B -ntp'
                    }
                }

                stage('Package') {
                    steps {
                        sh 'mvn package -DskipTests -B -ntp'
                    }
                }

                stage('SonarQube') {
                    steps {
                        script {
                            def branch = env.GIT_BRANCH?.split('/')?.last() ?: 'master'
                            echo "branch: ${branch}"
                            withSonarQubeEnv('sonarqube') {
                                sh "mvn sonar:sonar -Dsonar.branch.name=${branch} -B -ntp"
                            }
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Image') {
            agent any
            options { skipDefaultCheckout() }
            steps {
                sh 'docker --version'
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    sh 'docker buildx version'
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        sh """
                            docker buildx build \
                                -t guilleferru/${pom.artifactId}:${pom.version} \
                                -t guilleferru/${pom.artifactId}:latest \
                                --platform linux/amd64,linux/arm64 --push .
                        """
                    }
                }
            }
        }
    }
}
```