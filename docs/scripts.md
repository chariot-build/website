---
icon: lucide/file-code
---

# Scripts

Scripts are code snippets used to do various things. Currently there are four places where scripts are used:

- Source -> Prepare
- Package -> Configure
- Package -> Build
- Package -> Install

Scripts can be written in multiple languages. The current supported languages are:

- Bash
- Python

## Environment

Scripts run in an isolated container to maximize ease of reproducibility. The root filesystem is read-only, the designated directories for work are accessible via [environment variables](#environment-variables).

### Environment Variables

Chariot exposes various things into the runtime using environment variables.

| Variable      | Availability                    | Description                                         |
| ------------- | ------------------------------- | --------------------------------------------------- |
| `PARALLELISM` | Everywhere                      | Ideal number of threads to use for parallel builds. |
| `SOURCES_DIR` | Everywhere                      | Path to the directory where sources are available.  |
| `SOURCE_DIR`  | `prepare`                       | Path to the source directory.                       |
| `BUILD_DIR`   | `configure`, `build`, `install` | Path to the build directory.                        |
| `INSTALL_DIR` | `install`                       | Path to the install directory.                      |
