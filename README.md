# Rust Notes

Notes on Rust language semantics and API design, built with
[mdBook](https://rust-lang.github.io/mdBook/).

Book URL after deployment: <https://yunuuuu.github.io/rust-notes/>

## Local preview

Install mdBook 0.5.4, then run:

```sh
mdbook serve --open
```

To generate the static site in `book/`:

```sh
mdbook build
```

## GitHub Pages

In the repository's **Settings → Pages → Build and deployment**, select **GitHub Actions** as the
source. Commit and push the workflow to `main` to start the first deployment.

The [deployment workflow](.github/workflows/mdbook.yml) follows
[GitHub's official mdBook Pages template](https://github.com/actions/starter-workflows/blob/main/pages/mdbook.yml),
adapted for `main` and mdBook 0.5.4, with the Rust installer argument corrected and Cargo's
`--locked` option enabled.

The **Deploy mdBook site to Pages** workflow publishes the book on every push to `main`. To publish
manually, open the workflow in the **Actions** tab, select **Run workflow**, and choose `main`.

The workflow installs mdBook 0.5.4 and deploys the generated `book/` directory using the built-in
`GITHUB_TOKEN`. No personal access token is required.
