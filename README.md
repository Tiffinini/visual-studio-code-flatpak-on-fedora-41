# Visual Studio Code Flatpak on Fedora 41 Kinoite with Dev Containers and Podman
Here's how I personally got the Visual Studio Code Flatpak to play nicely with my setup on Fedora 41 Kinoite (Atomic) using Podman and Dev Containers. YMMV.

1. Install the Visual Studio Code and Podman (Remote Client) flatpaks:
    ```
    flatpak install com.visualstudio.code com.visualstudio.code.tool.podman
    ```

2. Enable the Podman API socket on the host:

    ```
    systemctl --user enable --now podman.socket
    ```
3. Override the VSC flatpak permissions to allow it to use Podman:

    ```
    flatpak override --user com.visualstudio.code --filesystem=xdg-run/podman
    ```
4. Optionally, set the BUILDAH_FORMAT environment variable so Podman uses the `docker` image format instead of `OCI`:

    ```
    flatpak override --user com.visualstudio.code --env BUILDAH_FORMAT=docker
    ```

5. Within Visual Studio Code, install the Dev Containers extension. Configure the following settings for that extension to it uses the podman-remote binary and the appropriate Podman socket. The "1000" part is dependent on your user's ID.
    ```
    "dev.containers.dockerPath": "/app/tools/podman/bin/podman-remote"
    "dev.containers.dockerSocketPath": "unix:///run/user/1000/podman/podman.sock"
    ```
    
6. Within `devcontainer.json`, I use some options to ensure the user and security options are appropriate. Here's a sample:
    ```
    {
        "build": {
            // Path is relative to the devcontainer.json file.
            "dockerfile": "Dockerfile"
        },
        "customizations": {
            "vscode": {
                "extensions": [
                    "ms-python.python",
                    "ms-python.debugpy",
                    "ms-python.pylint",
                    "ms-python.vscode-pylance",
                    "njpwerner.autodocstring"
                ]
            }
        },
        "postCreateCommand": "pip3 install -r /tmp/requirements.txt",
    
        "runArgs": ["--userns=keep-id"],
        "containerUser": "vscode",
        "remoteUser": "vscode",
        "securityOpt": ["label=disable"]
    }
    ```
    Pertinent to note, I've used the "securityOpt" setting to ensure SELinux isn't blocked from reading the code files mounted within the container and matched the user in the container matches my local user.

7. The `Dockerfile` used in conjunction with the above looks like this:
    ```
    FROM docker.io/python:3.12-slim-bookworm
    COPY requirements.txt /tmp
    RUN <<EOF
        useradd -m -u 1000 vscode
        apt-get update
        apt-get install -y git build-essential
        python3 -m pip config set global.break-system-packages true
    EOF
    ```
    Again, relevant to this is the addition of a UID 1000 user that matches my host system login.

Again, YMMV when using the stock MS dev container images. These settings are an example of a working setup for my Python development.

## Additional Settings for KDE Wallet Access
Additionally, since I use KDE, VSC needs access to the KDE Wallet in order to store/read secrets for cloud syncing settings, etc. This additional Flatpak override will fix that:
    ```
    flatpak override --user com.visualstudio.code --talk-name=org.kde.kwalletd6
    ```
