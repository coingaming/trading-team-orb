# TradeArt Orb Project

CircleCI orb for TradeArt project.

A starter template for orb projects. Build, test, and publish orbs automatically on CircleCI with [Orb-Tools](https://circleci.com/orbs/registry/orb/circleci/orb-tools).

Additional READMEs are available in each directory.

**Meta**: This repository is open for contributions! Feel free to open a pull request with your changes. Due to the nature of this repository, it is not built on CircleCI. The Resources and How to Contribute sections relate to an orb created with this template, rather than the template itself.

## Resources

[CircleCI Orb Registry Page](https://circleci.com/developer/orbs/orb/coingaming/trading-team-orb) - The official registry page of this orb for all versions, executors, commands, and jobs described.
[CircleCI Orb Docs](https://circleci.com/docs/2.0/orb-intro/#section=configuration) - Docs for using and creating CircleCI Orbs.

### How to Publish
Create and push a tag with new version to be released. Please use properly incremented version.


For further questions/comments about this or other orbs, visit the Orb Category of [CircleCI Discuss](https://discuss.circleci.com/c/orbs).

### How to pack your orb for validation or publishing a dev version

circleci orb pack src > orb.yml

### How to validate orb on your local via circleci cli

circleci orb validate orb.yml

### How to publish dev orb from your local

* Your orb will expire in 90 days unless a new version is published on the label `dev:alpha`

circleci orb publish orb.yml coingaming/trading-team-orb@dev:alpha