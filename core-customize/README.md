# SAP Commerce Project Template for CCv2

> **Initial project bootstrap**
>
> 1. Download the latest SAP Commerce 2211 release zip file and put it into the `dependencies` folder
>    using the correct file name, e.g.
>
>    ```bash
>    cp ~/Downloads/CXCOMCL221100U_23*.ZIP ./dependencies/hybris-commerce-suite-2211.23.zip
>    ```
>    *Or* configure your S-User (e.g. using `gradle.properties`) and run `./gradlew downloadAndVerifyPlatform`
> 2. Repeat the same for Integration extension pack if you want to use it.
> 3. Download the [cloudhotfolders ZIP] into the `dependencies` folder (save as `cloudhotfolders-2211.zip`). If you
>    don't need cloudhotfolders locally you can also remove them from the `build.gradle.kts` and the generated
>    `localextensions.xml` after the next step.
> 4. Bootstrap the starting point for your Commerce project by running the following command:
>
>    ```bash
>    ./gradlew -b bootstrap.gradle.kts \
>      -PprojectName=<name, e.g. coolshop> \
>      -ProotPackage=<package, e.g. com.cool.shop>
>    ```
>    **Read the output!**
>
>    The following optional settings are available:
>       - `-PintExtPackVersion=2102.1` enable "SAP Commerce Cloud, Integration Extension Pack" with version.
>       - `-PsolrVersion=9.2` set the solr version for the manifest. To check which versions are supported, see
>         Third-Party compatibility of the [Update Release Notes][update] of your selected version.
>       - `-PaccStorefrontEnabled` enable code generation for the deprecated accelerator storefront.
>
> 5. Review the generated configuration in `hybris/config`, especially the `hybris/config/environment/*.properties`
>    files and `localextensions.xml` (search for `TODO:` comments)
> 6. Update the `manifest.jsonnet` (again, search for `TODO:` comments).\
>    You can use the [jsonnet] file to update the `manifest.json` for your project.
> 7. Delete all bootstrap files, you don't need them any more:
>
>    ```bash
>    rm -r bootstrap*
>    ```
>
> 8. Delete this quote and the demo-section at the bottom of this README
> 9. Commit and push the changes to your project repository :)

[update]: https://help.sap.com/docs/SAP_COMMERCE_CLOUD_PUBLIC_CLOUD/75d4c3895cb346008545900bffe851ce/f18f6a711d07462b80137df6ed533eee.html?locale=en-US&q=Compatibility%20Matrix
[cloudhotfolders ZIP]: https://me.sap.com/notes/2817992

We use Gradle + [commerce-gradle-plugin][plugin] to automate whole project setup.

[plugin]: https://github.com/SAP/commerce-gradle-plugin

## Setup local development

```sh
git clone <project>
cd <project>
docker-compose up -d
cd core-customize
./gradlew setupLocalDevelopment
./gradlew yclean yall
./gradlew yinitialize
```

## FAQ

###  How to use manifest.jsonnet?

To generate the `manifest.json` with [Jsonnet][jsonnet]:

```bash
jsonnet --output-file manifest.json manifest.jsonnet
```

[jsonnet]: https://jsonnet.org/

### How do I add an addon to my storefront?

1. Add the addon to the `manifest.json` (either by hand or via `manifest.jsonnet`, [documentation][addon])
2. Run `./gradlew installManifestAddon`
3. Reformat `<storefront>/extensioninfo.xml` (unfortunately, the the platform build messes it up when adding addons)
4. Commit/push your changes
5. Tell your team to run `./gradlew installManifestAddon` after pulling your changes.

[addon]: https://help.sap.com/viewer/1be46286b36a4aa48205be5a96240672/LATEST/en-US/9a3ab7d08c704fccb7fd899e876d41d6.html

## Why does the configuration work for local development and the cloud?

By combining the [configuration reuse][reuse] mechanism of CCv2, the [optional configuration folder][folder]
of Commerce and a bit of clever symlinking of files and folders, we can use the same configuration
locally and in the cloud.

This setup uses:

- `hybris/config/localextensions.xml` to configure extensions
- `hybris/config/cloud/**/*.properties` to configure properties per CCv2 aspect and/or persona.
   There is one file per aspect, plus the special file `local-dev.properties` that configures the local development environment.
- `hybris/config/local-config` is configured as `hybris.optional.config.dir` and contains *symlinks* 
  to the relevant property files in `hybris/config/cloud` (by default: `common.properties`, `persona/development.properties` and `local-dev.properties`).\
  **Important** `local.properties` must not be modified at all (that's why it is in `.gitignore`).
  - If you have any configuration specific to your local machine, put it in `hybris/config/local-config/99-local.properties`.
  - If the local setup changes for the whole project, update `hybris/config/cloud/local-dev.properties`
- If you enabled solr customization during bootstrap (`./gradle -b boostrap.gradle.kts enableSolrCustomization`), the default cloud solr configuration set is moved to the correct folder structure for CCv2 ([documentation][solr]).
  A symlink in `hybris/config/solr` allows you to use the same configuration locally.

```
                                  core-customize
                                  ├── ...
                                  ├── hybris
                                  ├── ...
                                  │  ├── config
                                  │  │  ├── cloud
                        +--------------------> accstorefront.properties
                        +--------------------> admin.properties
                        +--------------------> api.properties
                        +--------------------> backgroundprocessing.properties
                        +--------------------> backoffice.properties
                        +--------------------> common.properties     <---+
                        |         │  │  │  └── local-dev.properties <--+ |
                        |         │  │  ├── ...                        | | symlinks
                        |         │  │  ├── local-config               | |
manifest.json           |         │  │  │  ├── 10-local.properties +-----+
  useConfig             |         │  │  │  ├── 50-local.properties +---+
    properties          |         │  │  │  └── 99-local.properties
      ... +-------------+         │  │  ├── local.properties
    extensions +--------------------------> localextensions.xml
    solr +---------+              │  │  ├── readme.txt
                   |              │  │  ├── solr
                   |              │  │  │  └── instances
                   |              │  │  │     └── cloud
                   |              │  │  │        ├── configsets +-------+
                   |              │  │  │        ├── ...                |
                   |              │  │  │        └── zoo.cfg            |
                   |              │  │  └── ...                         |
                   |              ├── ...                               |  symlink
                   +----------------> solr                              |
                                  │  └── server                         |
                                  │     └── solr                        |
                                  │        └── configsets <-------------+
                                  │           └── default
                                  │              └── conf
                                  │                 ├── lang
                                  │                 ├── protwords.txt
                                  │                 ├── schema.xml
                                  │                 ├── solrconfig.xml
                                  │                 ├── stopwords.txt
                                  │                 └── synonyms.txt
                                  └── ...

```

[reuse]: https://help.sap.com/viewer/1be46286b36a4aa48205be5a96240672/LATEST/en-US/2311d89eef9344fc81ef168ac9668307.html
[folder]: https://help.sap.com/viewer/b490bb4e85bc42a7aa09d513d0bcb18e/LATEST/en-US/8beb75da86691014a0229cf991cb67e4.html
[solr]: https://help.sap.com/viewer/b2f400d4c0414461a4bb7e115dccd779/LATEST/en-US/f7251d5a1d6848489b1ce7ba46300fe6.html

## Demo Setup

The file `bootstrap-demo.gradle.kts` bootstraps a demo storefront based on the `cx` [recipe][recipe],
including the `spartacussampledata` extension (necessary to demo the Spartacus storefront; [documentation][spartacussample])

To generate the demo, run:
```
./gradlew -b bootstrap-demo.gradle.kts
```
[spartacussample]: https://sap.github.io/spartacus-docs/spartacussampledata-extension/
[recipe]: https://help.sap.com/viewer/a74589c3a81a4a95bf51d87258c0ab15/2011/en-US/f09d46cf4a2546b586ed7021655e4715.html
