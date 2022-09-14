ARG baseImage="unityci/base:windows-latest"

FROM $baseImage

# Install unity hub
# Adding explicit version - current checksum mismatch on 3.2.0
RUN choco install unity-hub --no-progress -y --version=3.3.0
