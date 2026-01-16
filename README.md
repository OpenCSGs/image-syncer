# image-syncer

Sync docker images from foreign registries to your target registries.

This README explains the configuration format (images.yaml) and gives concrete examples and expected behavior based on the mappings you supplied.

## Configuration (images.yaml)

image-syncer uses a simple YAML mapping file where keys are source images (optionally with tags, tag lists, digests, or tag regexes) and values are destination image repositories (either a single string or a list of strings).

Key formats supported:

- source repository (no tag): sync all tags from the source repository
  - Example: `quay.io/coreos/kube-rbac-proxy:`
- specific tag: sync only that tag
  - Example: `quay.io/coreos/kube-rbac-proxy:v1.0:`
- multiple tags (comma separated): sync only those tags
  - Example: `quay.io/coreos/kube-rbac-proxy:v1.0,v2.0:`
- digest (immutable): sync that image by digest
  - Example: `quay.io/coreos/kube-rbac-proxy@sha256:14b267...:`
- tag regex (enclosed in slashes): sync tags that match the regex
  - Example: `quay.io/coreos/kube-rbac-proxy:/a+/:`

Values:
- A single destination string (the destination repository). If a destination doesn't include a tag, the source tag is preserved when pushing.
- A YAML list of destinations when you want to push the same source image to multiple targets.

Notes:
- When the source key omits a tag (ends with `:`) the syncer treats it as "all tags" for that repository.
- For digest mappings (using `@sha256:...`) the syncer will pull that specific content addressable image and push to the destinations.
- Tag regexes are standard regular expressions (enclosed in `/.../`) and are matched against tag strings.
- Multiple tags in a single key are comma-separated with no spaces.

## Example images.yaml

Below are the example entries you provided, shown verbatim and followed by what the syncer will do for each:

```yaml
quay.io/coreos/kube-rbac-proxy: quay.io/ruohe/kube-rbac-proxy
quay.io/coreos/kube-rbac-proxy:v1.0: quay.io/ruohe/kube-rbac-proxy
quay.io/coreos/kube-rbac-proxy:v1.0,v2.0: quay.io/ruohe/kube-rbac-proxy
quay.io/coreos/kube-rbac-proxy@sha256:14b267eb38aa85fd12d0e168fffa2d8a6187ac53a14a0212b0d4fce8d729598c: quay.io/ruohe/kube-rbac-proxy
quay.io/coreos/kube-rbac-proxy:v1.1:
  - quay.io/ruohe/kube-rbac-proxy1
  - quay.io/ruohe/kube-rbac-proxy2
quay.io/coreos/kube-rbac-proxy:/a+/: quay.io/ruohe/kube-rbac-proxy
```

Expected behavior for each mapping:

- `quay.io/coreos/kube-rbac-proxy: quay.io/ruohe/kube-rbac-proxy`
  - Sync all tags from `quay.io/coreos/kube-rbac-proxy` to `quay.io/ruohe/kube-rbac-proxy`, preserving tag names.

- `quay.io/coreos/kube-rbac-proxy:v1.0: quay.io/ruohe/kube-rbac-proxy`
  - Sync only tag `v1.0` from the source repo to the destination repo as `quay.io/ruohe/kube-rbac-proxy:v1.0`.

- `quay.io/coreos/kube-rbac-proxy:v1.0,v2.0: quay.io/ruohe/kube-rbac-proxy`
  - Sync tags `v1.0` and `v2.0` from the source and push both to the destination repository, preserving tags.

- `quay.io/coreos/kube-rbac-proxy@sha256:14b267...: quay.io/ruohe/kube-rbac-proxy`
  - Pull the image identified by the given digest and push that exact content to the destination repository. Destination will receive the image (it may be referenced by digest or by an implied tag if your workflow assigns one).

- `quay.io/coreos/kube-rbac-proxy:v1.1:
    - quay.io/ruohe/kube-rbac-proxy1
    - quay.io/ruohe/kube-rbac-proxy2`
  - Pull `v1.1` and push it to two different destination repositories (`quay.io/ruohe/kube-rbac-proxy1:v1.1` and `quay.io/ruohe/kube-rbac-proxy2:v1.1`).

- `quay.io/coreos/kube-rbac-proxy:/a+/: quay.io/ruohe/kube-rbac-proxy`
  - Match and sync all tags on the source that match the regex `a+` (i.e., tags like `a`, `aa`, `aaa`, ...) to the destination repo.

## Usage

The exact invocation depends on your build and distribution of image-syncer; the syncer reads the mapping file and performs pulls and pushes accordingly. Typical usage patterns:

- CLI (example)
  - image-syncer --config /path/to/images.yaml

- Docker run (example)
  - docker run --rm -v $(pwd)/images.yaml:/etc/images.yaml image-syncer:latest --config /etc/images.yaml

Make sure the runtime has access to credentials for source and destination registries (for example via Docker config, environment variables, or credential helpers), otherwise authenticated registries will fail.

## Best practices & tips

- Prefer using digests for immutable references when you need reproducible content.
- When syncing many tags, rate limits on public registries may apply—consider mirroring during off-peak times.
- Test mappings that use regex carefully to ensure you don't accidentally match unintended tags.
- Use multiple destinations only when you intentionally want duplicates pushed to different registries.

## Troubleshooting

- Authentication errors: verify registry credentials and permissions.
- Rate limiting: consider adding backoff/retry or splitting sync jobs.
- Missing tags: confirm the tag exists on the source registry (or adjust regex).

## Contributing

PRs and issues are welcome. Please include examples and test cases for new mapping behaviors.
