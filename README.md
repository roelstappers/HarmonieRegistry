# HarmonieRegistry

This README.md is a copy of [HolyLabRegistry](https://github.com/HolyLab/HolyLabRegistry) with small adaptations. 

This registry allows you to use packages from Harmonie in Julia 1.x.

# Usage

If you're using at least Julia 1.1, then you can add this registry with

```
]registry add git@github.com:roelstappers/HarmonieRegistry.git
```

(The `]` enters Pkg mode when you type it at the REPL prompt, see https://docs.julialang.org/en/latest/stdlib/Pkg/.)

For earlier Julia versions, manually `git clone` this repository under `DEPOT_PATH/registries`. (Usually, `DEPOT_PATH = /home/username/.julia`)

Then, we can use Harmonie private packages (or unregistered public ones) as if they are registered ones.

# For package developers

## Preparing your package before registering it

### Creating the directory and `Project.toml` file

You have two options:

- The manual way (generally not recommended):

  * Change your working directory to your development directory, typically `~/.julia/dev/`

  * Generate a package. Let your package name be 'MyPkg', then
    ```julia
    (v1.0) pkg> generate MyPkg
    ```
    This will generate a 'MyPkg' directory, 'Project.toml' including UUID, and sample source files.
    If you have already created the directory, this will cause an error.
    In this case, you can copy the 'Project.toml' file from a different package and edit it
    appropriately. Make sure you assign a new UUID, which can be generated with

- Using [PkgTemplates.jl](https://github.com/invenia/PkgTemplates.jl) (recommended):

  To create a new package and host it in your own GitHub account, use

  ```julia
  julia> using PkgTemplates

  julia> t = Template(ssh=true, plugins=[TravisCI()])  # creates a template for your personal account
  Template:
      → User: <username>
      → Host: github.com
      → License: MIT (<username> 2020)
      → Package directory: /tmp/pkgs/dev
      → Minimum Julia version: v1.1
      → SSH remote: Yes
      → Commit Manifest.toml: No
      → Plugins: None

  julia> generate("MyPkg", t)
  # lots of output
  ```

### Adding dependent packages

- Change your working directory to `MyPkg`.

- Activate and add dependencies. Here we'll add `SubPkg1` and `SubPkg2` as dependencies:
  ```julia
  (v1.0)  pkg> activate .
  (MyPkg) pkg> add SubPkg1 SubPkg2
  ```
  This will add the dependent packages under the `[deps]` field in the 'Project.toml' and generate 'Manifest.toml' file.
  This 'Manifest.toml' file includes  specific versions of the dependent packages resolved according to your current environment.
  Usually, this file is excluded when you commit your package to a repository---if you created `MyPkg`
  using the manual method above, consider adding this to the `.gitignore` file.
  (PkgTemplates does this by default, see the "Commit Manifest.toml: No" line above.)

- Write whatever code and tests you need, commit them, and then push your package up to GitHub.

## Registering your package with HarmonieRegistry

### Using LocalRegistry

Check out a local copy of https://github.com/GunnarFarneback/LocalRegistry.jl.
Then:

- navigate to HarmonieRegistry, `$HOME/.julia/registries/HarmonieRegistry`
- update to the latest `master` branch
- check out a new branch, e.g., `git checkout -b teh/SomeNewPkg`
- start Julia and enter the following:
```julia
using LocalRegistry, SomeNewPkg
register(SomeNewPkg, "HarmonieRegistry")
```
  where you replace the specific package name and path to the appropriate value on your system.
  This will add a new commit to the branch of HarmonieRegistry you just created
- Submit the branch as a PR to HarmonieRegistry
- Once the PR merges, from the HarmonieRegistry directory do
```
$ git checkout master
$ git pull
$ git branch -D teh/SomeNewPkg
```
- Push tags for the new release (`git tag -a vx.y.z` and then `git push --tags`)

## Accessing HarmonieRegistry in travis and appveyor tests

This is required only if your package uses other private packages.

- Include the following lines in the script section of the `.travis.yml` file in the root directory
  of your package (as an example, let your package name be 'RegisterFit')

  ```
  script:
  - if [[ -a .git/shallow ]]; then git fetch --unshallow; fi
  - julia -e 'using Pkg, LibGit2;
              user_regs = joinpath(DEPOT_PATH[1],"registries");
              mkpath(user_regs);
              all_registries = Dict("General" => "https://github.com/JuliaRegistries/General.git",
                                  "HarmonieRegistry" => "https://github.com/roelstappers/HarmonieRegistry.git");
              Base.shred!(LibGit2.CachedCredentials()) do creds
              for (reg, url) in all_registries
                  path = joinpath(user_regs, reg);
                  LibGit2.with(Pkg.GitTools.clone(url, path; header = "registry $reg from $(repr(url))", credentials = creds)) do repo end
              end
              end'
  - julia -e 'using Pkg; Pkg.build(); Pkg.test("RegisterFit"; coverage=false)'
  ```

  A similar script should be used with Appveyor (for testing on Windows).  However because multiline commands and variables are nightmarish in Windows it's recommended that you move the Julia command above into a separate script that gets called from `appveyor.yml`.  You can call the same script from `.travis.yml` as well to avoid code duplication.  See https://github.com/HolyLab/ImagineInterface for an example.

- Assign your private ssh key which is paired with a public key in your Github account to the package in the Travis site.
  * Copy the contents of the private key ('id_rsa' file generated in the 'To use git protocol in GitHub' section - not 'id_rsa.pub') in the local machine to your clipboard.
  * Go to the setup page of the package in the Travis site you want to make to access this registry. (You can get there by choosing the package in your Travis repositories, clicking ‘More options’ button on the upper right corner and selecting ‘setting’ menu.)
  * Assign the private key in the clipboard to the ‘SSH Key’ field.

## Tagging a new release

### In the package directory

- Edit the version number in `Project.toml`. All version numbers should be of the form `vX.X.X`, where each `X` is a number.
  See [semantic versioning](https://semver.org/) for guidelines about choosing version numbers.
- If your release has new version requirements for dependent packages, add those dependencies to the
  `[compat]` section. For example,
  ```
  [compat]
  DocStringExtensions = ">= 0.2"
  ```
- Commit the change(s) you made to `Project.toml`
- Create a git tag: `git tag -a vX.X.X` (where this matches the version number you used in `Project.toml`).
  In your editor, write a brief description of the new features of this release.
- Incorporate the changes in the `master` branch on GitHub, either by direct push or submitting a pull request.
- In preparation for the next step, execute `git cat-file -p vX.X.X` where again the version number matches your previous choices.

### In HarmonieRegistry

#### Using LocalRegistry

Just repeat the steps above for the initial registration, except that you don't have to specify the registry.

#### Manual approach (not recommended)

In the package's directory, update `Versions.toml` and, if necessary, `Compat.toml` and `Deps.toml`.
Use the sha from the `git cat-file` command above.

## Making a Harmonie package public on Github

- Make the package repo public by changing its Github settings.
- Be sure that you've done the same for any *dependencies* of the package.
- Submit a PR to this repo that changes the `url` field in `Project.toml` to use the https protocol instead of ssh (must also be done for any dependencies that you've made public).

# See also

- Creating a registry : https://discourse.julialang.org/t/creating-a-registry/12094