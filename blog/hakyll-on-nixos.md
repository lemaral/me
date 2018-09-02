# Starting Hakyll on NixOS

Many Nix users have a blog post about how they use it and how great it is; same goes for Hakyll users. Now I want to hop on the bandwagon and tell you how I happily started this blog with Hakyll and Nix.

I structured this blogs source to two trees; a `site/` folder containing the posts and templates and a `generator/` folder which contains the source of the generator. So, I wrote two derivations with Nix;

* One takes the `generator/` folder as an input and produces the executable:

```java
generator = pkgs.stdenv.mkDerivation {
  name = "utdemircom-generator";
  src = ./generator;
  phases = "unpackPhase buildPhase";
  buildInputs = [
    (pkgs.haskellPackages.ghcWithPackages (p: with p; [ hakyll ]))
  ];
  buildPhase = ''
    mkdir -p $out/bin
    ghc -O2 -dynamic --make Main.hs -o $out/bin/generate-site
  '';
};
```

* And another one takes the generator executable and `site/` folder and spits out the source of this blog:

```java
site = pkgs.stdenv.mkDerivation {
  name = "utdemircom-site";
  src = ./site;
  phases = "unpackPhase buildPhase";
  buildInputs = [ generator bootswatch ];
  buildPhase = ''
    cp ${bootswatch}/cosmo/bootstrap.min.css css/

    export LOCALE_ARCHIVE="${pkgs.glibcLocales}/lib/locale/locale-archive";
    export LANG=en_US.UTF-8
    generate-site build

    mkdir $out
    cp -r _site/* $out
  '';
}
```

`bootswatch` there is just an another derivation which fetches the source of the bootswatch:

```java
bootswatch = pkgs.fetchgit {
  url = "git://github.com/thomaspark/bootswatch";
  rev = "15748399fcd2d06b58206cfcb4b7560cf7243637";
  sha256 = "12l9s8lj4lwx0xf0w17xqaqhnvh5mj6qngxyppdi3mrg2zhvh3jc";
};
```

Now all we need to do is saving those expressions to a `default.nix` file and giving `nix-build` command.

Those expression will do what it needs to do to get `hakyll` on path (which includes fetching and building ghc and its dependencies, compiling pandoc etc). After that, it'll build the `generator/Main.hs` to `generate-site` executable, then fetch the bootswatch source, then use `bootswatch` and `generate-site` to compile `site/` and finally puts a symlink to the final output, which is the complete source of `utdemir.com`.

This wasn't pretty exciting, I know, an even shorter shell script could also give the same result. But with nix, I also got these for free:

* To build this site, the only dependency is `nix` package manager, which works on almost every Linux distribution and also on OSX. And on every system `nix` supports, the whole build process is just `nix-build`.
* All derivations are cached in `/nix/store` folder. It means that if I try `nix-build` again without changing any build inputs, it'll do nothing and just give the already compiled derivation. This also applies to dependencies, `nix` will only compile the changed derivations while reusing dependencies from `/nix/store`.
* Since a derivation is just a function, `nix` can do many optimizations to speed up the build process. It'll compile almost every dependency in parallel, can use other machines on its build farm (if any), search the derivation on public or private package caches. On this case, it'll probably fetch `hakyll` and it's runtime dependencies from public package cache (`https://cache.nixos.org`), while fetching `bootswatch` from `github` in parallel, and just use my system for compiling `generate-site` executable and using it to build this blog.
* Finally, it does every one of those things without polluting the system with any dependencies. There isn't anything (`ghc`, `hakyll` etc.) left on my system after `nix-build`, just a `result` symlink. (Well, they are cached in `/nix/store`, and I can easily get rid of them using `nix-collect-garbage`).

----

This was one of my first Nix derivations and it was easier than I thought. As someone who used to wrestle with package managers (from `apt-get` to `cabal-install`), using Nix was a joy. I hope I can find more use cases where Nix and NixOS shine!

