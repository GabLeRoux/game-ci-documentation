# Customize GameCI unity docker images

Sometimes, you will want to add tools to the existing unity docker images for specific needs. For example, in some projects, you might need Blender installed for unity to allow building a project with blender assets. Sometimes, you'll need some specific command lines or runtime that are used by unity in a _postprocess_ step. This documentation will guide you into building your own images on top of the existing ones to add your desired tools.

## Example: Add ruby to existing images

In the following example, we will be adding `Ruby` on top of the existing docker images (all unity components) for the `2021.1.16f1` unity version and GameCI's `0.15.0` version and publish them.

### Build script

`build.sh`

```bash
#!/usr/bin/env bash

set -ex

UNITY_VERSION=2021.1.16f1
GAME_CI_VERSION=0.15.0 # https://github.com/game-ci/docker/releases
MY_USERNAME=gableroux

# don't hesitate to remove unused components from this list
declare -a components=("android" "ios" "linux-il2cpp" "mac-mono" "webgl" "windows-mono")

for component in "${components[@]}"
do
  GAME_CI_UNITY_EDITOR_IMAGE=unityci/editor:ubuntu-${UNITY_VERSION}-${component}-${GAME_CI_VERSION}
  IMAGE_TO_PUBLISH=${MY_USERNAME}/editor:ubuntu-${UNITY_VERSION}-${component}-${GAME_CI_VERSION}
  docker build --build-arg GAME_CI_UNITY_EDITOR_IMAGE=${GAME_CI_UNITY_EDITOR_IMAGE} . -t ${IMAGE_TO_PUBLISH}
# uncomment the following to publish the built images to docker hub
#  docker push ${IMAGE_TO_PUBLISH}
done
```

### Dockerfile

`Dockerfile`

```dockerfile
# initially inspired from https://github.com/drecom/docker-ubuntu-ruby/blob/master/Dockerfile
ARG RUBY_PATH=/usr/local/
ARG RUBY_VERSION=2.6.0
ARG GAME_CI_UNITY_EDITOR_IMAGE

# build ruby
FROM ubuntu:18.04 AS rubybuild

RUN apt-get update \
&&  apt-get upgrade -y --force-yes \
&&  apt-get install -y --force-yes \
    libssl-dev \
    libreadline-dev \
    zlib1g-dev \
    wget \
    curl \
    git \
    build-essential \
&&  apt-get clean \
&&  rm -rf /var/cache/apt/archives/* /var/lib/apt/lists/*

ARG RUBY_PATH
ARG RUBY_VERSION
RUN git clone git://github.com/rbenv/ruby-build.git $RUBY_PATH/plugins/ruby-build \
&&  $RUBY_PATH/plugins/ruby-build/install.sh
RUN ruby-build $RUBY_VERSION $RUBY_PATH

# inject ruby and additional dependencies in game-ci's unity editor image
FROM $GAME_CI_UNITY_EDITOR_IMAGE

ARG RUBY_PATH
ENV PATH $RUBY_PATH/bin:$PATH
RUN apt-get update && \
    apt-get install -y \
        git \
        curl \
        gcc \
        make \
        libssl-dev \
        zlib1g-dev \
        libsqlite3-dev
COPY --from=rubybuild $RUBY_PATH $RUBY_PATH
```

### Usage example

Locally, invoke the `build.sh` script (example on a unix system):

```bash
chmod +x build.sh
./build.sh
```

As a result, you will now have your own docker image with desired tools available during build time. To confirm `ruby` is correctly installed, we can invoke the command within the image we just built:

```bash
docker run --rm -it gableroux/editor:ubuntu-2021.1.16f1-android-0.15.0 ruby --version
ruby 2.6.0p0 (2018-12-25 revision 66547) [x86_64-linux]
```

If you are using [github-actions unity-builder](https://github.com/marketplace/actions/unity-builder) you can then customize the docker image used during build using [the customimage parameter](https://game.ci/docs/github/builder#customimage)

For more information on how to publish to docker hub, you may refer to [docker's documentation](https://docs.docker.com/docker-hub/repos/)
