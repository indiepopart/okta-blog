---
layout: blog_post
title: "How to build a GraphQL API with Spring for GraphQL"
author: jimena-garbarino
by: contractor
communities: [security,java,javascript]
description: "A step-by-step guide for building a secured GraphQL API with Spring Boot and Auth0 authentication on React"
tags: [java, javascript]
tweets:
- ""
- ""
- ""
image:
type: awareness
github: https://github.com/oktadev/auth0-spring-graphql-react-example
---

GraphQL is a query language for APIs and a runtime that allows the API consumer to get exactly the required information instead of the server exclusively controlling the contents of the response. While some REST API implementations require loading the references of a resource from multiple URLs, GraphQL APIs can follow references between related objects and return them in a single request.

This step-by-step guide demonstrates how to build a GraphQL API with Spring Boot and Spring for GraphQL, for querying a sample dataset of related companies, persons, and properties, seeded to a Neo4j database. It also demonstrates how to build a React client with Next.js and MUI Datagrid to consume the API. Both the client and the server are secured with Auth0 authentication using Okta Spring Boot Starter for the server and Auth0 React SDK for the client.

{% img blog/spring-graphql-react/logos.png alt:"Spring, GraphQL and React" width:"700" %}{: .center-image }

> This example was created with the following tools and services:
> - [Node.js v18.16.1](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
> - [npm 9.5.1](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
> - [Java OpenJDK 17](https://jdk.java.net/java-se-ri/17)
> - [Docker 24.0.2](https://docs.docker.com/desktop/)
> - [Auth0 account](https://auth0.com/signup)
> - [Auth0 CLI 1.0.0](https://github.com/auth0/auth0-cli#installation)
> - [HTTPie 3.2.2](https://httpie.io/)
> - [Next.js 13.4.19](https://nextjs.org/)

{% include toc.md %}

## Build a GraphQL API with Spring for GraphQL

The resource server is a Spring Boot web application that exposes a GraphQL API with Spring for GraphQL. The API allows querying a Neo4j database containing information about companies and their related owners and properties, using Spring Data Neo4j. The data was obtained from a Neo4j [use case example](https://neo4j.com/graphgists/35a813ba-ea10-4165-9065-84f8802cbae8/).

Create the application with Spring Initializr and HTTPie:

```shell
https start.spring.io/starter.zip \
  bootVersion==3.1.3 \
  language==java \
  packaging==jar \
  javaVersion==17 \
  type==gradle-project \
  dependencies==data-neo4j,graphql,web \
  groupId==com.okta.developer \
  artifactId==spring-graphql  \
  name=="Spring Boot GraphQL API Application" \
  description=="Demo project of a Spring Boot GraphQL API" \
  packageName==com.okta.developer.demo > spring-graphql-api.zip
```

Unzip the file and start editing the project. Define the GraphQL API with a schema file named `schema.graphqls` in the `src/main/resources/graphql` directory:

__schema.graphqls__
```graphql
type Query {
    companyList(page: Int): [Company!]!
    companyCount: Int
}

type Company {
    id: ID
    SIC: String
    category: String
    companyNumber: String
    countryOfOrigin: String
    incorporationDate: String
    mortgagesOutstanding: Int
    name: String
    status: String
    controlledBy: [Person!]!
    owns: [Property!]!
}

type Person {
    id: ID
    birthMonth: String
    birthYear: String
    nationality: String
    name: String
    countryOfResidence: String
}

type Property {
    id: ID
    address: String
    county: String
    district: String
    titleNumber: String
}
```

As you can see the schema defines the object types `Company`, `Person` and `Property` and the query types `companyList` and `companyCount`.

Start adding classes for the domain. Create the package `com.okta.developer.demo.domain` under `src/main/java`. Add the classes `Person`, `Property` and `Company`.

__Person.java__
```java
package com.okta.developer.demo.domain;

import org.springframework.data.neo4j.core.schema.GeneratedValue;
import org.springframework.data.neo4j.core.schema.Id;
import org.springframework.data.neo4j.core.schema.Node;

@Node
public class Person {

    @Id @GeneratedValue
    private Long id;

    private String birthMonth;
    private String birthYear;
    private String countryOfResidence;

    private String name;
    private String nationality;

    public Person(String birthMonth, String birthYear, String countryOfResidence, String name, String nationality) {
        this.id = null;
        this.birthMonth = birthMonth;
        this.birthYear = birthYear;
        this.countryOfResidence = countryOfResidence;
        this.name = name;
        this.nationality = nationality;
    }

    public Person withId(Long id) {
        if (this.id.equals(id)) {
            return this;
        } else {
            Person newObject = new Person(this.birthMonth, this.birthYear, this.countryOfResidence, this.name, this.nationality);
            newObject.id = id;
            return newObject;
        }
    }

    public String getBirthMonth() {
        return birthMonth;
    }

    public void setBirthMonth(String birthMonth) {
        this.birthMonth = birthMonth;
    }

    public String getBirthYear() {
        return birthYear;
    }

    public void setBirthYear(String birthYear) {
        this.birthYear = birthYear;
    }

    public String getCountryOfResidence() {
        return countryOfResidence;
    }

    public void setCountryOfResidence(String countryOfResidence) {
        this.countryOfResidence = countryOfResidence;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getNationality() {
        return nationality;
    }

    public void setNationality(String nationality) {
        this.nationality = nationality;
    }

    public Long getId() {
        return this.id;
    }
}
```

__Property.java__
```java
package com.okta.developer.demo.domain;

import org.springframework.data.neo4j.core.schema.GeneratedValue;
import org.springframework.data.neo4j.core.schema.Id;
import org.springframework.data.neo4j.core.schema.Node;

@Node
public class Property {

    @Id
    @GeneratedValue  private Long id;
    private String address;
    private String county;
    private String district;
    private String titleNumber;

    public Property(String address, String county, String district, String titleNumber) {
        this.id = null;
        this.address = address;
        this.county = county;
        this.district = district;
        this.titleNumber = titleNumber;
    }

    public Property withId(Long id) {
        if (this.id.equals(id)) {
            return this;
        } else {
            Property newObject = new Property(this.address, this.county, this.district, this.titleNumber);
            newObject.id = id;
            return newObject;
        }
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public String getCounty() {
        return county;
    }

    public void setCounty(String county) {
        this.county = county;
    }

    public String getDistrict() {
        return district;
    }

    public void setDistrict(String district) {
        this.district = district;
    }

    public String getTitleNumber() {
        return titleNumber;
    }

    public void setTitleNumber(String titleNumber) {
        this.titleNumber = titleNumber;
    }
}
```

__Company.java__
```java
package com.okta.developer.demo.domain;

import org.springframework.data.neo4j.core.schema.GeneratedValue;
import org.springframework.data.neo4j.core.schema.Id;
import org.springframework.data.neo4j.core.schema.Node;
import org.springframework.data.neo4j.core.schema.Relationship;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

@Node
public class Company {
    @Id
    @GeneratedValue
    private Long id;
    private String SIC;
    private String category;
    private String companyNumber;
    private String countryOfOrigin;
    private LocalDate incorporationDate;
    private Integer mortgagesOutstanding;
    private String name;
    private String status;

    // Mapped automatically
    private List<Property> owns = new ArrayList<>();

    @Relationship(type = "HAS_CONTROL", direction = Relationship.Direction.INCOMING)
    private List<Person> controlledBy = new ArrayList<>();

    public Company(String SIC, String category, String companyNumber, String countryOfOrigin, LocalDate incorporationDate, Integer mortgagesOutstanding, String name, String status) {
        this.id = null;
        this.SIC = SIC;
        this.category = category;
        this.companyNumber = companyNumber;
        this.countryOfOrigin = countryOfOrigin;
        this.incorporationDate = incorporationDate;
        this.mortgagesOutstanding = mortgagesOutstanding;
        this.name = name;
        this.status = status;
    }

    public Company withId(Long id) {
        if (this.id.equals(id)) {
            return this;
        } else {
            Company newObject = new Company(this.SIC, this.category, this.companyNumber, this.countryOfOrigin, this.incorporationDate, this.mortgagesOutstanding, this.name, this.status);
            newObject.id = id;
            return newObject;
        }
    }

    public String getSIC() {
        return SIC;
    }

    public void setSIC(String SIC) {
        this.SIC = SIC;
    }

    public String getCategory() {
        return category;
    }

    public void setCategory(String category) {
        this.category = category;
    }

    public String getCompanyNumber() {
        return companyNumber;
    }

    public void setCompanyNumber(String companyNumber) {
        this.companyNumber = companyNumber;
    }

    public String getCountryOfOrigin() {
        return countryOfOrigin;
    }

    public void setCountryOfOrigin(String countryOfOrigin) {
        this.countryOfOrigin = countryOfOrigin;
    }

    public LocalDate getIncorporationDate() {
        return incorporationDate;
    }

    public void setIncorporationDate(LocalDate incorporationDate) {
        this.incorporationDate = incorporationDate;
    }

    public Integer getMortgagesOutstanding() {
        return mortgagesOutstanding;
    }

    public void setMortgagesOutstanding(Integer mortgagesOutstanding) {
        this.mortgagesOutstanding = mortgagesOutstanding;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}
```

Create the package `com.okta.developer.demo.repository` and the class `CompanyRepository`:

__CompanyRepository.java__
```java
package com.okta.developer.demo.repository;

import com.okta.developer.demo.domain.Company;
import org.springframework.data.neo4j.repository.ReactiveNeo4jRepository;

public interface CompanyRepository extends ReactiveNeo4jRepository<Company, Long> {

}
```

Create the configuration class `GraphQLConfig` under the root package. This class will enable CORS from the React client and log the GraphQL schema mappings:

__GraphQLConfig.java__
```java
package com.okta.developer.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.autoconfigure.graphql.GraphQlSourceBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
class GraphQLConfig {

    private static Logger logger = LoggerFactory.getLogger("graphql");

    @Bean
    public GraphQlSourceBuilderCustomizer sourceBuilderCustomizer() {
        return (builder) ->
                builder.inspectSchemaMappings(report -> {
                    logger.debug(report.toString());
                });
    }
}
```

Create a configuration class named `SpringBootGraphQlApiConfig` in the root package as well, defining a reactive transaction manager required for reactive Neo4j:

__SpringBootGraphQlApiConfig.java__
```java
package com.okta.developer.demo;

import org.neo4j.driver.Driver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.neo4j.core.ReactiveDatabaseSelectionProvider;
import org.springframework.data.neo4j.core.transaction.ReactiveNeo4jTransactionManager;
import org.springframework.data.neo4j.repository.config.ReactiveNeo4jRepositoryConfigurationExtension;
import org.springframework.transaction.ReactiveTransactionManager;

@Configuration
public class SpringBootGraphQlApiConfig {

    @Bean(ReactiveNeo4jRepositoryConfigurationExtension.DEFAULT_TRANSACTION_MANAGER_BEAN_NAME) //Required for neo4j
    public ReactiveTransactionManager reactiveTransactionManager(
            Driver driver,
            ReactiveDatabaseSelectionProvider databaseNameProvider) {
        return new ReactiveNeo4jTransactionManager(driver, databaseNameProvider);
    }
}
```

Create the package `com.okta.developer.demo.controller` and the class `CompanyController` implementing the query endpoints matching the queries defined in the graphql schema:

__CompanyController.java__
```java
package com.okta.developer.demo.controller;

import com.okta.developer.demo.domain.Company;
import com.okta.developer.demo.repository.CompanyRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.graphql.data.method.annotation.Argument;
import org.springframework.graphql.data.method.annotation.QueryMapping;
import org.springframework.stereotype.Controller;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Controller
public class CompanyController {

    @Autowired
    private CompanyRepository companyRepository;

    @QueryMapping
    public Flux<Company> companyList(@Argument Long page) {
        return companyRepository.findAll().skip(page * 10).take(10);
    }

    @QueryMapping
    public Mono<Long> companyCount() {
        return companyRepository.count();
    }
}
```

Create a `CompanyControllerTests` for the web layer in the folder `src/main/test/java` under the package `com.okta.developer.demo.controller`:

__CompanyControllerTests.java__
```java
package com.okta.developer.demo.controller;

import com.okta.developer.demo.domain.Company;
import com.okta.developer.demo.repository.CompanyRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.graphql.GraphQlTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.graphql.test.tester.GraphQlTester;
import reactor.core.publisher.Flux;

import java.time.LocalDate;

import static org.mockito.Mockito.when;

@GraphQlTest(CompanyController.class)
public class CompanyControllerTests {

    @Autowired
    private GraphQlTester graphQlTester;

    @MockBean
    private CompanyRepository companyRepository;

    @Test
    void shouldGetCompanies() {

        when(this.companyRepository.findAll())
                .thenReturn(Flux.just(new Company(
                        "1234",
                        "private",
                        "12345678",
                        "UK",
                        LocalDate.of(2020, 1, 1),
                        0,
                        "Test Company",
                        "active")));

        this.graphQlTester
                .documentName("companyList")
                .variable("page", 0)
                .execute()
                .path("companyList")
                .matchesJson("""
                    [{
                        "id": null,
                        "SIC": "1234",
                        "name": "Test Company",
                        "status": "active",
                        "category": "private",
                        "companyNumber": "12345678",
                        "countryOfOrigin": "UK"
                    }]
                """);
    }
}
```

Create the document file `companyList.graphql` containing the query definition for the test, in the folder `src/main/test/resources/graphql-test`:

__companyList.graphql__
```graphql
query companyList($page: Int) {
    companyList(page: $page) {
        id
        SIC
        name
        status
        category
        companyNumber
        countryOfOrigin
    }
}
```

Update the test configuration in `build.gradle` file, so passed tests are logged:

__build.gradle__
```groovy
tasks.named('test') {
    useJUnitPlatform()

    testLogging {
        // set options for log level LIFECYCLE
        events "failed", "passed"
    }
}
```

Run the test with:

```shell
./gradlew test
```

You should see logs for the successful tests:

```
...
SpringBootGraphQlApiApplicationTests > contextLoads() PASSED

CompanyControllerTests > shouldGetCompanies() PASSED
...
```


### Add Neo4j seed data

Let's add Neo4j migrations dependency for the seed data insertion. Edit the `build.gradle` file and add:

__build.gradle__
```groovy
dependencies {
    ...
    implementation 'eu.michael-simons.neo4j:neo4j-migrations-spring-boot-starter:2.5.3'
    ...
}
```

Create the folder `src/main/resources/neo4j/migrations` and the following migration files:

{% raw %}
__V001__Constraint.cypher__
```cypher
CREATE CONSTRAINT FOR (c:Company) REQUIRE c.companyNumber IS UNIQUE;
//Constraint for a node key is a Neo4j Enterprise feature only - run on an instance with enterprise
//CREATE CONSTRAINT ON (p:Person) ASSERT (p.birthMonth, p.birthYear, p.name) IS NODE KEY
CREATE CONSTRAINT FOR (p:Property) REQUIRE p.titleNumber IS UNIQUE;
```

__V002__Company.cypher__
```cypher
LOAD CSV WITH HEADERS FROM "file:///PSCAmericans.csv" AS row
MERGE (c:Company {companyNumber: row.company_number})
RETURN COUNT(*);
```

__V003__Person.cypher__
```cypher
LOAD CSV WITH HEADERS FROM "file:///PSCAmericans.csv" AS row
MERGE (p:Person {name: row.`data.name`, birthYear: row.`data.date_of_birth.year`, birthMonth: row.`data.date_of_birth.month`})
  ON CREATE SET p.nationality = row.`data.nationality`,
  p.countryOfResidence = row.`data.country_of_residence`
RETURN COUNT(*);
```

__V004__PersonCompany.cypher__
```cypher
LOAD CSV WITH HEADERS FROM "file:///PSCAmericans.csv" AS row
MATCH (c:Company {companyNumber: row.company_number})
MATCH (p:Person {name: row.`data.name`, birthYear: row.`data.date_of_birth.year`, birthMonth: row.`data.date_of_birth.month`})
MERGE (p)-[r:HAS_CONTROL]->(c)
SET r.nature = split(replace(replace(replace(row.`data.natures_of_control`, "[",""),"]",""),  '"', ""), ",")
RETURN COUNT(*);
```

__V005__CompanyData.cypher__
```cypher
LOAD CSV WITH HEADERS FROM "file:///CompanyDataAmericans.csv" AS row
MATCH (c:Company {companyNumber: row.` CompanyNumber`})
SET c.name = row.CompanyName,
c.mortgagesOutstanding = toInteger(row.`Mortgages.NumMortOutstanding`),
c.incorporationDate = Date(Datetime({epochSeconds: apoc.date.parse(row.IncorporationDate,'s','dd/MM/yyyy')})),
c.SIC = row.`SICCode.SicText_1`,
c.countryOfOrigin = row.CountryOfOrigin,
c.status = row.CompanyStatus,
c.category = row.CompanyCategory;
```

__V006__Land.cypher__
```cypher
LOAD CSV WITH HEADERS FROM "file:///LandOwnershipAmericans.csv" AS row
MATCH (c:Company {companyNumber: row.`Company Registration No. (1)`})
MERGE (p:Property {titleNumber: row.`Title Number`})
SET p.address = row.`Property Address`,
p.county  = row.County,
p.price   = toInteger(row.`Price Paid`),
p.district = row.District
MERGE (c)-[r:OWNS]->(p)
WITH row, c,r,p WHERE row.`Date Proprietor Added` IS NOT NULL
SET r.date = Date(Datetime({epochSeconds: apoc.date.parse(row.`Date Proprietor Added`,'s','dd-MM-yyyy')}));
CREATE INDEX FOR (c:Company) ON c.incorporationDate;
```
{% endraw %}

Update `application.properties` and add the following properties:

__application.properties__
```properties
...
spring.graphql.graphiql.enabled=true
spring.graphql.schema.introspection.enabled=true
org.neo4j.migrations.transaction-mode=PER_STATEMENT
spring.neo4j.uri=bolt://localhost:7687
spring.neo4j.authentication.username=neo4j

spring.graphql.cors.allowed-origins=http://localhost:3000
```

Create a `.env` file in the project root to store the Neo4j credentials:

__.env__
```shell
export SPRING_NEO4J_AUTHENTICATION_PASSWORD=verysecret
```

Download the following seed files to some folder:

- [CompanyDataAmericans](https://guides.neo4j.com/ukcompanies/data/CompanyDataAmericans.csv)
- [LandOwnershipAmericans](https://guides.neo4j.com/ukcompanies/data/LandOwnershipAmericans.csv)
- [PSCAmericans.csv](https://guides.neo4j.com/ukcompanies/data/PSCAmericans.csv)

Create the folder `src/main/docker` and create a file `neo4j.yml` there, with the following content:

__neo4j.yml__
```yml
# This configuration is intended for development purpose, it's **your** responsibility to harden it for production
name: companies
services:
  neo4j:
    image: neo4j:5
    volumes:
      - <csv-folder>:/var/lib/neo4j/import
    environment:
      - NEO4J_AUTH=neo4j/${NEO4J_PASSWORD}
      - NEO4JLABS_PLUGINS=["apoc"]
    # If you want to expose these ports outside your dev PC,
    # remove the "127.0.0.1:" prefix
    ports:
      - '127.0.0.1:7474:7474'
      - '127.0.0.1:7687:7687'
    healthcheck:
      test: ['CMD', 'wget', 'http://localhost:7474/', '-O', '-']
      interval: 5s
      timeout: 5s
      retries: 10
```

Create the file `src/main/docker/.env` with the following content:

__.env__
```dotenv
NEO4J_PASSWORD=verysecret
```

As you can see the compose file will mount `<csv-folder>` to a `/var/lib/neo4j/import` volume, making the content accessible from the running neo4j container.
Replace `<csv-folder>` with the path to the CSV files downloaded before.

In a terminal, go to the `docker` folder and run:

```shell
docker compose -f neo4j.yml up
```

### Run the API server

Go to the project root folder and start the application with:

```shell
source .env && ./gradlew bootRun
```

Wait for the logs to inform the seed data migrations have run (it might take a while):

```
2023-08-02T13:06:14.386-03:00  INFO 28673 --- [           main] a.s.neo4j.migrations.core.Migrations     : Applied migration 001 ("Constraint").
2023-08-02T13:06:23.379-03:00  INFO 28673 --- [           main] a.s.neo4j.migrations.core.Migrations     : Applied migration 002 ("Company").
2023-08-02T13:11:23.693-03:00  INFO 28673 --- [           main] a.s.neo4j.migrations.core.Migrations     : Applied migration 003 ("Person").
2023-08-02T13:21:03.680-03:00  INFO 28673 --- [           main] a.s.neo4j.migrations.core.Migrations     : Applied migration 004 ("PersonCompany").
2023-08-02T13:21:06.519-03:00  INFO 28673 --- [           main] a.s.neo4j.migrations.core.Migrations     : Applied migration 005 ("CompanyData").
2023-08-02T13:21:06.551-03:00  INFO 28673 --- [           main] a.s.neo4j.migrations.core.Migrations     : Applied migration 006 ("Land").
```

Test the API with GraphiQL at `http://localhost:8080/graphiql`. In the query box on the left, paste the following query:

```graphql
{
    companyList(page: 20) {
        id
        SIC
        name
        status
        category
        companyNumber
        countryOfOrigin
    }
}
```

You should see the query output in the box on the right:

{% img blog/spring-graphql-react/graphiql-test.png alt:"GraphiQL example" width:"900" %}{: .center-image }


> __NOTE:__
> If you see a warning message in the server logs, that reads _The query used a deprecated function: id_, you can ignore it, Spring Data Neo4j still [behaves correctly](https://github.com/spring-projects/spring-data-neo4j/issues/2716).


## Build a React client

Now let's create a Single Page Application (SPA) to consume the GraphQL API with React and Next.js. The list of companies will display in a [MUI](https://mui.com/material-ui/getting-started/) [Data Grid](https://mui.com/x/react-data-grid/) component. The application will use Next.js' `app` router. The `src/app` folder will only contain routing files, and the UI components and application code will be in other folders.

Install Node and in a terminal run:

```shell
npx create-next-app@13.4.19
```

Answer the questions as follows:

```
✔ What is your project named? ... react-graphql
✔ Would you like to use TypeScript? ... Yes
✔ Would you like to use ESLint? ... Yes
✔ Would you like to use Tailwind CSS? ... No
✔ Would you like to use `src/` directory? ... Yes
✔ Would you like to use App Router? (recommended) ... Yes
✔ Would you like to customize the default import alias? ... No
```

Then add the the MUI Datagrid dependencies and custom hooks from Vercel:

```shell
cd react-graphql && \
  npm install @mui/x-data-grid && \
  npm install @mui/material@5.14.5 @emotion/react @emotion/styled && \
  npm install react-use-custom-hooks
```

Run the application with:

```shell
npm run dev
```

Navigate to `http://localhost:3000` and you should see the default Next.js page:

{% img blog/spring-graphql-react/nextjs-default.png alt:"Next.js default page" width:"800" %}{: .center-image }

### Create the API client

Create the folder `src/services` and add the file `base.tsx` with the following code:

{% raw %}
__base.tsx__
```tsx
import axios from "axios";

export const backendAPI = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_SERVER_URL
});

export default backendAPI;
```
{% endraw %}

Add the file `src/services/companies.tsx` with the following content:

{% raw %}
__companies.tsx__
```tsx
import { AxiosError } from "axios";
import { backendAPI } from "./base";

export type CompaniesQuery = {
  page: number;
};

export type CompanyDTO = {
  name: string;
  SIC: string;
  id: string;
  companyNumber: string;
  category: string;
};

export const CompanyApi = {

  getCompanyCount: async () => {
    try {
      const response = await backendAPI.post("/graphql", {
        query: `{
        companyCount
      }`,
      });
      return response.data.data.companyCount as number;
    } catch (error) {
      console.log("handle get company count error", error);
      if (error instanceof AxiosError) {
        let axiosError = error as AxiosError;
        if (axiosError.response?.data) {
          throw new Error(axiosError.response?.data as string);
        }
      }
      throw new Error("Unknown error, please contact the administrator");
    }
  },

  getCompanyList: async (params?: CompaniesQuery) => {
    try {
      const response = await backendAPI.post("/graphql", {
        query: `{
        companyList(page: ${params?.page || 0}) {
          name,
          SIC,
          id,
          companyNumber,
          category
        }}`,
      });
      return response.data.data.companyList as CompanyDTO[];
    } catch (error) {
      console.log("handle get companies error", error);
      if (error instanceof AxiosError) {
        let axiosError = error as AxiosError;
        if (axiosError.response?.data) {
          throw new Error(axiosError.response?.data as string);
        }
      }
      throw new Error("Unknown error, please contact the administrator");
    }
  },

};
```
{% endraw %}

Add a file `.env.example` and `.env.local` in the root folder, both with the following content:

```shell
NEXT_PUBLIC_API_SERVER_URL=http://localhost:8080
```

### Create a companies home page

Create the folder `src/components/company` and add the file `CompanyTable.tsx` with the following content:

{% raw %}
__CompanyTable.tsx__
```tsx
import { DataGrid, GridColDef, GridEventListener, GridPaginationModel } from "@mui/x-data-grid";

export interface CompanyData {
  id: string,
  name: string,
  category: string,
  companyNumber: string,
  SIC: string
}

export interface CompanyTableProps {
  rowCount: number,
  rows: CompanyData[],
  columns: GridColDef[],
  pagination: GridPaginationModel,
  onRowClick?: GridEventListener<"rowClick">
  onPageChange?: (pagination: GridPaginationModel) => void,

}

const CompanyTable = (props: CompanyTableProps) => {

  return (
    <>
      <DataGrid
        rowCount={props.rowCount}
        rows={props.rows}
        columns={props.columns}
        pageSizeOptions={[props.pagination.pageSize ]}
        initialState={{
          pagination: {
            paginationModel: { page: props.pagination.page, pageSize: props.pagination.pageSize },
          },
        }}
        density="compact"
        disableColumnMenu={true}
        disableRowSelectionOnClick={true}
        disableColumnFilter={true}
        disableDensitySelector={true}
        paginationMode="server"
        onRowClick={props.onRowClick}
        onPaginationModelChange={props.onPageChange}
      />
    </>
  );
};

export default CompanyTable;
```
{% endraw %}

Create a `Loader.tsx` component in the folder `src/components/loader` with the following code:

{% raw %}
__Loader.tsx__
```tsx
import { Box, CircularProgress, Skeleton } from "@mui/material";

const Loader = () => {
  return (
    <Box sx={{ display: 'flex', justifyContent: 'center', alignItems: 'center', height: 200 }}>
      <CircularProgress />
    </Box>
  );
}

export default Loader;
```
{% endraw %}

Add the file `src/components/company/CompanyTableContainer.tsx`:

{% raw %}
__CompanyTableContainer.tsx__
```tsx
import { GridColDef, GridPaginationModel } from "@mui/x-data-grid";
import CompanyTable from "./CompanyTable";
import { usePathname, useRouter, useSearchParams } from "next/navigation";
import { CompanyApi } from "@/services/companies";
import Loader from "../loader/Loader";
import { useAsync } from "react-use-custom-hooks";

interface CompanyTableProperties {
  page?: number;
}

const columns: GridColDef[] = [
  { field: "id", headerName: "ID", width: 70 },
  {
    field: "companyNumber",
    headerName: "Company #",
    width: 100,
    sortable: false,
  },
  { field: "name", headerName: "Company Name", width: 350, sortable: false },
  { field: "category", headerName: "Category", width: 200, sortable: false },
  { field: "SIC", headerName: "SIC", width: 400, sortable: false },
];

const CompanyTableContainer = (props: CompanyTableProperties) => {
  const router = useRouter();
  const searchParams = useSearchParams()!;
  const pathName = usePathname();
  const page = props.page ? props.page : 1;

  const [dataList, loadingList, errorList] = useAsync(
    () => CompanyApi.getCompanyList({ page: page - 1 }),
    {},
    [page]
  );
  const [dataCount] = useAsync(() => CompanyApi.getCompanyCount(), {}, []);

  const onPageChange = (pagination: GridPaginationModel) => {
    const params = new URLSearchParams(searchParams.toString());
    const page = pagination.page + 1;
    params.set("page", page.toString());
    router.push(pathName + "?" + params.toString());
  };

  return (
    <>
      {loadingList && <Loader />}
      {errorList && <div>Error</div>}

      {!loadingList && dataList && (
        <CompanyTable
          pagination={{ page: page - 1, pageSize: 10 }}
          rowCount={dataCount}
          rows={dataList}
          columns={columns}
          onPageChange={onPageChange}
        ></CompanyTable>
      )}
    </>
  );
};

export default CompanyTableContainer;
```
{% endraw %}

Add `src/app/HomePage.tsx` for the homepage:

{% raw %}
__HomePage.tsx__
```tsx
"use client";

import CompanyTableContainer from "@/components/company/CompanyTableContainer";
import { Box, Typography } from "@mui/material";
import { useSearchParams } from "next/navigation";

const HomePage = () => {
  const searchParams = useSearchParams();
  const page = searchParams.get("page")
    ? parseInt(searchParams.get("page") as string)
    : 1;

  return (
    <>
      <Box>
        <Typography variant="h4" component="h1">
          Companies
        </Typography>
      </Box>
      <Box mt={2}>
        <CompanyTableContainer page={page}></CompanyTableContainer>
      </Box>
    </>
  );
};

export default HomePage;
```
{% endraw %}

Replace the contents of `src/app/page.tsx` and change it to render the `HomePage` component:

{% raw %}
__app/page.tsx__
```tsx
import HomePage from "./HomePage";

const Page = () => {
  return (
    <HomePage></HomePage>
  );
}

export default Page;
```
{% endraw %}

Add a component defining the page width, for using it in the root layout. Create `src/layout/WideLayout.tsx` with the following content:

{% raw %}
__WideLayout.tsx__
```tsx
"use client";

import { Container, ThemeProvider, createTheme } from "@mui/material";

const theme = createTheme({
  typography: {
    fontFamily: "inherit",
  },
});

const WideLayout = (props: { children: React.ReactNode }) => {
  return (
    <ThemeProvider theme={theme}>
      <Container maxWidth="lg" sx={{ mt: 4 }}>
        {props.children}
      </Container>
    </ThemeProvider>
  );
};

export default WideLayout;
```
{% endraw %}

With the implementation above, the page content will be wrapped in a `ThemeProvider` component, so [MUI child components inherit the font family](https://github.com/vercel/next.js/discussions/45433) from the root layout.
Update the contents of `src/app/layout.tsx` to be:

{% raw %}
__app/layout.tsx__
```tsx
import WideLayout from "@/layout/WideLayout";
import { Ubuntu} from "next/font/google";

const font = Ubuntu({
  subsets: ['latin'],
  weight: ['300','400','500','700'],
});

export const metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={font.className}>
        <WideLayout>{children}</WideLayout>
      </body>
    </html>
  );
}
```
{% endraw %}

Also, remove `src/app/globals.css` and `src/app/page.module.css`. Then run the client application with:

```shell
npm run dev
```

Navigate to `http://localhost:3000` and you should see the companies list:

{% img blog/spring-graphql-react/react-datagrid.png alt:"Home page companies datagrid" width:"900" %}{: .center-image }

## Add security with Auth0

For securing both the server and client, the Auth0 platform provides the best customer experience, and with a few simple configuration steps, you can add authentication to your applications. Sign up at [Auth0](https://auth0.com/signup) and install the [Auth0 CLI](https://github.com/auth0/auth0-cli) that will help you create the tenant and the client applications.

### Add resource server security to the GraphQL API server

In the command line login to Auth0 with the CLI:

```shell
auth0 login
```

The command output will display a device confirmation code and open a browser session to activate the device.

> __NOTE:__
> My browser was not displaying anything, so I had to manually activate the device by opening the URL `https://auth0.auth0.com/activate?user_code={deviceCode}`.

On successful login, you will see the tenant, which you will use as the issuer later:

```
✪ Welcome to the Auth0 CLI 🎊

If you don't have an account, please create one here: https://auth0.com/signup.

Your device confirmation code is: KGFL-LNVB

 ▸    Press Enter to open the browser to log in or ^C to quit...

Waiting for the login to complete in the browser... ⣻Opening in existing browser session.
Waiting for the login to complete in the browser... done

 ▸    Successfully logged in.
 ▸    Tenant: dev-avup2laz.us.auth0.com
```

The next step is to create a client app, which you can do in one command:

```shell
auth0 apps create \
  --name "GraphQL Server" \
  --description "Spring Boot GraphQL Resource Server" \
  --type regular \
  --callbacks http://localhost:8080/login/oauth2/code/okta \
  --logout-urls http://localhost:8080 \
  --reveal-secrets
```

Once the app is created, you will see the OIDC app's configuration:

```
=== dev-avup2laz.us.auth0.com application created

  CLIENT ID            ***
  NAME                 GraphQL Server
  DESCRIPTION          Spring Boot GraphQL Resource Server
  TYPE                 Regular Web Application
  CLIENT SECRET        ***
  CALLBACKS            http://localhost:8080/login/oauth2/code/okta
  ALLOWED LOGOUT URLS  http://localhost:8080
  ALLOWED ORIGINS
  ALLOWED WEB ORIGINS
  TOKEN ENDPOINT AUTH
  GRANTS               implicit, authorization_code, refresh_token, client_credentials

 ▸    Quickstarts: https://auth0.com/docs/quickstart/webapp
 ▸    Hint: Emulate this app's login flow by running `auth0 test login ***`
 ▸    Hint: Consider running `auth0 quickstarts download ***`
```

Add the `okta-spring-boot-starter` dependency to the `build.gradle` file in the spring-graphql-api project:

__build.gradle__
```groovy
dependencies {
    ...
    implementation 'com.okta.spring:okta-spring-boot-starter:3.0.5'
    ...
}
```

Set the client ID, issuer, and audience for OAuth 2.0 in the `application.properties` file:

__application.properties__
```properties
okta.oauth2.issuer=https://{yourAuth0Domain}/
okta.oauth2.client-id={clientId}
okta.oauth2.audience=${okta.oauth2.issuer}api/v2/
```

Add the client secret to the `.env` file:

__.env__
```shell
export OKTA_OAUTH2_CLIENT_SECRET={clientSecret}
```

Add the following factory method to the class `SpringBootGraphQlApiConfig`, for requiring a bearer token for all requests:

__SpringBootGraphQlApiConfig.java__
```java
    ...
    @Bean
    public SecurityFilterChain configure(HttpSecurity http) throws Exception {
        http.oauth2ResourceServer(oauth2ResourceServer -> oauth2ResourceServer.jwt(withDefaults()));
        return http.build();
    }
    ...
```

> __NOTE:__
> The Okta Spring Boot starter provides the security auto-configuration out of the box, and the resource server configuration should not be necessary. For some reason, the Spring for GraphQL cors allowed origins configuration does not take effect without the customization above.

Again, in the root folder, run the API server with:

```shell
source .env && ./gradlew bootRun
```

With HTTPie, send a request to the API server using a bearer access token:

```shell
ACCESS_TOKEN={auth0AccessToken}
```

```shell
echo -E '{"query":"{\n    companyList(page: 20) {\n        id\n        SIC\n        name\n        status\n        category\n        companyNumber\n        countryOfOrigin\n    }\n}"}' | \
  http -A bearer -a $ACCESS_TOKEN POST http://localhost:8080/graphql
```

> __NOTE:__
> Follow [these instructions](https://auth0.com/docs/secure/tokens/access-tokens/get-management-api-access-tokens-for-testing) to get an access token for the Auth0 Management API, which can be used for testing the server API.

### Add Auth0 Login to the React client

When using Auth0 as the identity provider, you can configure the Universal Login Page for a quick integration without having to build the login forms. First, register a SPA OIDC application using the Auth0 CLI:

```shell
auth0 apps create \
  --name "React client for GraphQL" \
  --description "SPA React client for a Spring GraphQL API" \
  --type spa \
  --callbacks http://localhost:3000/callback \
  --logout-urls http://localhost:3000 \
  --reveal-secrets
```

Copy the Auth0 domain and the client ID, and update the `src/.env.local` adding the following properties (add the new variables to the example file, too):

__.env.local__
```shell
NEXT_PUBLIC_AUTH0_DOMAIN={yourAuth0Domain}
NEXT_PUBLIC_AUTH0_CLIENT_ID={clientId}
NEXT_PUBLIC_AUTH0_CALLBACK_URL=http://localhost:3000/callback
NEXT_PUBLIC_AUTH0_AUDIENCE=https://{yourAuth0Domain}/api/v2/
```

For handling the Auth0 post-login behavior, you need to add the page `src/app/callback/page.tsx` with the following content.

__callback/page.tsx__
```tsx
import Loader from "@/components/loader/Loader";

const Page = () => {
  return <Loader/>
};

export default Page;
```

For this example, the callback page will render empty.

Add the `@auth0/auth0-react` dependency to the project:

```shell
npm install @auth0/auth0-react
```

Create the component `Auth0ProviderWithNavigate` in the folder `src/components/authentication` with the following content:

{% raw %}
__Auth0ProviderWithNavigate.tsx__
```tsx
import { AppState, Auth0Provider } from "@auth0/auth0-react";
import { useRouter } from "next/navigation";
import React from "react";

const Auth0ProviderWithNavigate = (props: { children: React.ReactNode }) => {
  const router = useRouter();

  const domain = process.env.NEXT_PUBLIC_AUTH0_DOMAIN || "";
  const clientId = process.env.NEXT_PUBLIC_AUTH0_CLIENT_ID || "";
  const redirectUri = process.env.NEXT_PUBLIC_AUTH0_CALLBACK_URL || "";
  const audience = process.env.NEXT_PUBLIC_AUTH0_AUDIENCE || "";

  const onRedirectCallback = (appState?: AppState) => {
    router.push(appState?.returnTo || window.location.pathname);
  };

  if (!(domain && clientId && redirectUri)) {
    return null;
  }

  return (
    <Auth0Provider
      domain={domain}
      clientId={clientId}
      authorizationParams={{
        audience: audience,
        redirect_uri: redirectUri,
      }}
      useRefreshTokens={true}
      onRedirectCallback={onRedirectCallback}
    >
      <>{props.children}</>
    </Auth0Provider>
  );
};

export default Auth0ProviderWithNavigate;
```
{% endraw %}

The component `Auth0ProviderWithNavigate` wraps the children component with `Auth0Provider`, the provider of the Auth0 context, remembering the requested URL for redirection after login.
Use the component in the `WideLayout` component:

{% raw %}
__WideLayout.tsx__
```tsx
const WideLayout = (props: { children: React.ReactNode }) => {
  return (
    <ThemeProvider theme={theme}>
      <Auth0ProviderWithNavigate>
        <Container maxWidth="lg" sx={{ mt: 4 }}>
          {props.children}
        </Container>
      </Auth0ProviderWithNavigate>
    </ThemeProvider>
  );
};
```
{% endraw %}

Add the file `src/components/authentication/AuthenticationGuard.tsx` with the following content:

__AuthenticationGuard.tsx__
```tsx
'use client'

import { useAuth0 } from "@auth0/auth0-react";
import { useEffect } from "react";
import Loader from "../loader/Loader";

const AuthenticationGuard = (props: { children: React.ReactNode }) => {
  const { isLoading, isAuthenticated, error, loginWithRedirect } = useAuth0();

  useEffect(() => {
    if (!isAuthenticated && !isLoading) {
      loginWithRedirect({
        appState: { returnTo: window.location.href },
      });
    }
  }, [isAuthenticated, isLoading, loginWithRedirect]);

  if (isLoading) {
    return <Loader />;
  }
  if (error) {
    return <div>Oops... {error.message}</div>;
  }
  return <>{isAuthenticated && props.children}</>;
};

export default AuthenticationGuard;
```

The `AuthenticationGuard` component will be used to protect pages that require authentication, redirecting to the Auth0 universal login. Protect the index page by wrapping its content in the `AuthenticationGuard` component:

{% raw %}
__app/page.tsx__
```tsx
import AuthenticationGuard from "@/components/authentication/AuthenticationGuard";
import HomePage from "./HomePage";

const Page = () => {
  return (
    <AuthenticationGuard>
      <HomePage></HomePage>
    </AuthenticationGuard>
  );
};

export default Page;
```
{% endraw %}

### Call the API server with an access token

Add the file `src/services/auth.tsx` with the following code:

__auth.tsx__
```tsx
import backendAPI from "./base";

let requestInterceptor: number;
let responseInterceptor: number;

export const clearInterceptors = () => {
  backendAPI.interceptors.request.eject(requestInterceptor);
  backendAPI.interceptors.response.eject(responseInterceptor);
};

export const setInterceptors = (accessToken: String) => {

  clearInterceptors();

  requestInterceptor = backendAPI.interceptors.request.use(
    // @ts-expect-error
    function (config) {
      return {
        ...config,
        headers: {
          ...config.headers,
          Authorization: `Bearer ${accessToken}`,
        },
      };
    },
    function (error) {
      console.log("request interceptor error", error);
      return Promise.reject(error);
    }
  );
};
```

Add the file `src/hooks/useAccessToken.tsx` with the following content:

__useAccessToken.tsx__
```tsx
import { setInterceptors } from "@/services/auth";
import { useAuth0 } from "@auth0/auth0-react";
import { useCallback, useState } from "react";

export const useAccessToken = () => {
  const { isAuthenticated, getAccessTokenSilently } = useAuth0();
  const [accessToken, setAccessToken] = useState("");

  const saveAccessToken = useCallback(async () => {
    if (isAuthenticated) {
      try {
        const tokenValue = await getAccessTokenSilently();
        if (accessToken !== tokenValue) {
          setInterceptors(tokenValue);
          setAccessToken(tokenValue);
        }
      } catch (err) {
        // Inactivity timeout
        console.log("getAccessTokenSilently error", err);
      }
    }
  }, [getAccessTokenSilently, isAuthenticated, accessToken]);

  return {
    saveAccessToken,
  };
};
```

The hook will call Auth0's `getAccessTokenSilently()` and trigger a token refresh if the access token expires. Then, it will update Axios interceptors to set the updated bearer token value in the request headers.
Create the `useAsyncWithToken` hook:

__useAsyncWithToken.tsx__
```tsx
import { useAccessToken } from "./useAccessToken";
import { useAsync } from "react-use-custom-hooks";

export const useAsyncWithToken = <T, P, E = string>(
  asyncOperation: () => Promise<T>, deps: any[]
) => {
  const { saveAccessToken } = useAccessToken();
  const [ data, loading, error ] = useAsync(async () => {
    await saveAccessToken();
    return asyncOperation();
  }, {},  deps);

  return {
    data,
    loading,
    error
  };
};
```

Update the calls in the `CompanyTableContainer` component to use the `useAsyncWithToken` hook:

__CompanyTableContainer.tsx__
```tsx
...
const {
  data: dataList,
  loading: loadingList,
  error: errorList,
} = useAsyncWithToken(
  () => CompanyApi.getCompanyList({ page: page - 1}),
  [props.page]
);

const { data: dataCount } = useAsyncWithToken(
  () => CompanyApi.getCompanyCount(),
  []
);
...
```

Run the application with:

```shell
npm run dev
```

Go to `http://localhost:3000` and you should be redirected to the Auth0 universal login page. After logging in, you should see the companies list again.

{% img blog/spring-graphql-react/auth0-universal-login.png alt:"Auth0 universal login form" width:"400" %}{: .center-image }

{% img blog/spring-graphql-react/auth0-authorize-app.png alt:"Auth0 authorize application form" width:"400" %}{: .center-image }

Once the companies load, you can inspect the network requests and see the bearer token is sent in the request headers. It will look like the example below:

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IlJzZHNsM211SjNYU2ZVT0tDOEMxSiJ9.eyJodHRwczovL3d3dy5qaGlwc3Rlci50ZWNoL3JvbGVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1VTRVIiXSwiaXNzIjoiaHR0cHM6Ly9kZXYtYXZ1cDJsYXoudXMuYXV0aDAuY29tLyIsInN1YiI6ImF1dGgwfDY0MzQxOTkxNTJmYjc2N2Y3ZWFlZDU2NyIsImF1ZCI6WyJhcGktc2VydmVyIiwiaHR0cHM6Ly9kZXYtYXZ1cDJsYXoudXMuYXV0aDAuY29tL3VzZXJpbmZvIl0sImlhdCI6MTY5MTU5NDE3NCwiZXhwIjoxNjkxNTk0Mjk0LCJhenAiOiI1WW5QeEJiTjRoYXFyd2JNaXRpZFBRVTBjb0l2YUI3SCIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgZW1haWwgb2ZmbGluZV9hY2Nlc3MifQ.oRMJNIwSeO9dfHi6Q2tr_B51YetEPZqjcVEEIBe3ky9tEe50tTB5ssMTbVaR78_1qXA663Cn4EMPEYTLlb_wiOHEqnKpZnpq0O07G2MGszkrgv5giaQBOXvr9UT_Kc3pFPK-xMVmOsicLoF_mz8iyOzReG1Gcw0UbS1fZsJthdtC9svLiPGC1rn-dwPJxPpKy118vWLbEgO3NsdVfaiPxbucv0TL_B8Msd-wBD8N4M_yej9jl7w2JT0ltLza_-Glbxr1aQBKd19O2QT8ovgyE325BYqeYiUOaV7efwMgqHUm7z4LngaPLRNhMLP2BExG7bUQD2JjH2Mh19LFYNM-Gw
```

## Update the GraphQL query in the client

The GraphQL query in the React client application can be easily updated to request more data from the server. For example, add the `status` and information about who controls the company. First, update the API client:

__companies.tsx__
```tsx
...

export type PersonDTO = {
  name: string;
}

export type CompanyDTO = {
  name: string;
  SIC: string;
  id: string;
  companyNumber: string;
  category: string;
  status: string;
  controlledBy: PersonDTO[]
};

...

  getCompanyList: async (params?: CompaniesQuery) => {

    try {
      const response = await backendAPI.post("/graphql", {
        query: `{
        companyList(page: ${params?.page || 0}) {
          name,
          SIC,
          id,
          companyNumber,
          category,
          status,
          controlledBy {
            name
          }
        }}`,
      });
      return response.data.data.companyList as CompanyDTO[];
    } catch (error) {
      console.log("handle get companies error", error);
      if (error instanceof AxiosError) {
        let axiosError = error as AxiosError;
        if (axiosError.response?.data) {
          throw new Error(axiosError.response?.data as string);
        }
      }
      throw new Error("Unknown error, please contact the administrator");
    }
  },
...
```

Then update the `CompanyData` interface in the `CompanyTable.tsx` component:

__CompanyTable.tsx__
```tsx
export interface CompanyData {
  id: string,
  name: string,
  category: string,
  companyNumber: string,
  SIC: string
  status: string,
  owner: string
}
```

Finally, update the `CompanyTableContainer` column definitions, and data formatting. The final code should look like below:

{% raw %}
__CompanyTableContainer.tsx__
```tsx
import { GridColDef, GridPaginationModel } from "@mui/x-data-grid";
import CompanyTable from "./CompanyTable";
import { usePathname, useRouter, useSearchParams } from "next/navigation";
import { CompanyApi, CompanyDTO } from "@/services/companies";
import Loader from "../loader/Loader";
import { useAsyncWithToken } from "@/app/hooks/useAsyncWithToken";

interface CompanyTableProperties {
  page?: number;
}

const columns: GridColDef[] = [
  { field: "id", headerName: "ID", width: 70 },
  {
    field: "companyNumber",
    headerName: "Company #",
    width: 100,
    sortable: false,
  },
  { field: "name", headerName: "Company Name", width: 250, sortable: false },
  { field: "category", headerName: "Category", width: 200, sortable: false },
  { field: "SIC", headerName: "SIC", width: 200, sortable: false },
  { field: "status", headerName: "Status", width: 100, sortable: false },
  { field: "owner", headerName: "Owner", width: 200, sortable: false },
];

const CompanyTableContainer = (props: CompanyTableProperties) => {
  const router = useRouter();
  const searchParams = useSearchParams()!;
  const pathName = usePathname();
  const page = props.page ? props.page : 1;

  const {
    data: dataList,
    loading: loadingList,
    error: errorList,
  } = useAsyncWithToken(
    () => CompanyApi.getCompanyList({ page: page - 1}),
    [props.page]
  );

  const { data: dataCount } = useAsyncWithToken(
    () => CompanyApi.getCompanyCount(),
    []
  );

  const onPageChange = (pagination: GridPaginationModel) => {
    const params = new URLSearchParams(searchParams.toString());
    const page = pagination.page + 1;
    params.set("page", page.toString());
    router.push(pathName + "?" + params.toString());
  };

  const companyData = dataList?.map((company: CompanyDTO) => {
    return {
      id: company.id,
      name: company.name,
      category: company.category,
      companyNumber: company.companyNumber,
      SIC: company.SIC,
      status: company.status,
      owner: company.controlledBy.map((person) => person.name).join(", "),
    }
  });

  return (
    <>
      {loadingList && <Loader />}
      {errorList && <div>Error</div>}

      {!loadingList && dataList && (
        <CompanyTable
          pagination={{ page: page - 1, pageSize: 10 }}
          rowCount={dataCount}
          rows={companyData}
          columns={columns}
          onPageChange={onPageChange}
        ></CompanyTable>
      )}
    </>
  );
};

export default CompanyTableContainer;
```
{% endraw %}

Give it a try, and notice how simple was the update.

## Learn More About Spring Boot and React

I hope you enjoyed this tutorial, and found this example useful. As you can see, not much work would be required to consume more company data from the GraphQL server, just a query update in the client. Also, the Auth0 Universal Login and Auth0 React SDK provide an efficient way to secure your React applications, following security best practices. You can find all the code for this example in the [GitHub repository](https://github.com/oktadev/auth0-spring-graphql-react-example).

Check out the Auth0 documentation for adding [sign-up](https://developer.auth0.com/resources/guides/spa/react/basic-authentication#add-user-sign-up-to-react) and [logout](https://developer.auth0.com/resources/guides/spa/react/basic-authentication#add-user-logout-to-reactfunctionality) to your React application. And for more fun tutorials about Spring Boot and React, you can visit the following links:

- [Build a Simple CRUD App with Spring Boot and Vue.js](https://auth0.com/blog/build-crud-spring-and-vue/)
- [Use React and Spring Boot to Build a Simple CRUD App](https://auth0.com/blog/simple-crud-react-and-spring-boot/)
- [Full Stack Java with React, Spring Boot, and JHipster](https://auth0.com/blog/full-stack-java-with-react-spring-boot-and-jhipster/)

Keep in touch! If you have questions about this post, please ask them in the comments below. And follow us! We're [@oktadev on Twitter](https://twitter.com/oktadev), [@oktadev on YouTube](https://youtube.com/c/oktadev), and frequently post to our [LinkedIn page](https://www.linkedin.com/company/oktadev/). You can also sign up for our [newsletter](https://a0.to/nl-signup/java) to stay updated on everything Identity and Security.