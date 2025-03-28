# Liberty Samples

In this folder, we can find sample projects with Jakarta EE and Liberty.

> [!NOTE]
> The `pom.xml` is used as a parent pom, providing common settings to handle the projects in a common way.

## How to use

> [!NOTE]
> These commands have to be executed within the sub projects.

To download and install Liberty, invoke Maven:

```bash
mvn \
  liberty:install-server \
  liberty:install-feature \
  dependency:copy-dependencies@copy-liberty-libraries \
  resources:copy-resources@copy-shared-liberty-config \
  -Pinclude-liberty
```

To build the project, simply invoke

```bash
mvn clean package
```

> [!TIP]
> You can build the application and install liberty in a single step by running
> `mvn clean package -Pinclude-liberty`.

To run the server with the project, use

```bash
# development mode (live reloading)
mvn liberty:dev
# normal mode
mvn liberty:start
# stop
mvn liberty:stop
```
