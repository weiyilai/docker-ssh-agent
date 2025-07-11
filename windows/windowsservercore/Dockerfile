# escape=`

# The MIT License
#
#  Copyright (c) 2019-2020, Alex Earl
#
#  Permission is hereby granted, free of charge, to any person obtaining a copy
#  of this software and associated documentation files (the "Software"), to deal
#  in the Software without restriction, including without limitation the rights
#  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#  copies of the Software, and to permit persons to whom the Software is
#  furnished to do so, subject to the following conditions:
#
#  The above copyright notice and this permission notice shall be included in
#  all copies or substantial portions of the Software.
#
#  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
#  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
#  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
#  THE SOFTWARE.

ARG WINDOWS_VERSION_TAG
ARG TOOLS_WINDOWS_VERSION
FROM mcr.microsoft.com/windows/servercore:"${WINDOWS_VERSION_TAG}" AS jdk-core

# $ProgressPreference: https://github.com/PowerShell/PowerShell/issues/2138#issuecomment-251261324
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ARG JAVA_VERSION
RUN New-Item -ItemType Directory -Path C:\temp | Out-Null ; `
    $javaMajorVersion = $env:JAVA_VERSION.substring(0,2) ; `
    $msiUrl = 'https://api.adoptium.net/v3/installer/version/jdk-{0}/windows/x64/jdk/hotspot/normal/eclipse?project=jdk' -f $env:JAVA_VERSION ; `
    Invoke-WebRequest $msiUrl -OutFile 'C:\temp\jdk.msi' ; `
    $proc = Start-Process -FilePath 'msiexec.exe' -ArgumentList '/i', 'C:\temp\jdk.msi', '/L*V', 'C:\temp\OpenJDK.log', '/quiet', 'ADDLOCAL=FeatureEnvironment,FeatureJarFileRunWith,FeatureJavaHome',  "INSTALLDIR=C:\openjdk-${javaMajorVersion}" -Wait -Passthru ; `
    $proc.WaitForExit() ; `
    Remove-Item -Path C:\temp -Recurse | Out-Null

FROM mcr.microsoft.com/windows/servercore:"${WINDOWS_VERSION_TAG}"

ARG JAVA_HOME
ENV JAVA_HOME=${JAVA_HOME}

COPY --from=jdk-core $JAVA_HOME $JAVA_HOME

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ARG user=jenkins
ARG JENKINS_AGENT_WORK="C:/Users/${user}/Work"
ENV JENKINS_AGENT_USER=${user}
ENV JENKINS_AGENT_WORK=${JENKINS_AGENT_WORK}

USER ContainerAdministrator

# Install git
ARG GIT_VERSION=2.50.1
ARG GIT_PATCH_VERSION=1
RUN [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; `
    # The patch "windows.1" always have a different URL than the subsequent patch (ZIP filename is different)
    if($env:GIT_PATCH_VERSION -eq 1) { $url = $('https://github.com/git-for-windows/git/releases/download/v{0}.windows.{1}/MinGit-{0}-64-bit.zip' -f $env:GIT_VERSION, $env:GIT_PATCH_VERSION); } `
    else {$url = $('https://github.com/git-for-windows/git/releases/download/v{0}.windows.{1}/MinGit-{0}.{1}-64-bit.zip' -f $env:GIT_VERSION, $env:GIT_PATCH_VERSION)} ; `
    Write-Host "Retrieving $url..." ; `
    Invoke-WebRequest $url -OutFile 'mingit.zip' -UseBasicParsing ; `
    Expand-Archive mingit.zip -DestinationPath c:\mingit ; `
    Remove-Item mingit.zip -Force

# Install git-lfs
ARG GIT_LFS_VERSION=3.7.0
RUN $CurrentPath = (Get-Itemproperty -path 'hklm:\system\currentcontrolset\control\session manager\environment' -Name Path).Path ; `
    $NewPath = $CurrentPath + ';{0}\bin;C:\mingit\cmd' -f $env:JAVA_HOME ; `
    Set-ItemProperty -path 'hklm:\system\currentcontrolset\control\session manager\environment' -Name Path -Value $NewPath ; `
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; `
    $url = $('https://github.com/git-lfs/git-lfs/releases/download/v{0}/git-lfs-windows-amd64-v{0}.zip' -f $env:GIT_LFS_VERSION) ; `
    Write-Host "Retrieving $url..." ; `
    Invoke-WebRequest $url -OutFile 'GitLfs.zip' -UseBasicParsing ; `
    Expand-Archive GitLfs.zip -DestinationPath c:\mingit\mingw64\bin ; `
    $gitLfsFolder = 'c:\mingit\mingw64\bin\git-lfs-{0}' -f $env:GIT_LFS_VERSION ; `
    Move-Item -Path "${gitLfsFolder}\git-lfs.exe" -Destination c:\mingit\mingw64\bin\ ; `
    Remove-Item -Path $gitLfsFolder -Recurse -Force ; `
    Remove-Item GitLfs.zip -Force ; `
    & C:\mingit\cmd\git.exe lfs install

# Setup SSH server
ARG OPENSSH_VERSION=v9.8.3.0p2-Preview
RUN [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; `
    $url = 'https://github.com/PowerShell/Win32-OpenSSH/releases/download/{0}/OpenSSH-Win64.zip' -f $env:OPENSSH_VERSION ; `
    Write-Host "Retrieving $url..." ; `
    Invoke-WebRequest -Uri $url -OutFile C:/openssh.zip -UseBasicParsing ; `
    Expand-Archive c:/openssh.zip 'C:/Program Files' ; `
    Remove-Item C:/openssh.zip ; `
    $env:PATH = '{0};{1}' -f $env:PATH,'C:\Program Files\OpenSSH-Win64' ; `
    & 'C:/Program Files/OpenSSH-Win64/install-sshd.ps1' ; `
    if(!(Test-Path 'C:\ProgramData\ssh')) { New-Item -Type Directory -Path 'C:\ProgramData\ssh' | Out-Null } ; `
    Copy-Item 'C:\Program Files\OpenSSH-Win64\sshd_config_default' 'C:\ProgramData\ssh\sshd_config' ; `
    $content = Get-Content -Path "C:\ProgramData\ssh\sshd_config" ; `
    $content | ForEach-Object { $_ -replace '#PermitRootLogin.*','PermitRootLogin no' `
                        -replace '#PasswordAuthentication.*','PasswordAuthentication no' `
                        -replace '#PermitEmptyPasswords.*','PermitEmptyPasswords no' `
                        -replace '#PubkeyAuthentication.*','PubkeyAuthentication yes' `
                        -replace '#SyslogFacility.*','SyslogFacility LOCAL0' `
                        -replace '#LogLevel.*','LogLevel INFO' `
                        -replace 'Match Group administrators','' `
                        -replace '(\s*)AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys','' `
                } | `
    Set-Content -Path "C:\ProgramData\ssh\sshd_config" ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value 'ChallengeResponseAuthentication no' ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value 'HostKeyAgent \\.\pipe\openssh-ssh-agent' ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value ('Match User {0}' -f $env:JENKINS_AGENT_USER) ; `
    Add-Content -Path "C:\ProgramData\ssh\sshd_config" -Value ('       AuthorizedKeysFile C:/Users/{0}/.ssh/authorized_keys' -f $env:JENKINS_AGENT_USER) ; `
    New-Item -Path HKLM:\SOFTWARE -Name OpenSSH -Force | Out-Null ; `
    New-ItemProperty -Path HKLM:\SOFTWARE\OpenSSH -Name DefaultShell -Value 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe' -PropertyType string -Force | Out-Null ; `
    Remove-Item -Path "C:\ProgramData\ssh\ssh_host*_key*"

COPY CreateProfile.psm1 C:/

# Create user and user directory
RUN Import-Module -Force C:/CreateProfile.psm1 ; `
    New-UserWithProfile -UserName $env:JENKINS_AGENT_USER -Description 'Jenkins Agent User' ; `
    Remove-Item -Force C:/CreateProfile.psm1

VOLUME "${JENKINS_AGENT_WORK}" "C:/Users/${user}/AppData/Local/Temp"
WORKDIR "${JENKINS_AGENT_WORK}"

COPY setup-sshd.ps1 C:/ProgramData/Jenkins/setup-sshd.ps1

EXPOSE 22

LABEL `
    org.opencontainers.image.vendor="Jenkins project" `
    org.opencontainers.image.title="Official Jenkins SSH Agent Docker image" `
    org.opencontainers.image.description="A Jenkins agent image which allows using SSH to establish the connection" `
    org.opencontainers.image.url="https://www.jenkins.io/" `
    org.opencontainers.image.source="https://github.com/jenkinsci/docker-ssh-agent" `
    org.opencontainers.image.licenses="MIT"

ENTRYPOINT ["powershell.exe", "-NoExit", "-Command", "& C:/ProgramData/Jenkins/setup-sshd.ps1"]
