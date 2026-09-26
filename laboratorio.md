# Laboratorio: Jenkins Controller + Remote Agent

## Objetivo

Configurar una arquitectura Jenkins con un Jenkins Controller en una EC2 y un Jenkins Agent en una segunda EC2, comunicados mediante SSH.

Arquitectura:

~~~text
                         AWS
                          |
             +------------+------------+
             |                         |
             v                         v
       EC2 Controller             EC2 Agent
             |                         |
       Docker + Jenkins          Linux + Java
             |                    Git + Maven
             |                         |
             +--------- SSH ------------+
                          |
                    Jenkins Agent
~~~

El Controller coordina los trabajos y el Agent ejecuta los pasos del pipeline. No se instala Jenkins en el Agent.

## 1. Requisitos

- Segunda instancia EC2.
- Linux compatible.
- Acceso SSH desde el Controller hacia el Agent.
- Java instalado en el Agent.
- Git instalado.
- Maven instalado.
- Security Group permitiendo TCP/22 desde el Controller.

## 2. Crear la segunda EC2

Ejemplo:

~~~text
Name: jenkins-agent-java
OS: Amazon Linux 2023
Tipo: t3.small
~~~

El primer servidor continúa ejecutando tu Jenkins Controller en Docker. El segundo servidor será únicamente el Agent.

## 3. Configurar Security Group

En el Security Group del Agent agrega:

~~~text
Type: SSH
Protocol: TCP
Port: 22
Source: Security Group del Jenkins Controller
~~~

Para el laboratorio evita abrir SSH a 0.0.0.0/0 si puedes restringirlo al Security Group o IP del Controller.

## 4. Preparar el Agent

Conéctate a la segunda EC2.

~~~bash
ssh -i tu-clave.pem ec2-user@IP_DEL_AGENT
sudo dnf update -y
~~~

En Ubuntu usa apt en lugar de dnf.

## 5. Instalar Java

En Amazon Linux 2023:

~~~bash
sudo dnf install -y java-17-amazon-corretto
java -version
~~~

El Agent necesita una JVM compatible con la versión de Jenkins utilizada por el Controller.

## 6. Instalar Git y Maven

~~~bash
sudo dnf install -y git maven
git --version
mvn -version
~~~

En Ubuntu:

~~~bash
sudo apt update
sudo apt install -y git maven
~~~

## 7. Crear el usuario Jenkins

~~~bash
sudo useradd --create-home --shell /bin/bash jenkins
sudo mkdir -p /home/jenkins/agent
sudo chown -R jenkins:jenkins /home/jenkins
id jenkins
~~~

Usaremos /home/jenkins/agent como Remote root directory.

## 8. Configurar SSH

Desde el servidor donde está el Controller genera una clave dedicada:

~~~bash
ssh-keygen -t ed25519 -C jenkins-agent
~~~

Guárdala como:

~~~text
~/.ssh/jenkins_agent
~/.ssh/jenkins_agent.pub
~~~

Para un laboratorio puedes usar una passphrase vacía. En producción utiliza la política de gestión de secretos de tu organización.

Autoriza la clave pública en el Agent. Si ssh-copy-id está disponible:

~~~bash
ssh-copy-id -i ~/.ssh/jenkins_agent.pub jenkins@IP_DEL_AGENT
~~~

Si no está disponible, agrega manualmente el contenido de la clave pública a /home/jenkins/.ssh/authorized_keys.

En el Agent:

~~~bash
sudo mkdir -p /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh
sudo chmod 600 /home/jenkins/.ssh/authorized_keys
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
~~~

## 9. Probar SSH antes de configurar Jenkins

Desde el Controller:

~~~bash
ssh -i ~/.ssh/jenkins_agent jenkins@IP_DEL_AGENT
hostname
java -version
git --version
mvn -version
~~~

Si esta conexión no funciona, corrige SSH antes de continuar con Jenkins.

## 10. Crear la credencial SSH en Jenkins

En Jenkins:

~~~text
Manage Jenkins
  -> Credentials
  -> System
  -> Global credentials
  -> Add Credentials
~~~

Configura:

~~~text
Kind: SSH Username with private key
Username: jenkins
ID: jenkins-agent-ssh
Private Key: contenido de ~/.ssh/jenkins_agent
~~~

No guardes la clave privada en Git.

## 11. Crear el Node / Agent

Ve a:

~~~text
Manage Jenkins
  -> Nodes
  -> New Node
~~~

Configura:

~~~text
Name: java-agent-01
Type: Permanent Agent
Executors: 1
Remote root directory: /home/jenkins/agent
Labels: java-agent maven linux
~~~

Para Usage puedes seleccionar que el nodo se use solamente con jobs cuyo label coincida.

Launch method:

~~~text
Launch agents via SSH
Host: IP privada del Agent
Credentials: jenkins-agent-ssh
~~~

Para Host Key Verification, usa una estrategia segura en producción. Para un laboratorio puedes utilizar temporalmente Non verifying Verification Strategy, entendiendo que desactiva la verificación de la identidad del host.

## 12. Verificar el Agent

El Node java-agent-01 debe aparecer Online.

En los logs del node busca mensajes equivalentes a:

~~~text
Authentication successful
Starting agent process
Agent successfully connected
~~~

## 13. Primer Pipeline ejecutado en el Agent

~~~groovy
pipeline {
    agent {
        label 'java-agent'
    }

    stages {
        stage('Identify Agent') {
            steps {
                sh '''
                    echo "Hostname:"
                    hostname
                    echo "Java:"
                    java -version
                    echo "Git:"
                    git --version
                    echo "Maven:"
                    mvn -version
                '''
            }
        }

        stage('Build') {
            steps {
                git branch: 'main', url: 'https://github.com/aldo2510/ec-maven-users-api.git'
                sh 'mvn -B -ntp clean package'
            }
        }
    }
}
~~~

El hostname debe corresponder al segundo servidor. Maven se ejecuta en el Agent, no en el Controller.

## 14. Entender los labels

El Agent tiene los labels:

~~~text
java-agent maven linux
~~~

Puedes seleccionar el Agent con:

~~~groovy
agent { label 'java-agent' }
~~~

o con una expresión:

~~~groovy
agent { label 'linux && maven' }
~~~

Los labels permiten que Jenkins seleccione automáticamente un nodo con las capacidades requeridas.

## 15. Integración con la Shared Library

Shared Library:

https://github.com/aldo2510/jenkins_sharedlib

Ejemplo:

~~~groovy
@Library('aldo-jenkins-shared') _

pipeline {
    agent {
        label 'java-agent'
    }

    stages {
        stage('Version') {
            steps {
                script {
                    def version = getPomVersion()
                    echo "Version: ${version}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} - ${version}"
                }
            }
        }

        stage('Build') {
            steps {
                buildMaven(goals: 'clean package')
            }
        }

        stage('Archive') {
            steps {
                archiveMavenArtifacts()
            }
        }
    }
}
~~~

La idea clave es:

~~~text
WHERE?
  agent { label 'java-agent' }
       -> Jenkins Agent

HOW?
  getPomVersion()
  buildMaven()
  archiveMavenArtifacts()
       -> Shared Library
~~~

## 16. Pipeline multi-agent

Como ejercicio avanzado, crea otro Agent, por ejemplo docker-agent.

~~~groovy
pipeline {
    agent none

    stages {
        stage('Build Java') {
            agent { label 'java-agent' }
            steps {
                sh 'mvn -version'
            }
        }

        stage('Docker') {
            agent { label 'docker-agent' }
            steps {
                sh 'docker version'
            }
        }
    }
}
~~~

Arquitectura:

~~~text
                   Controller
                       |
             +---------+---------+
             |                   |
             v                   v
        java-agent          docker-agent
             |                   |
           Maven               Docker
~~~

## 17. Ejercicio final

Construye un pipeline que:

1. Ejecute en java-agent.
2. Haga checkout de ec-maven-users-api.
3. Obtenga la versión del pom.xml.
4. Ejecute buildMaven con clean package.
5. Archive el JAR.
6. Muestre el hostname del Agent.
7. Muestre Java y Maven.
8. Muestre la versión de la aplicación en el nombre del build.

Resultado esperado:

~~~text
#15 - 1.0.0
Agent hostname: jenkins-agent-java
Java: 17.x
Maven: 3.x
Application version: 1.0.0
BUILD SUCCESS
~~~

## 18. Troubleshooting

### Agent offline

Revisa Security Group, TCP/22, IP privada, credenciales SSH, Java, usuario jenkins y permisos de /home/jenkins.

### Permission denied (publickey)

~~~bash
ssh -i ~/.ssh/jenkins_agent jenkins@IP_DEL_AGENT
~~~

Revisa /home/jenkins/.ssh/authorized_keys y sus permisos.

### Java no encontrado

~~~bash
java -version
which java
~~~

### Maven no encontrado

~~~bash
mvn -version
which mvn
~~~

### El pipeline queda esperando

Revisa Build Executor Status y confirma que exista un executor disponible en el Agent.

## 19. Preguntas para los alumnos

1. ¿Por qué no instalamos Jenkins en el Agent?
2. ¿Qué función cumple el Controller?
3. ¿Qué función cumple el Agent?
4. ¿Qué protocolo utilizamos para conectar ambos?
5. ¿Qué es un label?
6. ¿Qué sucede si un Agent está offline?
7. ¿Dónde se ejecuta mvn clean package?
8. ¿Puede un pipeline utilizar diferentes Agents?
9. ¿Qué ventajas tiene separar Controller y Agents?
10. ¿Qué diferencia existe entre un Agent permanente y uno efímero?

## 20. Concepto final

~~~text
                    JENKINS
                       |
                 CONTROLLER
                       |
                Orquestación
                       |
             +---------+---------+
             |                   |
             v                   v
        JAVA AGENT          DOCKER AGENT
             |                   |
           Maven               Docker
           Java                 Build
           Git                  Push
~~~

Encima de esta infraestructura podemos construir una plataforma con Controllers, Agents/Nodes y Shared Libraries. La meta de Platform Engineering es que los equipos consumidores no tengan que reinventar la implementación común en cada Jenkinsfile.