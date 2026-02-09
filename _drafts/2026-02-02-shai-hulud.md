---
title: How I got affected by Shai-Hulud in PHP World
date: 2026-02-02
categories: [Security]
---

Recently, in one of my projects, I got affected by a supply chain attack called Shai-Hulud that targeted npm packages. 
The interesting fact is that it happened to me in a PHP project.
That's why I decided to write about my experience and lessons learned.

### What is Shai-Hulud?

Shai-Hulud is an npm supply-chain attack "worm" that compromises JavaScript/Node.js projects by getting malicious code published into
legitimate packages and then spreading further from there. What makes it especially interesting is its self-replicating mechanism: 
once the malware runs on a developer machine or in CI, it hunts for credentials (such as npm tokens and other secrets) and then uses 
those stolen tokens to automatically inject and publish infected updates to other packages the victim maintains, 
creating a cascading infection across the ecosystem. 
The list of affected packages can be found [here](https://www.aikido.dev/blog/shai-hulud-strikes-again-hitting-zapier-ensdomains)

The trigger mechanism relies on post-install and pre-install scripts in npm packages, which are executed automatically when the package is installed.

### The interesting part: how it affected me in a PHP project

In one of my PHP projects, I was using a tool called [Optic](https://github.com/opticdev/optic) to manage OpenAPI documentation. 
One of the dependencies of Optic was infected by Shai-Hulud. It was a coincidence that during that day I decided to update some OpenAPI docs,
and the CI pipeline started a job using Optic to verify the changes. During that process, the infected dependency got installed, and 
the malicious script executed on the CI server. 

Optic was installed globally with the latest version (`npm install -g @useoptic/optic@latest`) because it was just a tool 
to help maintain the project, not a production dependency. I didn't care about backward compatibility for such tooling, 
so using the latest version seemed like the easiest and most convenient option. This turned out to be a security mistake.

The malicious script tried to look for environment variables that could contain sensitive information, such as npm tokens, AWS credentials, etc. Fortunately, 
the CI environment did not have such variables, so the attack was not successful. 

### Defense strategies

Here are some strategies that can help protect against such attacks:

#### Install NPM packages with additional options

In the case of Shai-Hulud, preventing the execution of post-install and pre-install scripts would have stopped the attack. So it's a good practice
to install npm packages with the `--ignore-scripts` option. Additionally, installing only packages that are older than a certain age can 
help avoid newly published malicious packages. This can be done using the `minimumReleaseAge` option. 

Consider using the pnpm package manager, as these options are available out of the box. 

[Read more about Security in pnpm](https://pnpm.io/supply-chain-security)

#### Do not install packages without specifying versions

Always use package-lock.json or equivalent file to ensure that the exact versions of dependencies are installed. 
Even if you don't care about backward compatibility, specifying versions helps to avoid installing newly published malicious packages.

#### Use Docker when possible

Execute the code in an isolated environment, such as a Docker container. This way, even if the malicious code executes, 
it will be contained within the container and will not affect the host system.

When using Docker images, always verify them using SHA256 checksums to ensure you're pulling the exact image you expect. 
Instead of using tags like `node:24.13.0` which can be updated, use the digest format: `node:24.13.0@sha256:...`. This ensures 
that even if a tag is compromised or updated, you'll only get the specific, verified image version. You can find the 
SHA256 digest by running `docker pull <image>` and noting the digest in the output, or by checking the image registry directly.

#### Add only necessary environment variables in CI

It's similar to the least privilege principle. Only add environment variables that are necessary for the CI job to run. This way, even if the malicious code executes,
it will have limited access to sensitive information.

#### Safe-chain 

Safe-chain is a proxy for package managers that detects and prevents installation of packages with known vulnerabilities or malicious code. 
It works by intercepting package installation requests and checking them against a database of known issues. It works with many npm package managers,
and also with Python's pip. It can also block packages newer than a specified age.
It can be used in CI/CD builds, as well as locally on developer machines. 

[Read more about safe-chain](https://github.com/AikidoSec/safe-chain)

#### Monitor your dependencies
Use tools that can monitor your dependencies and alert you if there are any known vulnerabilities or if a new version of a package is published. 
This way, you can quickly identify and address any potential issues with your dependencies. For example:
- Renovate
- Dependabot

Renovate and Dependabot can automatically create pull requests to update dependencies when new versions are released, 
which can help to ensure that you are using the latest and most secure versions of your dependencies. Keep in mind that in the case of supply chain attacks, 
the latest version may not always be the safest, so sometimes it's better to wait a few days before installing a new version.

Or at least use simple checks like npm audit or composer audit.

#### Implement detection mechanisms
Use tools that can detect unusual behavior on servers, like unexpected read of environment variables. 
Here are some popular security detection tools:

- **[Falco](https://falco.org/)** - Monitors system calls and runtime behavior to detect suspicious activity in real-time on cloud-native applications
- **[Semgrep](https://semgrep.dev/)** - Scans your codebase for potential security issues and malicious patterns using static analysis
- **[OSSEC](https://www.ossec.net/)** - Provides host-based intrusion detection to monitor system logs and detect security threats
- **[Wazuh](https://wazuh.com/)** - Offers comprehensive security monitoring, threat detection, and incident response capabilities
- **[Snyk](https://snyk.io/)** - Scans for vulnerabilities in dependencies, container images, and infrastructure as code
- **[SonarQube](https://www.sonarsource.com/products/sonarqube/)** - Performs continuous code quality and security analysis to identify vulnerabilities, code smells, and security hotspots

Implement Canary tokens that can help to identify if a system has been compromised. 
Canary tokens are fake credentials or data that, when accessed, trigger an alert, 
so you know that someone has accessed your system.

### Conclusion

Supply chain attacks like Shai-Hulud demonstrate that security threats don't respect language boundaries. As this incident shows, 
even a PHP project can be compromised through Node.js tooling dependencies. This highlights a crucial lesson: **every dependency, 
regardless of its role in your project (whether it's a production dependency or just a development tool), represents a potential attack vector.**

The multi-layered defense approach outlined above, combining preventive measures (version pinning, script disabling), 
containment strategies (Docker isolation, minimal CI permissions), and detection mechanisms (monitoring tools, canary tokens) can 
significantly reduce both the likelihood and impact of such attacks. No single measure is perfect, but together they create a robust 
security posture.

Most importantly, this experience reinforced that security is an ongoing process, not a one-time setup. Stay informed about emerging threats, 
regularly audit your dependencies across all ecosystems in your project, and always apply the principle of least privilege. 
The few extra minutes spent on security measures today can save hours or days of incident response tomorrow.
