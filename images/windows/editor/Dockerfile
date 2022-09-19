ARG baseImage="unityci/base:windows-latest"
# ARG hubImage="unityci/hub:windows-latest"
# Using local hub image b/c of issue with unity-hub chocolatey package
ARG hubImage="hub"

#############
#  Builder  #
#############

FROM $hubImage as Builder

# Install Editor
# The portion after the & is to change the exit code to 0 if we exited 1 from
# the installation. Otherwise docker believes there was an error
ARG version
ARG changeSet
RUN "C:/Program Files/Unity Hub/Unity Hub.exe" -- --headless install \
                                               --version %version% \
                                              # --changeset %changeSet% \
                                               & if %ERRORLEVEL% EQU 1 exit 0

ARG module
RUN IF NOT %module% == "base" ( \
    "C:/Program Files/Unity Hub/Unity Hub.exe" -- --headless install-modules \
                                               --version %version% \
                                               --module %module% \
                                               --childModules \
                                               & if %ERRORLEVEL% EQU 1 exit 0 \
    )

############
#  Editor  #
############

FROM $baseImage

# Copy the editor from the builder to the final editor image
COPY --from=Builder ["C:/Program Files/Unity/Hub/Editor/", "C:/Program Files/Unity/Hub/Editor/"]

# Need to grab these dependencies that the editor needs to run
COPY --from=Builder C:/windows/system32/MSVCP100.dll \
                    C:/windows/system32/MSVCP120.dll \
                    C:/windows/system32/MSVCR100.dll \
                    C:/windows/system32/MSVCR120.dll \
                    C:/windows/system32/

# Add version to file at editor path
ARG version
RUN echo %version% > "C:/Program Files/Unity/Hub/Editor/%version%/version"

# Need to enable these services to make Unity happy
# When these were in base, things blew up, not sure why they have to be specifically here
RUN powershell -Command foreach ($service in 'nlasvc', 'netprofm') {"Set-Service $service -StartupType Automatic"}

# Not needed with the dotnet base image
# RUN powershell -Command Set-Service 'wmiApSrv' -StartupType Automatic

# Packages/manifest.json could have git dependencies
RUN choco install git --no-progress -y

#=======================================================================================
# [2019.x/2020.x/2021.x-android/2022.x-android] Support usage of installed Android SDKs outside of editor
#=======================================================================================

SHELL ["powershell", "-Command"]
RUN $Env:UNITY_PATH="\"C:\\Program Files\\Unity\\Hub\\Editor\\$Env:VERSION\\\""; \
    [Environment]::SetEnvironmentVariable('UNITY_PATH',$Env:UNITY_PATH, 'Machine'); \
    Write-Host "UNITY_PATH is $Env:UNITY_PATH"; \
    $Env:RAW_ANDROID_SDK_ROOT=((Get-Content "\"$Env:UNITY_PATH/modules.json\"") -replace 'preselected','_preselected' | ConvertFrom-Json | % {$_} | Where-Object id -Match 'android-sdk-platform-tools.*' | Select-Object -ExpandProperty destination); \
    # We need to replace some characters common to paths that will break the sed expression when expanded
    $Env:ANDROID_SDK_ROOT=($Env:RAW_ANDROID_SDK_ROOT -replace '{UNITY_PATH}',"\"$Env:UNITY_PATH\"" | Convert-Path); \
    [Environment]::SetEnvironmentVariable('ANDROID_SDK_ROOT',$Env:ANDROID_SDK_ROOT, 'Machine'); \
    Write-Host "ANDROID_SDK_ROOT is $Env:ANDROID_SDK_ROOT"; \
    [Environment]::SetEnvironmentVariable('ANDROID_HOME',$Env:ANDROID_SDK_ROOT, 'Machine'); \
    Write-Host "ANDROID_HOME is $Env:ANDROID_SDK_ROOT"; \
    $Env:RAW_ANDROID_NDK_HOME=((Get-Content "\"$Env:UNITY_PATH/modules.json\"") -replace 'preselected','_preselected' | ConvertFrom-Json | % {$_} | Where-Object id -Match 'android-ndk.*' | Select-Object -ExpandProperty destination); \
    $Env:ANDROID_NDK_HOME=($Env:RAW_ANDROID_NDK_HOME -replace '{UNITY_PATH}',"\"$Env:UNITY_PATH\"" | Convert-Path); \
    [Environment]::SetEnvironmentVariable('ANDROID_NDK_HOME',$Env:ANDROID_NDK_HOME, 'Machine'); \
    Write-Host "ANDROID_NDK_HOME is $Env:ANDROID_NDK_HOME"; \
    $Env:RAW_JAVA_HOME=((Get-Content "\"$Env:UNITY_PATH/modules.json\"") -replace 'preselected','_preselected' | ConvertFrom-Json | % {$_} | Where-Object id -Match 'android-open-jdk.*' | Select-Object -ExpandProperty destination); \
    $Env:JAVA_HOME=(($Env:RAW_JAVA_HOME) -replace '{UNITY_PATH}',"\"$Env:UNITY_PATH\"" | Convert-Path); \
    [Environment]::SetEnvironmentVariable('JAVA_HOME',$Env:JAVA_HOME, 'Machine'); \
    Write-Host "JAVA_HOME is $Env:JAVA_HOME"; \
    # Unity 2019.x doesn't have the jdk in the modules, so put in a fallback. sdkmanager will fail if invaild
    [Environment]::SetEnvironmentVariable('PATH',("\"$Env:JAVA_HOME/bin;\"" + "\"$Env:ANDROID_SDK_ROOT/tools;\"" + "\"$Env:ANDROID_SDK_ROOT/tools/bin;\"" + "\"$Env:ANDROID_SDK_ROOT/platform-tools;\"" + 'C:/Program Files/Git/usr/bin' + $Env:PATH), 'Machine'); \
    # Accept licenses
    for($i=0;$i -lt 30;$i++) { $response += 'y`n'}; $response | & "\"$Env:ANDROID_SDK_ROOT/cmdline-tools/6.0/bin/sdkmanager.bat\"" --licenses
