This `build/` folder is unmodified upstream tooling (Docker Hub multi-arch
push via `docker buildx`, publishing to `karloygard/xcomfortd-go`). It is
**not** used by this App - Home Assistant Supervisor builds directly from
the `Dockerfile` and `build.yaml` at the repository root instead. Left in
place only because the rest of `src/` is an unmodified upstream checkout
except for `main.go` and the new `pkg/xc/collision_test.go`.
