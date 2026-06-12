# spi-images

This project contains the base images for Docker-based builds of the SwiftPackageIndex Build System.

The builds are processed as follows (in abbreviated form): 

```
docker run $IMAGE_NAME swift build
```

The `basic` image provides Linux builds, while the other variants add to that to provide build environments also for Android and for Wasm.
