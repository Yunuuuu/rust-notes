# Rust Notes

Notes on Rust language semantics and API design, built with
[mdBook](https://rust-lang.github.io/mdBook/).

Book URL after deployment: <https://yunuuuu.github.io/rust-notes/>

## Local preview

Install the latest mdBook release, then run:

```sh
mdbook serve --open
```

To generate the static site in `book/`:

```sh
mdbook build
```

## GitHub Pages

Before the first deployment, open the repository's
[Pages settings](https://github.com/Yunuuuu/rust-notes/settings/pages) and select **GitHub Actions**
under **Build and deployment → Source**. This enables the Pages site; adding the workflow alone does
not enable it. Commit and push the workflow to `main` to start the first deployment.

The [deployment workflow](.github/workflows/deploy.yml) follows the **Using deploy via actions**
example linked from the
[mdBook Automated Deployment wiki](https://github.com/rust-lang/mdBook/wiki/Automated-Deployment).
The
[example](https://github.com/rust-lang/mdBook/wiki/Automated-Deployment:-GitHub-Actions#using-deploy-via-actions)
downloads the latest precompiled mdBook release, builds the book, and deploys it in one job. This
workflow also retains manual triggering, deployment concurrency control, and the `github-pages`
environment. Repository contents only need read permission because deployment uploads an artifact.

The **Deploy** workflow publishes the book on every push to `main`. To publish manually, open the
workflow in the **Actions** tab, select **Run workflow**, and choose `main`.

Deployment uses the built-in `GITHUB_TOKEN`. No personal access token is required after Pages has
been enabled. If **Setup Pages** reports **Get Pages site failed / Not Found**, enable Pages as
described above, then rerun the failed workflow.
