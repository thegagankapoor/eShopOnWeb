pipeline {
  agent any

  environment {
    PROJECT     = 'src/Web/Web.csproj'
    PUBLISH_DIR = 'publish'
    IIS_PATH    = 'C:\\inetpub\\eShopOnWeb'
    APP_POOL    = 'eShopOnWeb'
    SITE_NAME   = 'eShopOnWeb'
    PORT        = '75'
    SONAR_URL   = 'http://172.28.203.170:9000'
  }

  stages {
    stage('Checkout') {
      steps {
        echo "📦 Checking out source code..."
        git branch: 'main', url: 'https://github.com/thegagankapoor/eShopOnWeb.git'
      }
    }

    stage('Restore') {
      steps {
        echo "🔧 Restoring NuGet packages..."
        sh "dotnet restore ${PROJECT}"
      }
    }

    stage('Build') {
      steps {
        echo "🏗️ Building the project..."
        sh "dotnet build ${PROJECT} --configuration Release --no-restore"
      }
    }

    stage('SonarQube Analysis') {
      steps {
        echo "🧠 Running SonarQube static code analysis..."
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_TOKEN')]) {
          dir('src/Web') {
            sh '''
              # Ensure the SonarScanner for .NET is installed
              dotnet tool install --global dotnet-sonarscanner || true
              export PATH="$PATH:$HOME/.dotnet/tools"

              # Start SonarQube analysis
              dotnet sonarscanner begin /k:"eShopOnWeb" /d:sonar.host.url=${SONAR_URL} /d:sonar.login=$SONAR_TOKEN

              # Build the project (required between begin & end)
              dotnet build Web.csproj --configuration Release

              # End SonarQube analysis
              dotnet sonarscanner end /d:sonar.login=$SONAR_TOKEN
            '''
          }
        }
      }
    }

    stage('OWASP Dependency-Check Scan') {
      steps {
        echo "🧩 Running OWASP Dependency-Check scan..."
        dir('src/Web') {
          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                            odcInstallation: 'DP-Check'
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
          }

          catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
            publishHTML(target: [
              allowMissing: true,
              keepAll: true,
              reportDir: 'src/Web',
              reportFiles: 'dependency-check-report.html',
              reportName: 'Dependency-Check Report'
            ])
          }
        }
      }
    }

    stage('Publish') {
      steps {
        echo "🚀 Publishing .NET app..."
        sh "dotnet publish ${PROJECT} -c Release -o ${PUBLISH_DIR}"
        stash includes: "${PUBLISH_DIR}/**", name: "publish-artifacts"
      }
    }

    stage('Deploy to IIS') {
      agent { label 'windows' }
      steps {
        echo "🟢 Deploying to IIS..."
        unstash "publish-artifacts"
        bat """
        if not exist "${IIS_PATH}" mkdir "${IIS_PATH}"

        REM Copy published files
        xcopy /s /y ${PUBLISH_DIR}\\* "${IIS_PATH}\\"

        REM Configure IIS site
        powershell -NoProfile -Command "Import-Module WebAdministration; if (!(Get-Website -Name '${SITE_NAME}' -ErrorAction SilentlyContinue)) { New-Website -Name '${SITE_NAME}' -Port ${PORT} -PhysicalPath '${IIS_PATH}' -ApplicationPool '${APP_POOL}' } else { Set-ItemProperty 'IIS:\\Sites\\${SITE_NAME}' -Name bindings -Value @{protocol='http';bindingInformation='*:${PORT}:'} }; Restart-WebAppPool '${APP_POOL}'"
        """
      }
    }
  }

  post {
    success {
      echo "✅ Deployment succeeded. App is live at: http://localhost:${PORT}/"
    }
    failure {
      echo "❌ Deployment failed. Please check the Jenkins console logs for details."
    }
  }
}
