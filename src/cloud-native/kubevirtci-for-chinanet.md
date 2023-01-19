release time :2022-08-31 12:58


Without modification, an error will be reported if executed according to the KubeVirt getting-started document make && make push && make manifests.

The reason is the domestic network environment.

The best way is to make the network environment scientific. If you can only use the domestic network environment due to network restrictions, there are at least two places that need to be modified:

# Add go package domestic proxy
hack/bootstrap.shset -eAdd two lines below

    GO_PROXY="https://goproxy.cn,direct"
    go env -w GOPROXY=${GO_PROXY}



# The image on gcr.io of the file in the root directory of the project WORKSPACEis replaced with an image that can be accessed in China.
For example release-0.53, the version of KubeVirt only needs to modify WORKSPACEthe following content:

    # Pull go_image_base
    container_pull(
        name = "go_image_base",
        digest = "sha256:f65536ce108fcc41cdcd5cb101006fcb82b9a1527409263feb9e34032f00bda0",
        registry = "gcr.io",
        repository = "distroless/base",
    )

> Because there is a large amount of content that needs to be accessed on Github, github.com is not blocked in China, but there is a certain probability that it cannot be accessed, so make && make push && make manifestseven if the process is modified according to the above, there is still a certain probability of failure.



