+++
title = "Let's Hijack Some Packages!"
date = 2020-10-29
authors = ["Matúš Ferech"]
+++


Being able to hijack a Python package gives you a lot of opportunities. `pip` does not just place a package to some specified location. It runs the `setup.py` file that comes with most packages. This means you are effectively running unknown code on your machine every time you install a new package. Moreover, since `pip` runs as your user, it has the same permissions you do[^1]. It can read ssh keys, gpg keys, inspect your home directory or install ransomware, you name it. Furthermore, according to the [tiobe index](https://www.tiobe.com/tiobe-index/python/), Python is one of the most used languages in the world which makes it even more valuable resource to hijack. So, I hope, you are sold. Let’s do it!

Installing a malicious package on someone's computer is easier than one would think. To find out how do we perform the attack, we need to understand how `pip` resolution works. Firstly, pip queries all repositories it knows. The default one is PyPI, however, you can specify additional ones with `--extra-index-url` option. After getting responses, it chooses the latest available version of the package. If responses with the same versions are received from multiple repositories, `pip` [prefers PyPI](https://github.com/pypa/pip/issues/5045#issue-301252726) or treats them equally.

The important part for us is that it chooses the package with the latest version. To inject our malicious package, all we need to do is to guess the name of a package and create a new one with a higher version. Then we upload it PyPI. We choose PyPI since it’s the most widely used registry. Now when someone tries to install the package with the same name from some internal registry, `pip` will find our malicious version on PyPI and download it instead.

## What can we do about it?

One thing we can do is to install only from trusted registries, which is not ideal. To install a package from an internal repository, we would need to host its dependencies as well. How will we choose which dependencies to host? The simplest solution would be to mirror the PyPI. However, that is not always possible[^2].

`pip` can be also used in [hash-checking mode](https://pip.pypa.io/en/stable/reference/pip_install/#hash-checking-mode) to verify packages against tempering. To ensure the authenticity, developers need to download hashes for every package by hand from registries and verify it themselves, which is so tedious and clumsy nobody does it. Some tools like [poetry](https://github.com/python-poetry/poetry) try to make it more userfriendly and download and verify those hashes for you, nonetheless, since they're downloaded automatically from repositories, you lose the authenticity and are back in integrity check only.

Another solution is to be faster than the attacker. We create placeholders in PyPI first. The difference is that we will create them with low enough version instead of a higher one. Hence, when `pip` will query repositories for the specified package, it will receive responses from the internal registry as well as from PyPI. However, the package version from PyPI will be lower and `pip` will install the desired version.

Nonetheless, creating and uploading a placeholder package for every internal package is an awful amount of work. That’s why we created [`artifactory-pypi-scanner`](https://github.com/pan-net-security/artifactory-pypi-scanner) which we use at Pan-Net. It lists all Python packages in a JFrog Artifactory instance. Then it checks whether they're present on PyPI and creates new ones with the same name if they do not exist on PyPI. What's more, when a conflicting package is found, the scanner will report it in a machine-readable format (`JSON`) for further processing.

If you are interested, you can go-get it from source or download pre-built binaries from [releases](https://github.com/pan-net-security/artifactory-pypi-scanner/releases).

```sh
go get github.com/pan-net-security/artifactory-pypi-scanner
```

### Example

`artifactory-pypi-scanner` is configured by environmental variables. For detailed usage instructions see [configuration section](https://github.com/pan-net-security/artifactory-pypi-scanner#configuration) in the GitHub repository.

```js
$ artifactory-pypi-scanner | jq
{
  "error": "",
  "artifactoryPackages": 3,
  "pypiPlaceholders": 2,
  "repositoryResults": [
    {
      "error": "",
      "url": "https://af.example.com/artifactory//repository1-pypi-local",
      "packageResults": [
        {
          "error": "",
          "name": "package1",
          "isOurs": false,
          "created": false
        }
      ]
    },
    {
      "error": "",
      "url": "https://af.example.com/artifactory//repository2-pypi-local",
      "packageResults": [
        {
          "error": "",
          "name": "package2",
          "isOurs": true,
          "created": false
        },
        {
          "error": "",
          "name": "package3",
          "isOurs": true,
          "created": true
        }
      ]
    }
  ]
}
```

[^1]: Please say you're not running `pip` with `sudo`.

[^2]: For example A lets you set permissions only on a registry level. Thus, enabling anyone with access right to Artifactory to overwrite your Python package.
